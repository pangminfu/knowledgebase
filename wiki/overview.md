---
type: overview
tags: [provisioning, tenancy-quotas, lifecycle, metering-billing, career]
sources: [2026-11-10-deep-learning-brrrr-first-principles]
created: 2026-11-10
updated: 2026-11-10
---

# Overview

**Goal:** working knowledge of GPU cloud control plane engineering across four domains: provisioning APIs, tenancy and quotas, Kubernetes/bare-metal lifecycle, and metering and billing.

## The product: what a GPU actually does

The first source covers the thing the control plane sells, not the control plane itself. A GPU's real throughput is limited by one of three things: compute, memory bandwidth, or software overhead ([[gpu-performance-regimes]]). Headline numbers like "312 TFLOPS" hold only under narrow conditions ([[flops]], [[nvidia-a100]]). (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

Control-plane consequences (general knowledge unless cited):

- **Metering-billing:** "busy" is ambiguous. `nvidia-smi` GPU-Util measures time-busy, not capacity used (src: [[2026-11-10-deep-learning-brrrr-first-principles]]), so dashboards and idle-detection have to choose their metrics deliberately. See [[gpu-utilization-metrics]].
- **Provisioning:** SKUs differ in form factor, memory size, and bandwidth. Which one fits depends on the customer's regime. See [[memory-bandwidth]].
- **Lifecycle:** [[nvidia-smi]] is the first-line tool for node health and "slow GPU" triage.

## Coverage so far

| Domain | Status |
| --- | --- |
| Provisioning APIs | Only GPU/SKU background; no API sources yet |
| Tenancy and quotas | Not covered |
| Kubernetes and bare-metal lifecycle | Not covered beyond nvidia-smi |
| Metering and billing | Utilization metrics only; no billing-pipeline sources |
| Career | Interview talking points on GPU performance |

## Suggested next sources

- A provider's GPU instance docs and pricing page (e.g. AWS EC2 P-series, GCP A3, CoreWeave): provisioning + billing granularity.
- Kubernetes docs on device plugins and the NVIDIA GPU Operator: lifecycle.
- NVIDIA DCGM docs: real utilization metrics for metering.
- Kubernetes ResourceQuota / cloud quota docs: tenancy and quotas.
