---
type: concept
tags: [metering-billing]
sources: [2026-11-10-deep-learning-brrrr-first-principles, 2026-11-12-transformer-inference-arithmetic]
created: 2026-11-10
updated: 2026-11-24
---

# GPU utilization metrics

Numbers meant to say how "busy" a GPU is. They measure very different things. **Time-based** metrics such as `nvidia-smi` GPU-Util report whether *any* work is running. **Throughput-based** metrics such as achieved FLOPS as a % of peak report *how much* of the hardware's capacity is in use. A GPU can score 100% on the first and a few percent on the second. For a provider this distinction shapes customer dashboards, idle detection, and capacity planning. (src: [[2026-11-10-deep-learning-brrrr-first-principles]]; framing: general knowledge)

## Details

| Metric | What it measures | Source |
| --- | --- | --- |
| `nvidia-smi` GPU-Util | % of the sample period (1 s to 1/6 s, product-dependent) in which one or more kernels was executing | src: [[2026-11-10-deep-learning-brrrr-first-principles]]; general knowledge; verified: [NVML nvmlUtilization_t](https://docs.nvidia.com/deploy/nvml-api/latest/api/structnvmlUtilization__t.html) |
| Achieved FLOPS ÷ peak FLOPS | Share of compute capacity actually used; 80% ⇒ at least 80% compute-bound | src: [[2026-11-10-deep-learning-brrrr-first-principles]] |
| Achieved bandwidth ÷ peak bandwidth | Share of [[memory-bandwidth]] used | src: [[2026-11-10-deep-learning-brrrr-first-principles]] (microbenchmark) |
| MFU (model FLOPS utilization) | FLOPs the model needs per second (excluding recomputation) ÷ peak FLOPS, for a training run | general knowledge; verified: [PaLM paper, App. B](https://arxiv.org/abs/2204.02311) |
| DCGM profiling counters: `DCGM_FI_PROF_GR_ENGINE_ACTIVE`, `DCGM_FI_PROF_SM_ACTIVE`, `DCGM_FI_PROF_SM_OCCUPANCY`, `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE`, `DCGM_FI_PROF_DRAM_ACTIVE` | Fraction of time/cycles the graphics engine, the SMs (streaming multiprocessors, the GPU's compute blocks), Tensor Core pipes, and memory interface are active, averaged over an interval. SM_ACTIVE counts how many SMs are busy, so a kernel using 1/5 of the SMs for the whole interval scores 0.2 | general knowledge; verified: [DCGM Feature Overview](https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/feature-overview.html) |

- GPU-Util is a good way to eyeball [[framework-overhead]]: gaps between kernels show up as lower util. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])
- GPU-Util can't distinguish a kernel using the whole chip from one barely using it. A kernel occupying a single SM, or a memory-bound kernel, still counts as 100% busy. (general knowledge; follows from the NVML definition above)
- Naming trap: the source stresses the column is "GPU-Util", **not** "Volatile GPU-Util". In `nvidia-smi`'s two-line header, "Volatile Uncorr. ECC" (an error counter) sits directly above "GPU-Util" in the same cell, so the two are easy to read as one label. (src: [[2026-11-10-deep-learning-brrrr-first-principles]]; layout explanation: general knowledge)
- LLM serving makes this concrete: a single-request decode is memory-bandwidth-bound by construction, so every kernel runs, util reads high, and achieved FLOPS sits far below peak. See [[inference-batching]]. (src: [[2026-11-12-transformer-inference-arithmetic]])

## Why it matters for a GPU cloud

- GPU clouds typically bill for *allocated* GPU time, whatever the utilization. Utilization metrics feed customer dashboards, idle-instance detection/reclaim, and capacity planning rather than the bill. (general knowledge, unverified against a provider source)
- Showing customers GPU-Util alone invites "it's at 100%, why is my training slow?" tickets. Throughput metrics answer that question better. (general knowledge)

## Related

- [[nvidia-smi]]: source of GPU-Util.
- [[gpu-performance-regimes]]: what the numbers diagnose.
- [[flops]]: peak vs achieved.
- [[framework-overhead]]: what time-based util exposes.
- [[control-bands]]: alerting on a metric's normal range; only as good as the metric chosen.

## Open questions

- How do major GPU clouds (AWS, GCP, CoreWeave, Lambda, etc.) actually meter, and which utilization metrics do they expose to customers?
- Does any provider bill on utilization or throughput rather than allocation time?
