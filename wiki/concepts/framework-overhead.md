---
type: concept
tags: [metering-billing]
sources: [2026-11-10-deep-learning-brrrr-first-principles]
created: 2026-11-10
updated: 2026-11-24
---

# Framework overhead

**Overhead** is GPU-program time spent on anything other than computing or moving tensors. Examples: the Python interpreter, the ML framework (e.g. PyTorch) deciding which kernel to run, and the CPU *launching* GPU kernels (not running them). GPUs are so fast that this "figuring out what to do" can leave a GPU idle, which the source calls "an expensive paperweight". (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

## Details

- Scale: an [[nvidia-a100]] does 312 TFLOPS, while Python does ~32 million additions/sec (the author's local benchmark). In the time Python does 1 operation, the A100 could do ~9.75 million FLOPs. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])
- PyTorch adds dispatch layers on top: ~280K ops/sec on tiny tensors. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])
- What `a + b` costs before any math: Python looks up `__add__` → PyTorch inspects the tensor (dtype, device, autograd) to choose a kernel → PyTorch launches the kernel. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

![[raw/assets/brrr-flamegraph.png]]
Flamegraph (a profiler chart where width = time spent) of one PyTorch addition. Only a tiny box is actual computation; the rest is overhead. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

- Overhead is hidden when GPU ops are large, because PyTorch runs **asynchronously**: the CPU queues kernels ahead while the GPU works. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

![[raw/assets/brrr-sketch-1.svg]]
Left: big GPU ops, so the CPU runs ahead and the GPU is never idle. Right: small GPU ops, so each finishes before the CPU queues the next and the GPU sits idle between them. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

- Diagnosis: overhead doesn't grow with problem size, but compute and memory do. Double the batch size; if runtime rises far less than 2×, you're overhead-bound. Or use the PyTorch profiler: (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

![[raw/assets/brrr-overhead_tracer.png]]
Profiler trace: CPU-side `aten::cos` calls (top) and the GPU kernels they launch (bottom). The GPU row has large gaps; the GPU waits on CPU overhead. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

![[raw/assets/brrr-no_overhead.png]]
Healthy trace: the CPU runs far ahead of the GPU, so the GPU stays continuously busy. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

- Fixes: tracing (`jit.trace`, `FX`, `jax.jit`) or CUDA Graphs (record a sequence of GPU launches and replay it), which trade away flexibility; the author hoped a JIT working at the Python VM (bytecode) level, TorchDynamo, could keep both. Also [[operator-fusion]] and avoiding Python. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

## Why it matters for a GPU cloud

- Overhead-bound workloads show gaps in GPU activity, so time-based utilization (see [[gpu-utilization-metrics]]) is low even though the customer pays for the full GPU-hour. These customers are candidates for smaller or cheaper SKUs. (general knowledge)

## Related

- [[gpu-performance-regimes]]: overhead is the third regime.
- [[gpu-utilization-metrics]]: GPU-Util shows overhead gaps.
- [[operator-fusion]]: fewer kernels means fewer launches.
