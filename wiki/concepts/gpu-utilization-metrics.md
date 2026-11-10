---
type: concept
tags: [metering-billing, career]
sources: [2026-11-10-deep-learning-brrrr-first-principles]
created: 2026-11-10
updated: 2026-11-10
---

# GPU utilization metrics

Numbers meant to say how "busy" a GPU is. They measure very different things. **Time-based** metrics such as `nvidia-smi` GPU-Util report whether *any* work is running. **Throughput-based** metrics such as achieved FLOPS as a % of peak report *how much* of the hardware's capacity is in use. A GPU can score 100% on the first and a few percent on the second. For a provider this distinction shapes customer dashboards, idle detection, and capacity planning. (src: [[2026-11-10-deep-learning-brrrr-first-principles]]; framing: general knowledge)

## Details

| Metric | What it measures | Source |
| --- | --- | --- |
| `nvidia-smi` GPU-Util | % of time a GPU kernel is running | src: [[2026-11-10-deep-learning-brrrr-first-principles]] |
| Achieved FLOPS ÷ peak FLOPS | Share of compute capacity actually used; 80% ⇒ at least 80% compute-bound | src: [[2026-11-10-deep-learning-brrrr-first-principles]] |
| Achieved bandwidth ÷ peak bandwidth | Share of [[memory-bandwidth]] used | src: [[2026-11-10-deep-learning-brrrr-first-principles]] (microbenchmark) |
| MFU (model FLOPS utilization) | FLOPS % of peak for a whole training run | general knowledge |
| DCGM counters (SM activity, Tensor Core activity, etc.) | Finer-grained hardware activity | general knowledge |

- GPU-Util is a good way to eyeball [[framework-overhead]]: gaps between kernels show up as lower util. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])
- GPU-Util can't distinguish a kernel using the whole chip from one barely using it. A memory-bound kernel counts as 100% busy. (general knowledge; follows from the definition in src)
- Naming trap: the source stresses the column is "GPU-Util", **not** "Volatile GPU-Util". In `nvidia-smi`'s table layout, "Volatile" belongs to the neighbouring "Volatile Uncorr. ECC" column. (src: [[2026-11-10-deep-learning-brrrr-first-principles]]; column explanation: general knowledge)

## Why it matters for a GPU cloud

- GPU clouds typically bill for *allocated* GPU time, whatever the utilization. Utilization metrics feed customer dashboards, idle-instance detection/reclaim, and capacity planning rather than the bill. (general knowledge, unverified against a provider source)
- Showing customers GPU-Util alone invites "it's at 100%, why is my training slow?" tickets. Throughput metrics answer that question better. (general knowledge)

## Related

- [[nvidia-smi]]: source of GPU-Util.
- [[gpu-performance-regimes]]: what the numbers diagnose.
- [[flops]]: peak vs achieved.
- [[framework-overhead]]: what time-based util exposes.

## Open questions

- How do major GPU clouds (AWS, GCP, CoreWeave, Lambda, etc.) actually meter, and which utilization metrics do they expose to customers?
- Does any provider bill on utilization or throughput rather than allocation time?
