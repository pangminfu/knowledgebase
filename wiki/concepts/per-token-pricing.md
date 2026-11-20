---
type: concept
tags: [metering-billing, tenancy-quotas, career]
sources: [2026-11-21-what-is-llm-inference, 2026-11-12-transformer-inference-arithmetic, 2026-11-10-deep-learning-brrrr-first-principles]
created: 2026-11-21
updated: 2026-11-21
---

# Per-token pricing

**Per-token pricing** means billing an LLM customer for the [[llm-inference|tokens]] processed (prompt in, response out) instead of for the GPU time used. Serverless LLM APIs work this way: "send a prompt, get a response, and pay by the token," with infrastructure and scaling hidden. The alternative is **GPU-time pricing** (per GPU-hour or per second), where the customer rents the hardware and runs [[inference-serving]] themselves. Analogy: a taxi fare per kilometre versus renting the car by the day. (src: [[2026-11-21-what-is-llm-inference]]; GPU-hour contrast: general knowledge)

## Details

### Two products, two risk owners

| | Per-token (serverless API) | Per GPU-hour (self-hosted / dedicated) |
| --- | --- | --- |
| Customer pays for | Tokens in and out | Wall-clock GPU time, busy or idle |
| Who runs serving | Provider | Customer (or provider as a managed endpoint) |
| Who carries utilization risk | Provider: idle GPUs and small batches eat margin | Customer: an idle or memory-bound replica still costs full price |
| Metering unit | Token counts per tenant per request | Instance/GPU allocation time |
| Why customers leave it | Cost at scale, latency, customization, compliance (src: [[2026-11-21-what-is-llm-inference]]) | Operational burden, low utilization (general knowledge) |

(general knowledge unless cited)

### Why a token's cost is not fixed

- The GPU cost of a token depends on how many tokens share each pass over the weights. Below ~208 tokens in flight on an A100, the GPU is memory-bound and extra tokens are nearly free; above it, cost per token is flat. (src: [[2026-11-12-transformer-inference-arithmetic]]) See [[inference-batching]].
- So a per-token provider is really selling **batching efficiency**: margin = price per token − (GPU-hour cost ÷ tokens per GPU-hour actually achieved). (general knowledge; the arithmetic is from [[2026-11-12-transformer-inference-arithmetic]])
- Input and output tokens cost differently to produce. Prompt tokens go through one parallel prefill pass; each output token needs its own memory-bound decode step ([[prefill-and-decode]]). Commercial APIs commonly price output tokens higher. (src for the phase asymmetry: [[2026-11-12-transformer-inference-arithmetic]]; pricing practice: general knowledge)
- [[inference-optimization]] (continuous batching, quantization, prefix caching) raises tokens per GPU-hour. The handbook claims a poorly optimized stack can cost 10× more in GPU hours, without evidence. (src: [[2026-11-21-what-is-llm-inference]])

### What metering has to capture

- Per-token: exact input/output (and possibly cached) token counts per request, attributed to a tenant even when requests share a batch. Counted by the serving layer, not by the GPU. (general knowledge)
- Per GPU-hour: allocation start/stop events from the lifecycle system. GPU-Util is not needed for billing, and it is misleading as an efficiency signal anyway ([[gpu-utilization-metrics]], src: [[2026-11-10-deep-learning-brrrr-first-principles]]).

## Why it matters for a GPU cloud

- A GPU cloud usually sells both: raw GPU-hours to customers who self-host, and possibly managed or serverless endpoints. Its control plane then needs two metering pipelines and two kinds of quota (GPUs vs tokens/requests per minute). (general knowledge)
- Token quotas (rate limits) protect shared replicas. GPU quotas protect capacity. They are not interchangeable, because tokens per GPU varies with batch, context length, and model. (general knowledge; variability: src [[2026-11-12-transformer-inference-arithmetic]])

## Related

- [[inference-serving]]: serverless vs self-hosted is a choice of who runs serving.
- [[inference-batching]]: sets cost per token.
- [[gpu-utilization-metrics]]: GPU-side metrics and their pitfalls.
- [[2026-11-12-sizing-a-gpu-inference-deployment]]: worked numbers behind cost per token.

## Open questions

- How do real providers price cached (prefix-hit) input tokens, long-context requests, and batch (non-interactive) jobs? Need provider pricing pages as sources.
- How is token metering made exact and tamper-resistant when requests are batched together and streams can be cancelled mid-response?
