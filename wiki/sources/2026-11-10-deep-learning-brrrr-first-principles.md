---
type: source
tags: [metering-billing, provisioning]
sources: []
raw: raw/2026-11-10-making-deep-learning-go-brrrr.md
url: https://horace.io/brrr_intro.html
author: Horace He
created: 2026-11-10
updated: 2026-11-24
---

# Making Deep Learning Go Brrrr From First Principles

Raw: [[raw/2026-11-10-making-deep-learning-go-brrrr]] · Horace He (PyTorch team), 2022.

## Summary

A first-principles guide to why deep-learning code runs slower than a GPU's spec sheet suggests. GPU time goes to three things: **compute** (doing the math), **memory bandwidth** (moving data between the GPU's big, slow memory and its small, fast on-chip memory), and **overhead** (everything else: Python, framework bookkeeping, launching work). Whichever one dominates is your "regime". Only fixes aimed at that regime help. The examples use PyTorch on an NVIDIA A100, but the author says the principles carry over to other hardware and frameworks.

**Why a control-plane engineer should care:** the GPU is the product you provision, isolate, and bill. This post explains why a customer can rent a "312 TFLOPS" GPU, see "100% GPU-Util", and still get a small fraction of that throughput. That gap comes up in [[gpu-utilization-metrics]] (metering dashboards), in SKU choice ([[nvidia-a100]]), and in "my GPU is slow" support tickets. (general knowledge)

![[raw/assets/brrr-factory.png]]
The author's central analogy: memory is a warehouse, compute is a factory, and conveyor belts move materials between them. Instructions sent to the factory are overhead, and the belts are memory bandwidth.

## Key points

- Three components of GPU time: compute (FLOPS), memory (moving tensors within the GPU), overhead (everything else). See [[gpu-performance-regimes]].
- An A100's 312 TFLOPS applies only to FP16/BF16 matrix multiplies on Tensor Cores (dense; 624 with structured sparsity). Non-matmul FP32 work gets 19.5 TFLOPS on paper, and only ~9.75 for fully general code because the 19.5 relies on fused multiply-add (FMA) hardware. (The halving reflects an FMA being counted as 2 FLOPs: general knowledge.) See [[flops]].
- On BERT (table from the "Data Movement Is All You Need" paper), non-matmul ops are 0.2% of FLOPs but about 39% of runtime (25.5% normalization + 13.5% element-wise). Normalization runs at 250× fewer FLOPS than matmuls, pointwise ops at 700× fewer.
- Compute grows faster than memory bandwidth (2004 CACM table: processor bandwidth doubles every ~1.6 yr vs memory modules ~2.7 yr), so keeping GPUs busy keeps getting harder. See [[memory-bandwidth]].
- Every GPU kernel reads its inputs from DRAM and writes its outputs back. For a simple FP32 op on an A100 (1.5 TB/s, 19.5 TFLOPS) you need roughly 100 operations per element before compute time exceeds memory time.
- [[operator-fusion]] (doing several ops in one pass without writing intermediate results back to memory) is called "the most important optimization in deep learning compilers". A fused `x.cos().cos()` costs about the same as a single `x.cos()`.
- In the time Python does 1 operation, an A100 can do ~9.75M FLOPs. Python does ~32M additions/sec (author's local benchmark); PyTorch on tiny tensors reaches only ~280K ops/sec. See [[framework-overhead]].
- PyTorch runs asynchronously: the CPU queues GPU work ahead of time. If each GPU operation is big enough, framework overhead is hidden.
- Overhead test: double the batch size. If runtime rises much less than 2× (e.g. +10%), you're overhead-bound.
- Microbenchmark (fused `x * 2` repeated N times): memory-bound for repeat < 32, compute-bound beyond 64; achieved FLOPS rise from ~0.2 TFLOPS toward the 9.75 peak.
- Compute-bound test: achieved FLOPS ÷ peak FLOPS. 80% means at least 80% compute-bound.
- `nvidia-smi` "GPU-Util" ≈ % of time a GPU kernel is running (NVML defines it as % of the sample period in which one or more kernels executed: general knowledge; verified: [NVML nvmlUtilization_t](https://docs.nvidia.com/deploy/nvml-api/latest/api/structnvmlUtilization__t.html)). It's a quick way to spot overhead. See [[nvidia-smi]] and [[gpu-utilization-metrics]].
- The GPU memory shown in `nvidia-smi` is the DRAM, which is what "CUDA out of memory" errors refer to.

| Regime | Fixes (from the post) |
| --- | --- |
| Overhead-bound | Tracing, operator fusion, avoid Python, a real JIT |
| Bandwidth-bound | Operator fusion |
| Compute-bound | Use Tensor Cores, "give Nvidia more money" (faster GPUs) |

## Takeaways

- "100% GPU-Util" does not mean the GPU is used well. Util measures *time busy*, not *how much of the hardware* is busy. (src + general knowledge)
- The three regimes are told apart by achieved FLOPS as a share of peak, by how runtime scales with batch size, and by gaps in a profiler trace. Customers' ML engineers think in these terms.
- Spec-sheet TFLOPS depend on precision, Tensor Core use, and (marked `*` on the A100 sheet) sparsity, so headline TFLOPS alone is not a fair way to compare SKUs. (sparsity: general knowledge; verified: [NVIDIA A100 datasheet](https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/a100/pdf/nvidia-a100-datasheet-us-nvidia-1758950-r4-web.pdf))

## Concepts & entities touched

[[gpu-performance-regimes]] · [[flops]] · [[memory-bandwidth]] · [[operator-fusion]] · [[framework-overhead]] · [[gpu-utilization-metrics]] · [[nvidia-a100]] · [[nvidia-smi]]

## Questions raised

- How do real GPU clouds meter usage: by allocation time only, or do they expose or bill on utilization? Which utilization metric do they show customers?
- What finer-grained metrics beyond nvidia-smi GPU-Util do providers collect (e.g. NVIDIA DCGM's SM-activity and Tensor-Core-activity counters)? (DCGM exposes e.g. `DCGM_FI_PROF_SM_ACTIVE` and `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE`: general knowledge; verified: [DCGM Feature Overview](https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/feature-overview.html))
- Does the post's "data transfer cost" (CPU↔GPU) and "network cost" (node↔node), which it deliberately skips, drive SKU and interconnect choices in cloud provisioning?
- The numbers are A100-era (2022). How have the compute/bandwidth ratios shifted on H100/B200?
- Small slip in the source: it says data is shipped "from our storage to the warehouse" where the analogy needs "warehouse to factory".
