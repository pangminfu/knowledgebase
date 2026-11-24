---
type: concept
tags: [provisioning, tenancy-quotas, metering-billing]
sources: [2026-11-21-what-is-llm-inference, 2026-11-12-transformer-inference-arithmetic, 2026-11-10-deep-learning-brrrr-first-principles]
created: 2026-11-21
updated: 2026-11-24
---

# Inference optimization

**Inference optimization** is the set of techniques that make [[llm-inference]] faster (lower latency), higher-volume (more throughput), and cheaper (less hardware) without hurting model quality. It decides whether an application "feels sluggish and expensive" or "snappy, cost-efficient." This page is a hub and reading roadmap: each technique gets its own page once a source explains it. (src: [[2026-11-21-what-is-llm-inference]])

## Details

### Map: which bottleneck each technique attacks

Every technique below attacks one of the limits already covered in this wiki: memory capacity (how many tokens fit), memory bandwidth (how fast weights are read per decode step), compute, or wasted repeat work. The "How it works / attacks" column is general knowledge unless a link is given; the handbook only names each technique. (technique list and one-liners: src [[2026-11-21-what-is-llm-inference]])

| Technique | Handbook's one-liner | How it works / what it attacks | Wiki status | Control-plane angle |
| --- | --- | --- | --- | --- |
| Continuous batching | Dynamically grouping requests for better GPU utilization | The scheduler rebuilds the batch at every decode step: finished requests leave and waiting ones join immediately, instead of the whole batch waiting for its longest request ([[inference-batching]]). Verified: [Orca, OSDI 2022](https://www.usenix.org/conference/osdi22/presentation/yu) ("iteration-level scheduling") | Not yet sourced | Higher packing → more tokens/sec per GPU → lower cost per token |
| KV cache management | Reusing or offloading attention caches to handle long prompts efficiently | Memory capacity ([[kv-cache]], [[gpu-memory-capacity]]): pack the cache tightly (e.g. vLLM's paged layout) and move cold entries off the GPU | Not yet sourced | Raises concurrency per replica; offload uses the node's CPU RAM or SSD |
| Speculative decoding | Using a smaller draft model to speed up token generation | A cheap drafter guesses several next tokens; the big model checks them all in one pass and keeps the ones it agrees with, with no change to the output distribution. Uses the spare compute of memory-bound decode steps ([[prefill-and-decode]], [[gpu-performance-regimes]]). Verified: [Leviathan et al., 2023](https://arxiv.org/abs/2211.17192). The drafter need not be a separate model: n-gram and EAGLE methods also exist ([vLLM README](https://github.com/vllm-project/vllm)) | Not yet sourced | A draft model adds memory per replica; gains depend on how often guesses are accepted |
| Quantization | Running models in lower precision (e.g. INT8, FP8) to save memory and compute | Memory capacity *and* bandwidth: 8-bit weights are half the bytes of 16-bit, so roughly half the weight-read time per decode step ([[memory-bandwidth]]). Lower precision can cost some accuracy, so it is checked per model, despite the handbook's "without hurting model quality" framing | Not yet sourced | Fewer GPUs per replica; the arithmetic in [[2026-11-12-sizing-a-gpu-inference-deployment]] assumes 16-bit |
| Prefix caching | Caching common prompt segments to reduce redundant computation | Repeated prefill work: requests that share a prefix (system prompt, document, chat history) reuse its stored KV cache instead of recomputing it | Not yet sourced | Needs cache-aware routing (same prefix → same replica); providers bill cached input tokens at a discount ([[per-token-pricing]]) |
| Multi-GPU parallelism | Splitting LLMs across multiple GPUs for larger context windows | Memory capacity and latency. The handbook's "larger context windows" is one benefit (more total memory leaves room for a bigger KV cache); the main reasons are fitting a model too big for one GPU and cutting per-token latency | Covered: [[model-parallelism]] | Gang scheduling, NVLink topology |
| Routing | (mentioned only in the "why care" section) | Sending each request to the replica best placed to serve it (load, cached prefix) | Not yet sourced | Load balancing in [[inference-serving]] |
| Operator fusion | (not in this source) | Memory round-trips between small ops | Covered: [[operator-fusion]] (src: [[2026-11-10-deep-learning-brrrr-first-principles]]) | Mostly framework-internal |

### Reading roadmap

Suggested order: each step builds on pages that already exist.

1. **Static, dynamic, continuous batching**: extends [[inference-batching]]. <https://handbook.modular.com/inference-optimization/static-dynamic-continuous-batching/>
2. **Prefix caching**, then **KV cache offloading**: extend [[kv-cache]]. <https://handbook.modular.com/inference-optimization/prefix-caching/> · <https://handbook.modular.com/inference-optimization/kv-cache-offloading/>
3. **Quantization**: revisits the 2-bytes-per-parameter assumption in [[gpu-memory-capacity]]. <https://handbook.modular.com/model-preparation/llm-quantization/>
4. **Parallelism (data/tensor/pipeline/expert/hybrid)**: extends [[model-parallelism]]. <https://handbook.modular.com/inference-optimization/data-tensor-pipeline-expert-hybrid-parallelism/>
5. **Inference routing**: the load-balancing row of [[inference-serving]]. <https://handbook.modular.com/inference-optimization/inference-routing/>
6. **Speculative decoding**. <https://handbook.modular.com/inference-optimization/speculative-decoding/>
7. Business context: **serverless vs self-hosted** (<https://handbook.modular.com/getting-started/serverless-vs-self-hosted-llm-inference/>) and **choosing a framework** (<https://handbook.modular.com/getting-started/choosing-the-right-inference-framework/>).

(URLs from the source's links: [[2026-11-21-what-is-llm-inference]])

## Why it matters for a GPU cloud

- The handbook claims a poorly optimized setup "can cost 10× more in GPU hours while delivering worse performance." The source gives no workload or baseline. (src: [[2026-11-21-what-is-llm-inference]]) The order of magnitude is plausible: in the idealized A100 model of [[2026-11-12-transformer-inference-arithmetic]], a decode pass takes the same time for 1 to ~208 sequences (ignoring KV-cache reads), so serving one request at a time instead of ~208 uses up to ~208× more GPU time per token. (derived from src: [[2026-11-12-transformer-inference-arithmetic]])
- Optimizations change the capacity a quota or price was set against: the same GPU serves more tokens. Quotas set in GPUs and prices set in tokens drift apart as the serving stack improves. (general knowledge)

## Related

- [[inference-frameworks]]: where these techniques are implemented.
- [[gpu-performance-regimes]]: the diagnosis step before choosing a technique.
- [[inference-serving]]: optimization serves the serving goals of latency, throughput, and cost.
