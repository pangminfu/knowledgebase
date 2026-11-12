---
type: overview
tags: [provisioning, tenancy-quotas, lifecycle, metering-billing, career]
sources: [2026-11-10-deep-learning-brrrr-first-principles, 2026-11-12-transformer-inference-arithmetic]
created: 2026-11-10
updated: 2026-11-12
---

# Overview

**Goal:** working knowledge of GPU cloud control plane engineering across four domains: provisioning APIs, tenancy and quotas, Kubernetes/bare-metal lifecycle, and metering and billing.

## The product: what a GPU actually does

A GPU's real throughput is limited by one of three things: compute, memory bandwidth, or software overhead ([[gpu-performance-regimes]]). Headline numbers like "312 TFLOPS" hold only under narrow conditions ([[flops]], [[nvidia-a100]]). (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

Control-plane consequences (general knowledge unless cited):

- **Metering-billing:** "busy" is ambiguous. `nvidia-smi` GPU-Util measures time-busy, not capacity used (src: [[2026-11-10-deep-learning-brrrr-first-principles]]), so dashboards and idle-detection have to choose their metrics deliberately. See [[gpu-utilization-metrics]].
- **Provisioning:** SKUs differ in form factor, memory size, and bandwidth. Which one fits depends on the customer's regime. See [[memory-bandwidth]].
- **Lifecycle:** [[nvidia-smi]] is the first-line tool for node health and "slow GPU" triage.

## Serving the product: inference arithmetic

The second source turns the same physics into a sizing procedure for model serving, which is where most GPU-cloud demand now comes from. (src: [[2026-11-12-transformer-inference-arithmetic]])

The chain is four arithmetic steps, laid out end to end in [[2026-11-12-sizing-a-gpu-inference-deployment]]:

1. **Weights → GPUs.** 2 bytes per parameter at 16-bit; a 52B model is 104 GB, so it cannot fit on a 40 GB card ([[gpu-memory-capacity]]).
2. **Leftover memory → concurrency.** The [[kv-cache]] costs ~2 MB per token for that model, so the remaining 56 GB on four cards is ~23,000 cached tokens.
3. **Concurrency → cost.** Batching below ~208 tokens in flight leaves the GPU memory-bandwidth-bound; above it, flops-bound and cheaper per token ([[inference-batching]]).
4. **GPUs → latency floor.** Tensor parallelism cuts compute by `1/N` but adds a fixed four-per-layer communication cost, so more GPUs stop helping past a point ([[model-parallelism]], [[inference-latency-model]]).

Control-plane consequences (general knowledge unless cited):

- **Quotas are memory limits.** Concurrency follows from `(VRAM − weights) ÷ (cache per token × context)`, not from a GPU count (src: [[2026-11-12-transformer-inference-arithmetic]]).
- **Multi-GPU replicas are gang-scheduled.** A model too large for one card needs all the GPUs of a node, wired with NVLink — a topology constraint, not a quantity constraint. (general knowledge)
- **Tokens/second is the inference-native metering unit**, and it is not fixed per SKU: it depends on the batch the provider can pack (src: [[2026-11-12-transformer-inference-arithmetic]]; product framing: general knowledge).
- **Memory-bound replicas look busy.** GPU-Util can read high while the GPU does a fraction of its peak flops ([[gpu-utilization-metrics]]).

## Coverage so far

| Domain | Status |
| --- | --- |
| Provisioning APIs | GPU/SKU background plus inference sizing rules (how many GPUs a model needs); no actual API sources yet |
| Tenancy and quotas | Concurrency-as-memory derived from inference arithmetic; no quota/tenancy system sources yet |
| Kubernetes and bare-metal lifecycle | Not covered beyond nvidia-smi and the gang-scheduling constraint |
| Metering and billing | Utilization metrics and tokens-vs-GPU-hours reasoning; no billing-pipeline sources |
| Career | Interview talking points on GPU performance, capacity, batching, and parallelism |

## Suggested next sources

- A provider's GPU instance docs and pricing page (e.g. AWS EC2 P-series, GCP A3, CoreWeave): provisioning, billing granularity, and how they present concurrency.
- vLLM / TensorRT-LLM documentation on continuous batching and paged attention: the scheduling half of serving that both sources omit.
- Kubernetes docs on device plugins, the NVIDIA GPU Operator, and gang scheduling: lifecycle.
- Kubernetes ResourceQuota and cloud quota docs: tenancy and quotas.
- NVIDIA DCGM docs: real utilization metrics for metering.
