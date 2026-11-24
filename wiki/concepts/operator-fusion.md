---
type: concept
tags: [metering-billing]
sources: [2026-11-10-deep-learning-brrrr-first-principles, 2026-11-12-transformer-inference-arithmetic]
created: 2026-11-10
updated: 2026-11-24
---

# Operator fusion

**Operator fusion** combines several operations ("operators") into one GPU kernel. Data is loaded from memory once, all the steps run on-chip, and only the final result is written back, so intermediate results never round-trip to DRAM. Analogy: finish every manufacturing step at the factory instead of shipping half-made parts back to the warehouse between steps. The source calls it "the most important optimization in deep learning compilers". (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

## Details

![[raw/assets/brrr-multi_operators.png]]
Unfused: each of three pointwise ops (square → triangle → circle → rectangle) reads from memory and writes back, so there are three round-trips. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

![[raw/assets/brrr-operator_fusion.png]]
Fused: one read, all three transforms on-chip, one write. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

- `x.cos().cos()` unfused = 4 global memory reads/writes; fused = 2, so about 2× faster. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])
- Requirements: the system has to know the upcoming ops in advance, which rules out PyTorch's default **eager mode** (run one op at a time). It also has to generate GPU (CUDA) code. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])
- Beyond simple pointwise chains: pointwise ops can be fused onto reductions or onto matmuls. A matmul itself can be seen as a fused broadcast-multiply plus a reduction. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])
- Tools: NVFuser and XLA do simple fusions automatically. Triton is suggested for writing custom fused kernels. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])
- Consequences: fused `x.cos().cos()` ≈ cost of `x.cos()`, which is why `gelu` costs about the same as `relu`. Recomputing values instead of storing them (rematerialization / activation checkpointing) can reduce *both* memory and runtime. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])
- Helps both bandwidth-bound and overhead-bound workloads, since fewer kernels also means fewer launches. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

## How much memory-bound time remains

- In a 336M-parameter transformer, memory-bound intermediate ops (softmax, layernorm, biases, ReLU, dropout) are still **43% of total latency** even after the paper's fused kernels, while contributing a fraction of a percent of flops. (src: [[2026-11-12-transformer-inference-arithmetic]])
- In plain PyTorch (unfused), scaled softmax took 453 µs against 0.2 GFLOP, while q/k/v projection took 333 µs for 25.8 GFLOP — the memory-bound signature. Fusion cut softmax only to 433 µs. (src: [[2026-11-12-transformer-inference-arithmetic]]; 433 µs read from the post's Table A.1)
- Counting memory passes is a diagnosis: softmax needs 1 read + 1 write in theory but commonly does 4 passes, and `q·k` 2 reads + 1 write, so the measured time reveals the missing fusion. (src: [[2026-11-12-transformer-inference-arithmetic]])
- The share shrinks with model width: per-layer traffic grows ~`d_model` while flops grow ~`d_model²`, so at 52B parameters (with the same optimisations) these ops are ~5% of latency. (src: [[2026-11-12-transformer-inference-arithmetic]])

## Why it matters for a GPU cloud

- Customer-side optimization: the provider doesn't control it. It does explain why the same GPU gives very different throughput across frameworks, compilers, and versions, which helps when triaging "slow GPU" reports. (general knowledge)

## Related

- [[memory-bandwidth]]: the cost fusion removes.
- [[framework-overhead]]: fusion also cuts kernel launches.
- [[gpu-performance-regimes]]: listed as a fix for two of the three regimes.
