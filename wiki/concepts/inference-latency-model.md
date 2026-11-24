---
type: concept
tags: [provisioning, metering-billing]
sources: [2026-11-12-transformer-inference-arithmetic]
created: 2026-11-12
updated: 2026-11-24
---

# Inference latency model

A **closed-form latency model** estimates how long one decode step takes from hardware constants and model size, without running anything. For a language model it comes down to two numbers per regime, the time to stream the weights through [[memory-bandwidth]] and the time to do the [[flops]], plus a communication term for multi-GPU serving ([[model-parallelism]]). The result is a *floor*: in the source's benchmarks the naive estimate came in 24% (1 GPU) and 38% (2 GPUs) below measured latency. (src: [[2026-11-12-transformer-inference-arithmetic]])

## Details

### Inputs

| Symbol | Meaning |
| --- | --- |
| `P` | parameters in the model |
| `N` | GPUs used |
| `B` | batch size (sequences decoded together) |
| `n_layers`, `d_model` | transformer depth and width |
| `A_bm` | per-GPU memory bandwidth, bytes/s (post's A100 value: 1.5e12) |
| `A_f` | per-GPU peak flops/s (A100: 312e12, dense FP16/BF16 on Tensor Cores) |
| `A_c` | inter-GPU link bandwidth, bytes/s (A100 NVLink: 300e9 per direction) |

Numbers are 16-bit, so a parameter costs 2 bytes. (src: [[2026-11-12-transformer-inference-arithmetic]]) NVIDIA's datasheet gives 1,555 GB/s for the 40 GB A100 and 312 TFLOPS dense (624 with sparsity) for FP16/BF16 Tensor Core math; the post rounds bandwidth to 1.5e12. (general knowledge; verified: [NVIDIA A100 datasheet](https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/a100/pdf/nvidia-a100-datasheet-us-nvidia-1758950-r4-web.pdf))

### Small batch (memory-bandwidth bound)

```
compute = 2 bytes · P / (N · A_bm)      # one pass over the weights
comms   = 4 · n_layers · 8 µs           # message latency, N > 1 only
```

- `2 bytes · P` is the size of the weights (16-bit), read once per decode step; divide by bandwidth and by `N` GPUs. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Comms here is pure latency: four messages per layer, each ~8 µs (a V100 NVLink measurement), not multiplied by batch because one message carries the whole batch. Its throughput cost "rounds away". (src: [[2026-11-12-transformer-inference-arithmetic]])
- How to combine the two terms depends on overlap: the post takes the larger when comms and compute run in parallel, and in its 2-GPU benchmark the comms were *not* overlapped and added to compute. (src: [[2026-11-12-transformer-inference-arithmetic]])
- 52B on one A100: `2 · 52e9 / 1.5e12` ≈ 69 ms per step (hypothetical: the weights don't fit on one 40 GB card); on four GPUs ≈ 17 ms. The post computes it with `P ≈ 12 · n_layers · d_model²` = 51.5B, which gives the same 69 ms. (src: [[2026-11-12-transformer-inference-arithmetic]])

### Large batch (flops bound)

```
compute = B · 2 · P / (N · A_f)
comms   = B · 2 · 4 · n_layers · d_model bytes / A_c
latency ≈ max(compute, comms)           # assumes they overlap
```

- Here comms becomes a throughput term: 4 messages × `n_layers` × `d_model` values × 2 bytes per sequence (the post rounds `(N−1)/N` up to 1). (src: [[2026-11-12-transformer-inference-arithmetic]])
- Taking the larger of the two assumes compute and comms overlap; the source says not all systems do this, and never perfectly. This is also why latency cannot approach zero as GPUs are added: eventually comms exceed compute. (src: [[2026-11-12-transformer-inference-arithmetic]])

### Where the flops term comes from

- `2 · P` per token is the same as `24 · n_layers · d_model²`: per layer, 6 for q/k/v, 2 for the output projection, 16 for the two MLP matrices. (src: [[2026-11-12-transformer-inference-arithmetic]])
- On the 52B model that totals 103,079,215,104 ≈ 103 GFLOP, implying 51.5B parameters. The source attributes the gap to the token (un)embeddings and calls it under 2%, so `2 · P` is a fine approximation. (src: [[2026-11-12-transformer-inference-arithmetic]]) The model's paper counts 52B as non-embedding parameters, so the gap is really rounding in the name. (general knowledge; verified: [Askell et al. 2021](https://arxiv.org/abs/2112.00861))
- The attention step itself (`softmax(q·kᵀ)·v`) multiplies no weight matrices; the source says its flops are "some factor of `d_model`" and negligible. (src: [[2026-11-12-transformer-inference-arithmetic]]) More precisely it also scales with context length: Kaplan et al. estimate a forward pass at `2N + 2 · n_layers · n_ctx · d_model` flops per token, so the attention share is roughly `n_ctx / (12 · d_model)`. For the 52B model at 2048 tokens that is ~2%, but it is not negligible for very long contexts. (general knowledge; verified: [Kaplan et al. 2020, Scaling Laws, eq. 2.2](https://arxiv.org/abs/2001.08361))

### Worked examples from the post

| Case | Prediction |
| --- | --- |
| Gopher-size 260B (80 layers, `d_model` 16384), 16 GPUs, batch 1 | 22 ms per step (21.7); comms latency `4·80·8 µs` = 2.56 ms; comms throughput ≈ 35 µs |
| Same, batch 512 | compute 53 ms, comms throughput ≈ 18 ms → ≈ 53 ms per step, producing 512 tokens |
| 52B, 4 GPUs, batch 256 (post's exercise) | compute ≈ 21 ms, comms ≈ 4 ms |
| 13B, 1 GPU, 512-token context, batch 1 | 16.8 ms per step (measured 22.0) |
| 13B, 2 GPUs, same | 8.4 ms + ~1 ms comms (measured 13.5) |

(src: [[2026-11-12-transformer-inference-arithmetic]]) The post also writes "62 ms" for the batch-512 step, which its own terms do not produce; and Gopher is actually 280B, with 260B ≈ `12 · 80 · 16384²`. See [[2026-11-12-transformer-inference-arithmetic#Contradictions]]. (arithmetic check, this wiki; verified: [Gopher paper](https://arxiv.org/abs/2112.11446))

### What the model omits

- Memory-bound elementwise/normalisation kernels: 43% of one BERT layer's time at 336M parameters even with the paper's fused kernels; the post scales this to ~5% at 52B. See [[operator-fusion]]. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Reads of the [[kv-cache]] (grows with context and batch) and of the unembedding matrix (`d_model × n_vocab`, to produce logits), both memory-bandwidth costs. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Intermediate activations, which the source calls hard to count and software-dependent. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Cross-node networking, queueing, padding, and any batching across requests of different lengths. (general knowledge)
- Achieved rather than peak hardware: the benchmark showed ~87–90% of peak bandwidth, and 54–72% of peak flops on its matmul shapes. (src: [[2026-11-12-transformer-inference-arithmetic]])

### Reconciling prediction with measurement

- One GPU, 13B: predicted 16.8 ms, measured 22.0 ms. At 90% achieved bandwidth → 18.5 ms (the post's figure); plus 2.2 ms of intermediate ops → 20.7 ms. The last ~1.3 ms (the post says 1.4) is token embeddings, sampling (top-k/top-p), average bandwidth below 90%, and kernel launches. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Two GPUs: predicted 8.4 ms, measured 13.5 ms. At 87% bandwidth → 9.5 ms; plus 2 ms intermediates → 11.5 ms (the source writes 11.7 ms); plus ~1.7 ms of un-overlapped comms (40–50 µs per layer, 40 layers) → ≈ 13.2–13.4 ms. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Context (prefill) pass: predicted as decode step × 512/231 → ≈ 48 ms (1 GPU) and ≈ 30 ms (2 GPUs); measured 63.17 ms and 45.45 ms. Part of the 1-GPU gap is kv storing (≈ 0.7 ms), memsets (0.2 ms), and matmuls reaching only 54–72% of peak flops. (src: [[2026-11-12-transformer-inference-arithmetic]])

## Why it matters for a GPU cloud

- This is the arithmetic a capacity planner uses before buying: model size → GPUs needed → batch that fits → tokens/s → cost per token. See the worked version in [[2026-11-12-sizing-a-gpu-inference-deployment]]. (general knowledge)
- It also answers "why is my GPU not at 100%?": at small batch you are memory-bound by construction, yet nvidia-smi's GPU-Util can read near 100%, because it only reports the share of time at least one kernel was running, not how much of the compute units that kernel used ([[gpu-utilization-metrics]]). (general knowledge; verified: [NVML `nvmlUtilization_t`](https://docs.nvidia.com/deploy/archive/R550/nvml-api/structnvmlUtilization__t.html))
- Naive predictions 24–38% off, closed by measured corrections, are good enough for SKU selection and bad enough that any per-token price must come from a benchmark. (general knowledge)

## Related

- [[inference-batching]]: which of the two regimes applies.
- [[model-parallelism]]: where the comms terms come from.
- [[kv-cache]]: omitted memory term that grows with context.
- [[flops]] · [[memory-bandwidth]]: the two hardware constants.
- [[gpu-performance-regimes]]: the general framing this specializes.
