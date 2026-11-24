---
type: entity
tags: [metering-billing, lifecycle]
sources: [2026-11-10-deep-learning-brrrr-first-principles]
created: 2026-11-10
updated: 2026-11-24
---

# nvidia-smi

NVIDIA's command-line tool ("System Management Interface") for checking the state of the GPUs on a machine. It prints a table per GPU covering memory used, utilization, temperature, power, error counts, and running processes. It's usually the first command anyone runs on a GPU box. (general knowledge; verified: [nvidia-smi documentation](https://docs.nvidia.com/deploy/nvidia-smi/index.html))

## Details

- The memory it reports is the GPU's DRAM, the pool behind "CUDA out of memory" errors. See [[memory-bandwidth]]. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])
- **GPU-Util** = % of time a GPU kernel is running. Useful for spotting [[framework-overhead]], but it does not measure how much of the GPU's capacity is used. See [[gpu-utilization-metrics]]. (src: [[2026-11-10-deep-learning-brrrr-first-principles]]) Precisely: the percent of the past sample period (between 1 s and 1/6 s depending on the product) during which one or more kernels was executing, as reported by NVML, the library nvidia-smi is built on. (general knowledge; verified: [NVML nvmlUtilization_t](https://docs.nvidia.com/deploy/nvml-api/latest/api/structnvmlUtilization__t.html))
- The column is "GPU-Util", not "Volatile GPU-Util". (src: [[2026-11-10-deep-learning-brrrr-first-principles]]) In `nvidia-smi`'s two-line header, "Volatile Uncorr. ECC" sits directly above "GPU-Util" in the same cell, so the two are easy to read as one label. "Volatile" refers to ECC error counts since the last driver load. (general knowledge; verified: [nvidia-smi documentation](https://docs.nvidia.com/deploy/nvidia-smi/index.html))

## Why it matters for a GPU cloud

- Node health checks and break/fix triage commonly start with nvidia-smi: is the GPU visible, are there ECC (memory error-correction) errors, is it throttling on temperature or power? This ties into bare-metal lifecycle ([[kubernetes-node]]). (general knowledge)
- Fleet-scale monitoring usually uses NVIDIA DCGM (Data Center GPU Manager), e.g. via `dcgm-exporter` for Prometheus, rather than parsing nvidia-smi output. (general knowledge; verified: [DCGM-Exporter docs](https://docs.nvidia.com/datacenter/dcgm/latest/gpu-telemetry/dcgm-exporter.html))

## Related

- [[gpu-utilization-metrics]]: what GPU-Util does and doesn't mean.
- [[nvidia-a100]]: an example GPU it reports on.
- [[kubectl]]: `kubectl exec <pod> -- nvidia-smi` checks GPUs from inside a Pod.
