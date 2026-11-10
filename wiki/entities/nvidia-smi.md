---
type: entity
tags: [metering-billing, lifecycle]
sources: [2026-11-10-deep-learning-brrrr-first-principles]
created: 2026-11-10
updated: 2026-11-10
---

# nvidia-smi

NVIDIA's command-line tool ("System Management Interface") for checking the state of the GPUs on a machine. It prints a table per GPU covering memory used, utilization, temperature, power, error counts, and running processes. It's usually the first command anyone runs on a GPU box. (general knowledge)

## Details

- The memory it reports is the GPU's DRAM, the pool behind "CUDA out of memory" errors. See [[memory-bandwidth]]. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])
- **GPU-Util** = % of time a GPU kernel is running. Useful for spotting [[framework-overhead]], but it does not measure how much of the GPU's capacity is used. See [[gpu-utilization-metrics]]. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])
- The column is "GPU-Util", not "Volatile GPU-Util". The word "Volatile" belongs to the adjacent ECC column. (src: [[2026-11-10-deep-learning-brrrr-first-principles]]; column explanation: general knowledge)

## Why it matters for a GPU cloud

- Node health checks and break/fix triage commonly start with nvidia-smi: is the GPU visible, are there ECC (memory error-correction) errors, is it throttling on temperature or power? This ties into bare-metal lifecycle. (general knowledge)
- Fleet-scale monitoring usually uses NVIDIA DCGM rather than parsing nvidia-smi output. (general knowledge)

## Related

- [[gpu-utilization-metrics]]: what GPU-Util does and doesn't mean.
- [[nvidia-a100]]: an example GPU it reports on.
