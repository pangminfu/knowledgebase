---
type: entity
tags: [provisioning]
sources: [2026-11-10-deep-learning-brrrr-first-principles, 2026-11-12-transformer-inference-arithmetic]
created: 2026-11-10
updated: 2026-11-24
---

# NVIDIA A100

NVIDIA's datacenter GPU from the Ampere generation, launched in 2020. The source (2022) calls it "generally speaking, the best GPUs we can get for inference", and it is still offered as a cloud SKU. It comes in two **form factors** (physical packages): PCIe cards and SXM modules, which mount on special server boards. Each comes with 40 GB or 80 GB of memory. (inference quote: src: [[2026-11-12-transformer-inference-arithmetic]]; generation/year/SXM description: general knowledge; form factors and memory sizes: src: [[2026-11-10-deep-learning-brrrr-first-principles]])

## Details

![[raw/assets/brrr-a100_specs.png]]
NVIDIA spec table, the same across all four variants (40/80 GB × PCIe/SXM): FP64 9.7 TFLOPS; FP64 Tensor Core 19.5; FP32 19.5; TF32 156 (312*); BF16 and FP16 Tensor Core 312 (624*). (src: [[2026-11-10-deep-learning-brrrr-first-principles]]) The `*` figures assume structured sparsity. (general knowledge)

- 312 TFLOPS needs Tensor Core matmuls; non-matmul work gets 19.5, or ~9.75 for fully general code. See [[flops]]. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])
- Memory bandwidth ≈ 1.5 TB/s, per the source. See [[memory-bandwidth]]. (src: [[2026-11-10-deep-learning-brrrr-first-principles]]) NVIDIA lists ~1.6 TB/s for 40 GB and ~2 TB/s for 80 GB variants. (general knowledge)
- Successors include H100, H200, and the Blackwell generation (B200 and later). (general knowledge)
- GPU-to-GPU bandwidth used in inference arithmetic: **300 GB/s unidirectional** over NVLink. NVIDIA's 600 GB/s figure adds both directions. (src: [[2026-11-12-transformer-inference-arithmetic]])
- As of the source (2022) the 80 GB variant was not offered by large clouds, so 40 GB was the practical ceiling — which is what forced 52B-parameter models onto 3–4 GPUs. (src: [[2026-11-12-transformer-inference-arithmetic]]) See [[gpu-memory-capacity]].

## Why it matters for a GPU cloud

- The variants are separate SKUs. SXM parts allow NVLink (fast GPU-to-GPU links) and higher power, while PCIe fits standard servers. That affects host design, pricing, and which multi-GPU jobs a node can run. (general knowledge)

## Related

- [[flops]]: what the spec-sheet numbers mean.
- [[memory-bandwidth]]: the bandwidth figure.
- [[nvidia-smi]]: tool for inspecting it.
