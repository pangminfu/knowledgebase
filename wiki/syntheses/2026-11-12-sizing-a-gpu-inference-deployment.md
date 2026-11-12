---
type: synthesis
tags: [provisioning, tenancy-quotas, metering-billing, career]
sources: [2026-11-12-transformer-inference-arithmetic, 2026-11-10-deep-learning-brrrr-first-principles]
created: 2026-11-12
updated: 2026-11-12
---

# Sizing a GPU inference deployment

How to go from "we want to serve this model" to a concrete GPU count, concurrency limit, and cost per token — using only arithmetic. The chain is: **weights decide GPUs → leftover memory decides concurrency → concurrency decides cost per token → link bandwidth decides whether more GPUs help**. All constants and formulas come from [[2026-11-12-transformer-inference-arithmetic]]; the framing for a control plane is general knowledge unless cited.

## The four-step chain

1. **Weights → GPUs.** Size = 2 bytes × parameters (16-bit). Divide by per-GPU memory, round up to a power of two GPUs (splitting over 3 GPUs wastes a third of the memory).
2. **Leftover memory → concurrency.** Subtract weights, divide by bytes per cached token (`4 · n_layers · d_model`), then by context length. That is how many requests one replica can hold.
3. **Concurrency → cost.** Batch size sets whether the GPU is [[memory-bandwidth]]-bound (few tokens, mostly paying to stream weights) or flops-bound (many tokens, paying per token). Cost per token is GPU-hours ÷ tokens, so it falls until the flops bound.
4. **GPUs → latency floor.** Each added GPU cuts compute time by `1/N` but adds four communication rounds per layer; past the flops-per-byte threshold (1040 for A100 NVLink) more GPUs stop helping. See [[model-parallelism]].

## Worked example: two model sizes on A100-40GB

Constants: 312e12 flops/s, 1.5e12 bytes/s, 300 GB/s NVLink, 16-bit weights. (src: [[2026-11-12-transformer-inference-arithmetic]])

| | 13B model | 52B model |
| --- | --- | --- |
| Weights | 26 GB | 104 GB |
| GPUs required | 1 (fits with room) | 3 minimum, 4 in practice |
| Memory left for cache | ~14 GB (1 GPU) | 56 GB (4 GPUs) |
| Cache per token | 0.8 MB (`4·40·5120`) | 2 MB (`4·64·8192`) |
| Tokens of cache | ~17,000 | ~23,000 (source's figure, after overhead) |
| Per-token latency, batch 1 | 16.8 ms on 1 GPU, 8.4 ms on 2 | ~69 ms on 1 GPU, ~17 ms on 4 |
| Measured (FasterTransformer) | 22.0 ms / 13.5 ms | not measured |

Weights, cache-per-token and latency figures are from the source's worked examples; the 13B cache budget is derived by the same formula.

- The 52B row is the important one: **the model does not fit on one GPU**, so every replica is a multi-GPU gang, and the provider must place all four on NVLink-connected cards. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Both models sit below the 208-token threshold at batch 1, so both are memory-bound at startup and become cheaper per token only as concurrency rises. (src: [[2026-11-12-transformer-inference-arithmetic]])

## What this implies for the control plane

| Control-plane concern | Consequence |
| --- | --- |
| Provisioning / SKU choice | A SKU must advertise memory *and* link topology: 52B-class models only run on multi-GPU NVLink domains. (general knowledge) |
| Quotas | Concurrency limits are memory limits: `(VRAM − weights) ÷ (cache per token × context)`. Counting GPUs alone overcommits. (general knowledge) |
| Admission control | Reject or queue a request when the cache budget is exhausted rather than letting the container OOM. (general knowledge) |
| Scheduling | Tensor-parallel replicas are gang-scheduled (all GPUs or none); pipeline-parallel deployments want many small requests in flight. (src + general knowledge) |
| Metering | Tokens/second is the inference-native unit; GPU-hours is the training-native one. A batch service and an interactive service have very different tokens/GPU-hour. (general knowledge) |
| Autoscaling | Scale replicas on queue depth and cache occupancy, not only on GPU-Util: memory-bound replicas can look idle while saturated. (general knowledge) |

## What the arithmetic cannot tell you

- **Real efficiency:** predictions land within 10–35% of benchmarks, and the shortfall is software (fusion, kernel quality, achieved bandwidth). Price from a benchmark, size from the model. (src: [[2026-11-12-transformer-inference-arithmetic]])
- **Mixed context lengths:** the model assumes a uniform batch. Real traffic has variable lengths, which padding and static batching waste; continuous batching and paged caches exist to recover that. (general knowledge)
- **Quantisation:** FP8/INT8 serving halves cache and weight bytes, roughly doubling the concurrency the same card supports, at some quality cost. (general knowledge; not in the source)

## Related

- [[2026-11-12-transformer-inference-arithmetic]]: the source with all formulas and measurements.
- [[gpu-memory-capacity]] · [[kv-cache]] · [[inference-batching]] · [[model-parallelism]] · [[inference-latency-model]]
- [[2026-11-10-deep-learning-brrrr-first-principles]]: the compute/memory/overhead framing this builds on.
- [[overview]]: where this fits in the topic.
