---
type: concept
tags: [provisioning]
sources: [2026-11-10-deep-learning-brrrr-first-principles, 2026-11-12-transformer-inference-arithmetic]
created: 2026-11-10
updated: 2026-11-24
---

# Memory bandwidth

**Memory bandwidth** is how many bytes per second can move between a GPU's main memory and its compute units. A GPU has two kinds of memory. **DRAM** is large and relatively slow, like a warehouse; on datacenter GPUs this is HBM, "high-bandwidth memory" (the A100 uses HBM2/HBM2e: general knowledge; verified: [NVIDIA A100 datasheet](https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/a100/pdf/nvidia-a100-datasheet-us-nvidia-1758950-r4-web.pdf)). **SRAM** is small and fast and sits next to the compute units, like the factory floor. Every operation has to haul its data from the warehouse to the factory and back. When that hauling takes longer than the math, the operation is **memory-bound**. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

## Details

![[raw/assets/brrr-factory_bandwidth.png]]
Memory (DRAM) on the left, compute (SRAM + compute units) on the right. The conveyor belts between them, circled, are the bandwidth cost. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

- Three kinds of data movement (src: [[2026-11-10-deep-learning-brrrr-first-principles]]):
  - CPU → GPU: "data transfer cost".
  - Node → node (between machines): "network cost".
  - GPU global memory (DRAM) → on-chip shared memory (SRAM): "memory bandwidth cost". This is the post's focus.
- The DRAM is the memory `nvidia-smi` reports and the source of "CUDA out of memory" errors. See [[nvidia-smi]]. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])
- Every GPU **kernel** (one function launched on the GPU) reads its inputs from DRAM and writes its outputs back. A unary op like `torch.cos` spends nearly all its time moving data. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])
- Back-of-envelope for an [[nvidia-a100]]: 1.5 TB/s bandwidth and 19.5 TFLOPS (FP32, non-Tensor-Core). With 4-byte floats, it loads ~400 billion numbers (1.5e12 ÷ 4 = 375e9) in the time it does ~20 trillion operations, i.e. ~50 operations per number loaded. A simple op also has to write results back, so you need about 100 operations per element before compute time exceeds memory time. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])
- **Compute intensity** (arithmetic intensity) = math done per byte moved. Raising it moves you from memory-bound to compute-bound. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

![[raw/assets/brrr-microbench.png]]
Microbenchmark: repeat `x * 2` N times in one fused kernel. Runtime stays flat up to 32 repeats and only rises noticeably from 64 (memory-bound: compute idle). Achieved FLOPS climb linearly from ~0.2 TFLOPS toward the 9.75 TFLOPS general-purpose FP32 peak. Achieved bandwidth sits near peak until ~32 repeats, then drops as compute takes over. The source's split: repeat < 32 memory-bound, repeat > 64 compute-bound. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

- Compute has historically grown faster than bandwidth, so memory-bound behaviour gets more common with each hardware generation. See [[flops]]. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])
- Main fix: [[operator-fusion]]. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

## The per-SKU ratio

- Compressing the above into one number: peak flops ÷ bandwidth. For an A100, 312e12 ÷ 1.5e12 = **208 flops per byte** (FP16 Tensor Core peak ÷ the source's rounded 1.5 TB/s). Workloads below that arithmetic intensity are memory-bound; above, compute-bound. (src: [[2026-11-12-transformer-inference-arithmetic]])
- It is the fastest way to compare SKUs for inference: a card with double the flops but the same bandwidth does not help a memory-bound workload. (general knowledge; arithmetic from the source)
- With 2-byte (FP16) weights each weight costs 2 bytes and 2 flops per token, so intensity ≈ tokens per pass and the ratio becomes a token count: below ~208 tokens per forward pass (at decode, a batch below ~208 sequences), reading the weights dominates. See [[inference-batching]]. (src: [[2026-11-12-transformer-inference-arithmetic]])
- The ratio depends on the SKU's exact bandwidth: the source itself notes 1.555 TB/s would be more accurate than 1.5 (src: [[2026-11-12-transformer-inference-arithmetic]]). With datasheet values: 40 GB A100 312 ÷ 1.555 ≈ 201; 80 GB SXM 312 ÷ 2.039 ≈ 153. (general knowledge; verified: [NVIDIA A100 datasheet](https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/a100/pdf/nvidia-a100-datasheet-us-nvidia-1758950-r4-web.pdf))

## Why it matters for a GPU cloud

- GPU memory size and bandwidth are SKU dimensions (e.g. A100 40GB at 1,555 GB/s vs 80GB SXM at 2,039 GB/s: general knowledge; verified: [NVIDIA A100 datasheet](https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/a100/pdf/nvidia-a100-datasheet-us-nvidia-1758950-r4-web.pdf)). Memory-bound customers gain from more bandwidth, not more TFLOPS. (general knowledge)
- The two costs the post skips, CPU↔GPU transfer and node↔node networking, are mostly the provider's job: host design, PCIe/NVLink, and cluster networking for multi-node jobs. (general knowledge)

## Related

- [[gpu-performance-regimes]]: bandwidth is the "memory" regime.
- [[operator-fusion]]: cuts DRAM round-trips.
- [[flops]]: the other side of compute intensity.
- [[nvidia-smi]]: reports DRAM usage.
