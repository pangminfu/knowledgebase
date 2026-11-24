---
type: entity
tags: [provisioning]
sources: [2026-11-10-deep-learning-brrrr-first-principles, 2026-11-12-transformer-inference-arithmetic]
created: 2026-11-10
updated: 2026-11-24
---

# NVIDIA A100

NVIDIA's datacenter GPU from the Ampere generation, launched in May 2020. It comes in two **form factors** (physical packages): PCIe cards, which plug into standard servers, and SXM4 modules, which mount on NVIDIA HGX server boards. Each comes with 40 GB or 80 GB of memory. (form factors and memory sizes: src: [[2026-11-10-deep-learning-brrrr-first-principles]]; HGX boards: general knowledge; verified: [NVIDIA A100 datasheet](https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/a100/pdf/nvidia-a100-datasheet-us-nvidia-1758950-r4-web.pdf); generation and launch date: general knowledge) An inference source (2022) calls A100s "generally speaking, the best GPUs we can get for inference". (src: [[2026-11-12-transformer-inference-arithmetic]])

## Details

![[raw/assets/brrr-a100_specs.png]]
NVIDIA spec table, the same across all four variants (40/80 GB × PCIe/SXM): FP64 9.7 TFLOPS; FP64 Tensor Core 19.5; FP32 19.5; TF32 156 (312*); BF16 and FP16 Tensor Core 312 (624*). (src: [[2026-11-10-deep-learning-brrrr-first-principles]]) The `*` figures assume structured sparsity; the unstarred figures are dense. (general knowledge; verified: [NVIDIA A100 datasheet](https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/a100/pdf/nvidia-a100-datasheet-us-nvidia-1758950-r4-web.pdf))

- 312 TFLOPS needs FP16/BF16 matmuls on Tensor Cores; non-matmul work gets 19.5 (FP32), or ~9.75 for fully general code without fused multiply-add. See [[flops]]. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])
- Memory bandwidth ≈ 1.5 TB/s, per both sources (src: [[2026-11-10-deep-learning-brrrr-first-principles]], [[2026-11-12-transformer-inference-arithmetic]]). See [[memory-bandwidth]] and the per-SKU table below.

| Variant | Memory | Bandwidth | Max TDP | GPU-to-GPU |
| --- | --- | --- | --- | --- |
| 40GB PCIe | 40 GB HBM2 | 1,555 GB/s | 250 W | NVLink Bridge (pairs only) 600 GB/s; PCIe Gen4 64 GB/s |
| 80GB PCIe | 80 GB HBM2e | 1,935 GB/s | 300 W | same as 40GB PCIe |
| 40GB SXM | 40 GB HBM2 | 1,555 GB/s | 400 W | NVLink 600 GB/s (up to 16 GPUs via NVSwitch) |
| 80GB SXM | 80 GB HBM2e | 2,039 GB/s | 400 W | same as 40GB SXM |

(general knowledge; verified: [NVIDIA A100 datasheet](https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/a100/pdf/nvidia-a100-datasheet-us-nvidia-1758950-r4-web.pdf))

- GPU-to-GPU bandwidth used in inference arithmetic: **300 GB/s per direction** over NVLink. NVIDIA's 600 GB/s figure adds 300 GB/s in and 300 GB/s out. (src: [[2026-11-12-transformer-inference-arithmetic]])
- As of that source (2022) the 80 GB variant (quoted there as 2 TB/s) was not offered by large clouds, so 40 GB was the practical ceiling, which is why a 52B-parameter model (~104 GB of FP16 weights) needed 3–4 GPUs (four preferred). (src: [[2026-11-12-transformer-inference-arithmetic]]) See [[gpu-memory-capacity]].
- Successors include H100, H200, and the Blackwell generation (B200 and later). (general knowledge)
- MIG (Multi-Instance GPU) can split one A100 into up to 7 isolated instances (5 GB each on 40 GB parts, 10 GB on 80 GB parts). (general knowledge; verified: [NVIDIA A100 datasheet](https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/a100/pdf/nvidia-a100-datasheet-us-nvidia-1758950-r4-web.pdf))

## Why it matters for a GPU cloud

- The variants are separate SKUs. SXM parts run at higher power (400 W) and join NVLink/NVSwitch fabrics of 4–16 GPUs on HGX boards, while PCIe cards fit standard servers and can only NVLink-bridge in pairs. That affects host design, pricing, and which multi-GPU jobs a node can run. (general knowledge)

## Related

- [[flops]]: what the spec-sheet numbers mean.
- [[memory-bandwidth]]: the bandwidth figure.
- [[nvidia-smi]]: tool for inspecting it.

## Contradictions

- Bandwidth: both sources use 1.5 TB/s (src: [[2026-11-10-deep-learning-brrrr-first-principles]], [[2026-11-12-transformer-inference-arithmetic]]); NVIDIA's datasheet gives 1,555 GB/s for 40 GB parts and 1,935/2,039 GB/s for 80 GB PCIe/SXM (general knowledge; verified: [NVIDIA A100 datasheet](https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/a100/pdf/nvidia-a100-datasheet-us-nvidia-1758950-r4-web.pdf)). The inference source acknowledges 1.555 would be more accurate. The rounding is harmless for back-of-envelope math but shifts ratios a few percent (e.g. 208 vs ~201 flops/byte).
