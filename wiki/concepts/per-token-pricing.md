---
type: concept
tags: [metering-billing, tenancy-quotas]
sources: [2026-11-21-what-is-llm-inference, 2026-11-12-transformer-inference-arithmetic, 2026-11-10-deep-learning-brrrr-first-principles, 2026-11-22-training-vs-inference]
created: 2026-11-21
updated: 2026-11-24
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

- The GPU cost of a token depends on how many tokens share each pass over the weights. In the source's idealized A100 40 GB model (16-bit weights, ignoring KV-cache reads), a pass takes the same time for 1 to ~208 tokens (at decode: 1 to ~208 sequences in the batch): below that the GPU is memory-bound and extra tokens are nearly free; above it each extra token adds compute time, so cost per token stops falling. (src: [[2026-11-12-transformer-inference-arithmetic]]) See [[inference-batching]].
- So a per-token provider is really selling **batching efficiency**: margin per token = price per token − (GPU-hour cost ÷ tokens per GPU-hour actually achieved). (general knowledge; the arithmetic is from [[2026-11-12-transformer-inference-arithmetic]])
- Input and output tokens cost differently to produce. Prompt tokens go through one parallel prefill pass; each output token needs its own decode step, which is memory-bound at small batch ([[prefill-and-decode]]). (src: [[2026-11-12-transformer-inference-arithmetic]]) Providers do not publish their cost basis, so this is a plausible reason for, not a stated cause of, output tokens costing more (next section). (general knowledge)
- [[inference-optimization]] (continuous batching, quantization, prefix caching) raises tokens per GPU-hour. The handbook claims a poorly optimized stack can cost 10× more in GPU hours, without giving a workload or baseline. (src: [[2026-11-21-what-is-llm-inference]])

### What providers actually charge (list prices, checked 2026-11-24)

| Price lever | Anthropic (Claude API) | OpenAI (API) |
| --- | --- | --- |
| Output vs input | Output = 5× input on every listed model (e.g. Claude Sonnet 4.6: $3 in / $15 out per million tokens) | Output = 4–8× input depending on model (e.g. gpt-4.1: $2 / $8; gpt-5: $1.25 / $10) |
| Cached input (prefix-cache hit) | Cache read 0.1× base input on most models (0.05× or 0.025× on some newer ones); cache *write* costs 1.25× (5-minute cache) or 2× (1-hour cache) | "Cached input" discounted, e.g. 0.1× on gpt-5, 0.5× on gpt-4o |
| Batch (asynchronous) jobs | 50% off input and output | 50% off (e.g. gpt-6.2-sol $1 / $5 vs $2 / $10) |
| Long context | Claude 4.6+ models (except Haiku 5.5): full 1M-token window at standard price; Haiku 5.5 charges more for prompts over 100k tokens | Several models charge higher "long context" rates (e.g. gpt-5.5 above 272K tokens: $10 / $45 vs $5 / $30) |

(general knowledge; verified: [Anthropic pricing](https://platform.claude.com/docs/en/about-claude/pricing), [OpenAI pricing](https://developers.openai.com/api/docs/pricing)) Prices change often; treat the numbers as a snapshot and the structure (output premium, cache discount, batch discount, long-context tier) as the durable part.

### What metering has to capture

- Per-token: exact input/output (and possibly cached) token counts per request, attributed to a tenant even when requests share a batch. Counted by the serving layer, not by the GPU. (general knowledge)
- Per GPU-hour: allocation start/stop events from the lifecycle system. GPU utilization is not needed for billing. It is also a weak efficiency signal: NVIDIA's "GPU utilization" is the percent of time at least one kernel was running, not how much of the GPU's compute was used ([[gpu-utilization-metrics]]; src: [[2026-11-10-deep-learning-brrrr-first-principles]]; definition verified: [NVML nvmlUtilization_t](https://docs.nvidia.com/deploy/nvml-api/latest/api/structnvmlUtilization__t.html)).

## Why it matters for a GPU cloud

- A GPU cloud usually sells both: raw GPU-hours to customers who self-host, and possibly managed or serverless endpoints. Its control plane then needs two metering pipelines and two kinds of quota (GPUs vs tokens/requests per minute). (general knowledge)
- Token quotas (rate limits) protect shared replicas. GPU quotas protect capacity. They are not interchangeable, because tokens per GPU varies with batch, context length, and model. (general knowledge; variability: src [[2026-11-12-transformer-inference-arithmetic]])
- Inference spend is ongoing and grows with traffic; over time it can exceed the mostly one-time training cost, especially with heavy usage or long prompts. (src: [[2026-11-22-training-vs-inference]]) Training capacity, by contrast, is usually sold as reserved GPU-hours, so a GPU cloud serving both phases also runs both meters ([[training-vs-inference]]). (general knowledge)

## Related

- [[inference-serving]]: serverless vs self-hosted is a choice of who runs serving.
- [[inference-batching]]: sets cost per token.
- [[gpu-utilization-metrics]]: GPU-side metrics and their pitfalls.
- [[2026-11-12-sizing-a-gpu-inference-deployment]]: worked numbers behind cost per token.
- [[training-vs-inference]]: why inference is a recurring cost while training is mostly one-time.

## Open questions

- How do providers set the size of the output premium and the cache discount: from measured GPU cost, or from market positioning? Their cost basis is not public.
- How is token metering made exact and tamper-resistant when requests are batched together and streams can be cancelled mid-response?
