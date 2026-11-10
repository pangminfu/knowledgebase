---
type: concept
tags: [career, metering-billing]
sources: [2026-11-10-deep-learning-brrrr-first-principles]
created: 2026-11-10
updated: 2026-11-10
---

# GPU performance regimes

A deep-learning program's GPU time goes to one of three places: **compute** (doing arithmetic), **memory bandwidth** (moving data between GPU memory and the chips doing the math), or **overhead** (everything else). Whichever dominates is the program's *regime*, described as being "compute-bound", "memory-bound" (or "bandwidth-bound"), or "overhead-bound". The regime decides which optimizations can help: fixing the wrong one does nothing. Analogy: a factory (compute) supplied by trucks from a warehouse (bandwidth), waiting on paperwork from head office (overhead). (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

## Details

- The three components: compute = time spent on floating-point operations ([[flops]]); memory = time moving tensors (multi-dimensional arrays of numbers) within the GPU; overhead = everything else. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])
- Compute is the one cost you mostly can't cut without changing the math itself, so the goal is to spend as much time as possible compute-bound. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])
- Doubling a GPU's compute without doubling its bandwidth doesn't double real performance:

![[raw/assets/brrr-factory_doubled.png]]
A factory twice the size, still fed by the same conveyor belts. Output can't double because materials don't arrive fast enough. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

**How to tell which regime you're in** (src: [[2026-11-10-deep-learning-brrrr-first-principles]]):

| Regime | Symptom / test | Fixes |
| --- | --- | --- |
| Compute-bound | Achieved FLOPS close to peak. 80% of peak ⇒ at least 80% compute-bound | Use Tensor Cores; buy faster GPUs |
| Memory-bound | Little math per byte loaded (e.g. pointwise ops like `cos`, multiply-by-2). FLOPS far below peak while bandwidth is near peak | [[operator-fusion]] |
| Overhead-bound | Doubling batch size raises runtime only slightly (e.g. +10%). Profiler shows gaps between GPU kernels | Tracing, operator fusion, avoid Python, JIT compilers ([[framework-overhead]]) |

- Large models usually mix compute-bound and memory-bound parts, so the regime is a matter of degree. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])
- Caveat: a bigger batch can also raise *compute intensity* (math per byte), e.g. a [B, D]×[D, D] matmul with B < D. So the batch-size test isn't purely about overhead. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

## Why it matters for a GPU cloud

- A customer's GPU can look fully busy while doing a small fraction of its peak math. Utilization dashboards and billing need to say clearly what they measure. See [[gpu-utilization-metrics]]. (general knowledge)
- When a customer reports "slow GPUs", triage starts by separating a workload in the wrong regime (the customer's problem) from a hardware or host fault (the provider's problem). (general knowledge)
- An overhead-bound customer on a top-tier GPU is overpaying; a smaller SKU might perform the same. This matters for right-sizing advice and capacity planning. (general knowledge)

## Interview talking points

- Name the three regimes, a test for each, and a fix for each (table above).
- Explain why the industry trend (compute outgrowing bandwidth) makes memory-bound behaviour more common over time. See [[memory-bandwidth]].

## Related

- [[flops]]: the compute side; peak vs achieved.
- [[memory-bandwidth]]: the bandwidth side; compute intensity.
- [[framework-overhead]]: the overhead side.
- [[operator-fusion]]: main fix for memory-bound work.
- [[gpu-utilization-metrics]]: how "busy" gets measured and misread.
