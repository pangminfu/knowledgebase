---
type: concept
tags: [provisioning, tenancy-quotas, metering-billing]
sources: [2026-11-12-transformer-inference-arithmetic, 2026-11-21-what-is-llm-inference]
created: 2026-11-12
updated: 2026-11-22
---

# Inference batching

**Batching** in model inference means serving several requests (or several tokens) in one pass through the model, so the cost of reading the weights is shared instead of paid per request. In training, the batch is a fixed number of examples; in generation, the useful axis is **tokens in flight** — batch size × context length — because every token needs the whole model applied to it. Batch size is the knob that trades latency against throughput against cost. (src: [[2026-11-12-transformer-inference-arithmetic]])

## Details

### Why bigger batches are cheaper

- Reading the weights costs the same regardless of batch size: `2 · P` bytes per pass for `P` parameters. Compute costs `B · 2 · P` flops for batch `B`. So throughput per request improves with `B` until compute becomes the limit. (src: [[2026-11-12-transformer-inference-arithmetic]])
- The crossover is the hardware's flops-per-byte ratio: 312e12 flops/s ÷ 1.5e12 bytes/s = **208** for an [[nvidia-a100]]. Below ~208 tokens in flight you are [[memory-bandwidth]]-bound; above, compute-bound ([[gpu-performance-regimes]]). (src: [[2026-11-12-transformer-inference-arithmetic]])
- Concretely: a 52B model's forward pass takes ~69 ms per token-set on one GPU for up to 208 tokens (about 17 ms across four GPUs), and doubles at 416 tokens. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Past the crossover, enlarging the batch does not make anything faster per token — it only increases total work. (src: [[2026-11-12-transformer-inference-arithmetic]])

### What batch size changes, per the source's own summary

![[raw/assets/tia-batch-size.png]]
Left to right: small batch means memory is mostly weights, the bottleneck is bandwidth, and communication shows up as latency; large batch means memory is mostly the cache, the bottleneck is flops, and communication shows up as throughput.

| Axis | Small batch | Large batch |
| --- | --- | --- |
| GPU memory | mostly weights | mostly [[kv-cache]] |
| Bottleneck | memory bandwidth | flops |
| Comms | latency term | throughput term |
| Cache worth it? | maybe not — recompute instead | yes |

(src: [[2026-11-12-transformer-inference-arithmetic]])

### Capacity is the upper bound

- Batch size cannot exceed what memory allows: `(VRAM − weights) ÷ (bytes per cached token × context length)`. On 4×40 GB A100s with a 52B model, that is ~23,000 tokens. (src: [[2026-11-12-transformer-inference-arithmetic]]) See [[gpu-memory-capacity]].
- That is why the same model can be cheap (big batch) at one provider and expensive (small batch) at another with smaller cards. (general knowledge)

### Choosing a batch size

- **Low-load API (interactive):** batches are small, so memory-bound; dropping the [[kv-cache]] and recomputing can be the right call, and the goal is per-request latency. (src: [[2026-11-12-transformer-inference-arithmetic]])
- **Batch workload (mass generation, e.g. AlphaCode-style jobs):** use as many chips as available and the largest batch that fits. (src: [[2026-11-12-transformer-inference-arithmetic]])
- **Loaded API:** serve the smallest batch size that reaches flops-bound, keeping spare capacity for latency spikes. (src: [[2026-11-12-transformer-inference-arithmetic]])
- The post frames these as absolute cases rather than preferences: the load profile determines the batch strategy. (src: [[2026-11-12-transformer-inference-arithmetic]])

## Why it matters for a GPU cloud

- **Metering and billing:** cost per token falls with batch size, so a provider selling "tokens" is really selling *batching efficiency*; GPU-hours and tokens-per-second are different products. (general knowledge; the arithmetic is src)
- **Tenancy and quotas:** concurrency limits are batch-size limits. A tenant allowed "one replica" gets whatever concurrency memory permits, not a fixed request count. (general knowledge)
- **Scheduling:** real serving mixes requests with different context lengths, which static batching cannot pack efficiently. That is why continuous/in-flight batching and paged caches exist. (general knowledge; the post's model assumes a uniform batch)
- **Where it runs:** batching across concurrent requests is a built-in feature of [[inference-frameworks]] and one of the core [[inference-serving]] responsibilities. The handbook lists continuous batching ("dynamically grouping requests for better GPU utilization") first among optimizations. Roadmap: [[inference-optimization]]. (src: [[2026-11-21-what-is-llm-inference]])
- **SLA design:** latency and throughput are the same knob at different settings, so an SLA needs to say which one is guaranteed. (src + general knowledge)

## Open questions

- The post assumes every sequence in a batch has the same context length; how much efficiency is lost when they differ (padding, fragmentation), and what does continuous batching recover? (general knowledge)
- At what batch size does per-request latency (queueing + one decode step) become unacceptable for interactive use?

## Related

- [[inference-latency-model]]: the two-batch-regime equations.
- [[kv-cache]]: the memory that limits batch size.
- [[gpu-memory-capacity]]: the hard ceiling.
- [[model-parallelism]]: trading GPUs for latency at fixed batch.
- [[gpu-performance-regimes]]: the bound you are in at each batch size.
- [[2026-11-12-sizing-a-gpu-inference-deployment]]: batch size as the cost lever in a full deployment sizing.
- [[inference-optimization]]: continuous batching and the other techniques that raise tokens per GPU.
- [[per-token-pricing]]: batch efficiency is the margin in per-token prices.
