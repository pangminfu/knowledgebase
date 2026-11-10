---
type: concept
tags: [career]
sources: [2026-11-10-deep-learning-brrrr-first-principles]
created: 2026-11-10
updated: 2026-11-10
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

## Why it matters for a GPU cloud

- Customer-side optimization: the provider doesn't control it. It does explain why the same GPU gives very different throughput across frameworks, compilers, and versions, which helps when triaging "slow GPU" reports. (general knowledge)

## Related

- [[memory-bandwidth]]: the cost fusion removes.
- [[framework-overhead]]: fusion also cuts kernel launches.
- [[gpu-performance-regimes]]: listed as a fix for two of the three regimes.
