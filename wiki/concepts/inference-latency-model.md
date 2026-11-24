---
type: concept
tags: [provisioning, metering-billing]
sources: [2026-11-12-transformer-inference-arithmetic]
created: 2026-11-12
updated: 2026-11-24
---

# Inference latency model

A **closed-form latency model** estimates how long one decode step takes from hardware constants and model size, without running anything. For a language model it comes down to two numbers per candidate regime — time to stream the weights through [[memory-bandwidth]], and time to do the [[flops]] — plus a communication term for multi-GPU serving ([[model-parallelism]]). The result is a *floor*: in the source's benchmarks the naive estimate came in 24% (1 GPU) and 38% (2 GPUs) below measured latency. (src: [[2026-11-12-transformer-inference-arithmetic]])

## Details

### Inputs

| Symbol | Meaning |
| --- | --- |
| `P` | parameters in the model |
| `N` | GPUs used |
| `B` | batch size (requests in flight) |
| `n_layers`, `d_model` | transformer depth and width |
| `A_bm` | per-GPU memory bandwidth, bytes/s (A100: 1.5e12) |
| `A_f` | per-GPU peak flops/s (A100: 312e12 on Tensor Cores) |
| `A_c` | inter-GPU link bandwidth (A100 NVLink: 300e9 one-way) |

Numbers are 16-bit, so a parameter costs 2 bytes. (src: [[2026-11-12-transformer-inference-arithmetic]])

### Small batch (memory-bandwidth bound)

```
compute = 2 bytes · P / (N · A_bm)      # one pass over the weights
comms   = 4 · n_layers · 8 µs
latency ≈ max(compute, comms)
```

- `2 bytes · P` is the size of the weights (16-bit), read once per token; divide by bandwidth and by `N` GPUs. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Comms is pure latency: four messages per layer, each ~8 µs, not multiplied by batch because a message can carry the whole batch. (src: [[2026-11-12-transformer-inference-arithmetic]])
- 52B on one A100: `2 · 52e9 / 1.5e12` ≈ 69 ms per token (hypothetical: the weights don't fit on one 40 GB card); on four GPUs ≈ 17 ms. Same numbers whether you count bytes (`2·P`) or flops (`2·12·n_layers·d_model²`). (src: [[2026-11-12-transformer-inference-arithmetic]])

### Large batch (flops bound)

```
compute = B · 2 · P / (N · A_f)
comms   = B · 8 · n_layers · d_model bytes / A_c
latency ≈ max(compute, comms)
```

- Here comms becomes a throughput term: `4` messages × `n_layers` × `d_model` values × 2 bytes. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Taking the larger of the two assumes compute and comms overlap; the source calls this a real assumption, not a guarantee. (src: [[2026-11-12-transformer-inference-arithmetic]])

### Where the flops term comes from

- `2 · P` is the same as `24 · n_layers · d_model²` per token: 6 for q/k/v, 2 for the output projection, 16 for the two MLP matrices. (src: [[2026-11-12-transformer-inference-arithmetic]])
- On the 52B model that totals 103 GFLOP, implying 51.5B parameters; the missing 0.5B is the token embeddings, under 2% — so `2 · P` is a fine approximation. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Attention itself (the `softmax(q·k)·v` step) contributes only ~`d_model`-scale flops, so it never appears in the count. (src: [[2026-11-12-transformer-inference-arithmetic]])

### Worked examples from the post

| Case | Prediction |
| --- | --- |
| Gopher 260B, 16 GPUs, batch 1 | 22 ms per token |
| Gopher 260B, 16 GPUs, batch 512 | 53 ms per token (62 ms per batch of 512) |
| 13B, 2 GPUs, 512-token context, 1 token | 8.4 ms (measured 13.5) |
| 13B, 1 GPU, same | 16.8 ms (measured 22.0) |

(src: [[2026-11-12-transformer-inference-arithmetic]])

### What the model omits

- Memory-bound elementwise/normalisation kernels: 43% of latency at 336M parameters even with the paper's fused kernels, ~5% at 52B. See [[operator-fusion]]. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Reads of the [[kv-cache]] and of the unembedding matrix (to produce logits), both memory costs that grow with context and vocabulary. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Intermediate activations, which the source calls hard to count and software-dependent. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Cross-node networking, queueing, padding, and any batching across requests of different lengths. (general knowledge)
- Achieved rather than peak hardware: the benchmark showed ~87–90% of bandwidth and 54–72% of peak flops on small matrices. (src: [[2026-11-12-transformer-inference-arithmetic]])

### Reconciling prediction with measurement

- One GPU, 13B: predicted 16.8 ms, measured 22.0 ms. At 90% achieved bandwidth → 18.5 ms; plus 2.2 ms of intermediate ops → 20.7 ms; the last 1.4 ms is token embeddings, sampling (top-k/top-p), sub-90% average bandwidth, kernel launch. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Two GPUs: predicted 8.4 ms, measured 13.5 ms. At 87% bandwidth → 9.5 ms; plus 2.0 ms intermediates → 11.5 ms (the source writes 11.7 ms); plus ~1.7 ms of un-overlapped comms (40–50 µs per layer) → 13.2 ms. (src: [[2026-11-12-transformer-inference-arithmetic]])

## Why it matters for a GPU cloud

- This is the arithmetic a capacity planner uses before buying: model size → GPUs needed → batch that fits → tokens/s → cost per token. See the worked version in [[2026-11-12-sizing-a-gpu-inference-deployment]]. (general knowledge)
- It is also the honest way to answer "why is my GPU not at 100%?": at small batch you are memory-bound by definition, and no amount of GPU-Util will show otherwise ([[gpu-utilization-metrics]]). (src + general knowledge)
- Naive predictions 24–38% off, closed by measured corrections, are good enough for SKU selection and bad enough that any per-token price must come from a benchmark. (general knowledge)

## Related

- [[inference-batching]]: which of the two regimes applies.
- [[model-parallelism]]: where the comms terms come from.
- [[kv-cache]]: omitted memory term that grows with context.
- [[flops]] · [[memory-bandwidth]]: the two hardware constants.
- [[gpu-performance-regimes]]: the general framing this specializes.
