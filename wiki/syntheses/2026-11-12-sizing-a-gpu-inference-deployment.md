---
type: synthesis
tags: [provisioning, tenancy-quotas, metering-billing]
sources: [2026-11-12-transformer-inference-arithmetic, 2026-11-10-deep-learning-brrrr-first-principles]
created: 2026-11-12
updated: 2026-11-24
---

# Sizing a GPU inference deployment

How to go from "we want to serve this model" to a concrete GPU count, concurrency limit, and cost per token, using only arithmetic. The chain is: **weights decide GPUs → leftover memory decides concurrency → concurrency decides cost per token → link bandwidth decides whether more GPUs help**. All constants and formulas come from [[2026-11-12-transformer-inference-arithmetic]]; the framing for a control plane is general knowledge unless cited.

## The four-step chain

1. **Weights → GPUs.** Size = 2 bytes × parameters (16-bit). Divide by per-GPU memory and round up, then usually to a count that divides the model's dimensions evenly (2, 4, 8). The source picks 4 GPUs over 3 for a 52B model, partly because "it's silly to divide powers of two over three GPUs". (src: [[2026-11-12-transformer-inference-arithmetic]]) In tensor parallelism the number of attention heads must also divide evenly across the GPUs. (general knowledge)
2. **Leftover memory → concurrency.** Subtract weights, divide by bytes per cached token (`4 · n_layers · d_model` at 16-bit with standard multi-head attention; less with grouped-query attention, see [[kv-cache]]), then by context length. That is how many requests one replica can hold. (src: [[2026-11-12-transformer-inference-arithmetic]])
3. **Concurrency → cost.** Batch size sets whether the GPU is [[memory-bandwidth]]-bound (few tokens per pass, mostly paying to stream weights) or flops-bound (many tokens, paying per token). Cost per token is GPU-hours ÷ tokens, so it falls with batch size until the flops bound (~208 tokens per pass on the source's A100 numbers). (src: [[2026-11-12-transformer-inference-arithmetic]])
4. **GPUs → latency floor.** Each added GPU cuts compute time by `1/N` but keeps four communication rounds per layer whose per-message latency does not shrink; once comms time exceeds compute time, more GPUs stop helping. Separately, each layer's flops per byte of comms (`d_model` to `4·d_model`) must exceed the link's ratio (1040 for A100 NVLink) for compute and comms to overlap. See [[model-parallelism]]. (src: [[2026-11-12-transformer-inference-arithmetic]])

## Worked example: two model sizes on A100-40GB

Constants: 312e12 flops/s, 1.5e12 bytes/s, 300 GB/s NVLink per direction, 16-bit weights and cache. (src: [[2026-11-12-transformer-inference-arithmetic]])

| | 13B model | 52B model |
| --- | --- | --- |
| Shape | 40 layers, `d_model` 5120 | 64 layers, `d_model` 8192 |
| Weights | 26 GB | 104 GB |
| GPUs required | 1 (fits with room) | 3 minimum, 4 in practice |
| Memory left for cache | ~14 GB (1 GPU) | 56 GB (4 GPUs) |
| Cache per token | 819,200 B ≈ 0.8 MB (`4·40·5120`) | 2,097,152 B ≈ 2 MB (`4·64·8192`) |
| Tokens of cache | ≈ 17,000 | ≈ 26,700 (source prints 23,000) |
| 2048-token requests at once | ≈ 8 | ≈ 13 |
| Per-token latency, batch 1 (predicted) | 16.8 ms on 1 GPU, 8.4 ms + ~1 ms comms on 2 | ~17 ms on 4 (~69 ms on 1 is hypothetical: it doesn't fit) |
| Measured (FasterTransformer) | 22.0 ms / 13.5 ms | not measured |

Shapes, weights, cache-per-token and latency figures are from the source's worked examples. The 13B cache budget, the token counts (exact division by bytes per token), and the request counts are derived here with the same formulas; the source's 23,000 for the 52B case does not follow from its own numbers (see [[gpu-memory-capacity#Contradictions]]). (arithmetic check, this wiki)

- The 52B row is the important one: **the model does not fit on one GPU**, so every replica spans four GPUs. (src: [[2026-11-12-transformer-inference-arithmetic]]) Those four should be NVLink-connected GPUs in one server: the source's comms numbers assume same-host NVLink (300 GB/s per direction), and over PCIe Gen4 (64 GB/s) the comms terms grow several-fold. (general knowledge; verified: [NVIDIA A100 datasheet](https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/a100/pdf/nvidia-a100-datasheet-us-nvidia-1758950-r4-web.pdf))
- Both models sit far below the 208-token threshold at batch 1, so both are memory-bound at low concurrency and get cheaper per token only as concurrency rises. (src: [[2026-11-12-transformer-inference-arithmetic]])

## What this implies for the control plane

| Control-plane concern | Consequence |
| --- | --- |
| Provisioning / SKU choice | A SKU must advertise memory *and* link topology: 52B-class 16-bit models need several NVLink-connected GPUs on 40 GB cards. (general knowledge) |
| Quotas | Concurrency limits are memory limits: `(GPU memory − weights) ÷ (cache per token × context)`. Counting GPUs alone overcommits. (general knowledge) |
| Admission control | Reject or queue a request when the cache budget is exhausted rather than letting the container OOM. (general knowledge) |
| Scheduling | A single-Node tensor-parallel replica is one Pod requesting all its GPUs, which the default Kubernetes scheduler places all-or-nothing on one Node; choosing NVLink-connected GPUs needs topology-aware allocation. Gang scheduling (start several Pods together) is only needed when a replica spans Nodes. Pipeline-parallel deployments want many requests in flight. See [[kubernetes-scheduling]]. (general knowledge; pipeline point: src: [[2026-11-12-transformer-inference-arithmetic]]) |
| Metering | Tokens/second is the inference-native unit; GPU-hours is the training-native one. A batch service and an interactive service have very different tokens/GPU-hour. (general knowledge) |
| Autoscaling | Scale replicas on queue depth and cache occupancy, not only on GPU-Util. GPU-Util reports the share of time any kernel was running, so a memory-bound replica reads high whether it serves one request or is saturated. (general knowledge; verified: [NVML `nvmlUtilization_t`](https://docs.nvidia.com/deploy/archive/R550/nvml-api/structnvmlUtilization__t.html)) |

## What the arithmetic cannot tell you

- **Real efficiency:** naive predictions came in 24–38% below measured latency; measured bandwidth efficiency, intermediate ops, and comms explain the gap. Price from a benchmark, size from the model. (src: [[2026-11-12-transformer-inference-arithmetic]])
- **Mixed request lengths:** the model assumes a uniform, static batch. Real traffic has variable lengths and arrival times; continuous (iteration-level) batching re-forms the batch every step, and paged kv-cache allocation (PagedAttention) cuts the memory wasted per request so more fit. See [[inference-batching#Batching across requests in practice]]. (general knowledge)
- **Quantisation:** 8-bit weights *and* cache halve both byte counts. For the 52B model on 4×A100-40GB that is 52 GB of weights, 108 GB left, and ~1 MB per token: ≈ 103,000 cached tokens versus ≈ 26,700, about 3.9×. The model would also fit on 2 GPUs with ≈ 26,700 tokens. The A100 has INT8 but not FP8 Tensor Cores; FP8 needs Hopper (H100/H200) or newer. Quality cost varies by model and method. (arithmetic: this wiki; hardware: general knowledge; verified: [NVIDIA A100 datasheet](https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/a100/pdf/nvidia-a100-datasheet-us-nvidia-1758950-r4-web.pdf), [NVIDIA H200](https://www.nvidia.com/en-us/data-center/h200/))

## Related

- [[2026-11-12-transformer-inference-arithmetic]]: the source with all formulas and measurements.
- [[gpu-memory-capacity]] · [[kv-cache]] · [[inference-batching]] · [[model-parallelism]] · [[inference-latency-model]]
- [[kubernetes-scheduling]]: how the multi-GPU replica is placed.
- [[2026-11-10-deep-learning-brrrr-first-principles]]: the compute/memory/overhead framing this builds on.
- [[overview]]: where this fits in the topic.
