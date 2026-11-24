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

Every technique below attacks one of the limits already covered in this wiki: memory capacity (how many tokens fit), memory bandwidth (how fast weights are read per decode step), compute, or wasted repeat work. The "attacks" column is this wiki's analysis and should be checked against each technique's chapter. (technique list and one-line descriptions: src [[2026-11-21-what-is-llm-inference]]; analysis: general knowledge unless cited)

| Technique | Handbook's one-liner | Attacks | Wiki status | Control-plane angle |
| --- | --- | --- | --- | --- |
| Continuous batching | Dynamically grouping requests for better GPU utilization | Idle slots when requests of different lengths finish at different times ([[inference-batching]]) | Not yet sourced | Higher packing → more tokens/sec per GPU → price per token |
| KV cache management | Reusing or offloading attention caches to handle long prompts | Memory capacity ([[kv-cache]], [[gpu-memory-capacity]]) | Not yet sourced | Raises concurrency per replica; cache offload uses CPU RAM/SSD on the node |
| Speculative decoding | A smaller draft model speeds up token generation | Memory-bound decode steps ([[prefill-and-decode]], [[gpu-performance-regimes]]) | Not yet sourced | Extra model per replica; changes memory sizing |
| Quantization | Lower precision (e.g. INT8, FP8) saves memory and compute | Memory capacity *and* bandwidth: half the bytes per weight means half the read time per decode step ([[memory-bandwidth]]) | Not yet sourced | Fewer GPUs per replica; the arithmetic in [[2026-11-12-sizing-a-gpu-inference-deployment]] assumes 16-bit |
| Prefix caching | Caching common prompt segments to avoid redundant computation | Repeated prefill work | Not yet sourced | Requires cache-aware routing; cached tokens may be billed differently ([[per-token-pricing]]) |
| Multi-GPU parallelism | Splitting LLMs across GPUs for larger context windows | Memory capacity; latency | Covered: [[model-parallelism]] | Gang scheduling, NVLink topology |
| Routing | (mentioned in the "why care" section) | Sending each request to the replica best placed to serve it | Not yet sourced | Load balancing in [[inference-serving]] |
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

- The handbook claims a poorly optimized setup "can cost 10× more in GPU hours while delivering worse performance." The number is unsupported in the source. (src: [[2026-11-21-what-is-llm-inference]]) The direction is consistent with the 208-token batching crossover in [[2026-11-12-transformer-inference-arithmetic]]. (general knowledge)
- Optimizations change the capacity a quota or price was set against: the same GPU serves more tokens. Quotas set in GPUs and prices set in tokens drift apart as the serving stack improves. (general knowledge)

## Related

- [[inference-frameworks]]: where these techniques are implemented.
- [[gpu-performance-regimes]]: the diagnosis step before choosing a technique.
- [[inference-serving]]: optimization serves the serving goals of latency, throughput, and cost.
