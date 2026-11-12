---
type: concept
tags: [provisioning, tenancy-quotas, metering-billing, career]
sources: [2026-11-12-transformer-inference-arithmetic]
created: 2026-11-12
updated: 2026-11-12
---

# GPU memory capacity

**GPU memory capacity** is how many bytes of DRAM a GPU (or a set of GPUs) can hold at once. For inference it is the hard limit on *how much model and how many concurrent requests* a replica can serve. Unlike speed, it cannot be optimised away: if the weights plus the per-request [[kv-cache]] do not fit, requests are queued or refused. Analogy: a warehouse's floor area, not its loading docks. (src: [[2026-11-12-transformer-inference-arithmetic]])

## Details

- Three occupants of GPU memory during inference: **weights** (the model's parameters), the **kv cache** (one entry per token in flight), and **intermediate activations** (temporaries). The source treats the third as negligible at these sizes. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Weights at 16-bit precision: parameters × 2 bytes. A 52B model is 104 GB, so it does **not** fit on a 40 GB [[nvidia-a100]] — three GPUs minimum, four in practice, since splitting powers of two over three chips wastes capacity. (src: [[2026-11-12-transformer-inference-arithmetic]])
- The leftover after weights is the cache budget. Four A100-40GB = 160 GB, minus 104 GB of weights, leaves 56 GB ≈ 23,000 cached tokens (2 MB each) — e.g. 4 concurrent requests of 2048 tokens. Three GPUs leave 16 GB ≈ 8,000 tokens, e.g. batch 4 × 2048. (src: [[2026-11-12-transformer-inference-arithmetic]])
- So for a fixed model and GPU count, **max concurrency is a memory calculation**: `(total VRAM − weights − overhead) ÷ bytes per cached token ÷ context length`. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Memory is why replicas exist at all. If traffic exceeds one replica's batch capacity, you add replicas (more GPUs) rather than growing the batch indefinitely. (src: [[2026-11-12-transformer-inference-arithmetic]])
- The post notes the 80 GB A100 variant was not offered by large clouds at the time (2022), so capacity was effectively capped at 40 GB per GPU. (src: [[2026-11-12-transformer-inference-arithmetic]])

## Why it matters for a GPU cloud

- **Provisioning:** GPU memory size is a SKU axis (40 vs 80 GB, HBM generations), and the *quotable* fact for a customer is "this SKU holds models up to X and contexts up to Y at concurrency Z". (general knowledge)
- **Tenancy and quotas:** concurrency limits are memory limits in disguise. Quota systems that count GPUs but not context length will overcommit and hit out-of-memory at admission time. (general knowledge)
- **Scheduling/lifecycle:** a serving replica must reserve weights plus worst-case cache, or be prepared to reject requests; memory pressure is a first-class scheduling signal, not just an alert. (general knowledge)
- **Billing:** cache memory held for an idle conversation is real cost. Providers that charge per token usually still charge for the reserved memory (or evict the session). (general knowledge)

## Related

- [[kv-cache]]: the per-request consumer of leftover memory.
- [[inference-batching]]: batch size is bounded above by capacity.
- [[model-parallelism]]: how a model too large for one GPU is spread across several.
- [[nvidia-a100]]: the 40/80 GB SKU examples.
- [[memory-bandwidth]]: the other, speed-related memory property.
- [[2026-11-12-sizing-a-gpu-inference-deployment]]: the whole chain in one worked example.
