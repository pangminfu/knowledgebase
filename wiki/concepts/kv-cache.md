---
type: concept
tags: [provisioning, tenancy-quotas, metering-billing, career]
sources: [2026-11-12-transformer-inference-arithmetic, 2026-11-21-what-is-llm-inference]
created: 2026-11-12
updated: 2026-11-21
---

# KV cache

The **kv cache** (also "past cache" or "context cache") is a GPU-memory buffer holding the attention **key** and **value** vectors of every token a language model has already read or generated, so the model does not recompute them when it generates the next token. Analogy: instead of re-reading the whole book each time you write the next sentence, you keep notes on every page. It trades memory for compute, and it is the single biggest reason a chat request's memory grows as the conversation gets longer. (src: [[2026-11-12-transformer-inference-arithmetic]])

## Details

### Why it is needed

- A transformer generates text one token at a time ("autoregressive": each output becomes part of the next input). Producing each token runs the whole model once, and attention needs the key/value vectors of every earlier token. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Without a cache, each step recomputes those vectors from the start, so total sampling cost grows **quadratically** in the number of tokens. (src: [[2026-11-12-transformer-inference-arithmetic]])
- The prompt phase ("prefill") can process all prompt tokens in parallel and fills the cache; the sampling phase ("decode") adds one token per step. (src: [[2026-11-12-transformer-inference-arithmetic]]) See [[prefill-and-decode]].

### Cost

- Stored shape is `[batch, 2, n_heads, seq_len, d_head]`: two vectors (k and v) per layer, per head, per token. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Bytes per token at 16-bit precision: `2 · 2 · n_layers · n_heads · d_head`, which equals `4 · n_layers · d_model` because `n_heads · d_head = d_model`. (src: [[2026-11-12-transformer-inference-arithmetic]])
- 52B example (`d_model` 8192, 64 layers): `4 · 64 · 8192` = 2,097,152 bytes ≈ **2 MB per token**, so a 2048-token conversation costs ~4 GB. (src: [[2026-11-12-transformer-inference-arithmetic]])
- What it saves: recomputing k,v for one token is `4 · n_layers · d_model²` flops = 17.2 GFLOP on the 52B model, 1/6 of a full forward pass. (src: [[2026-11-12-transformer-inference-arithmetic]])

### When not to use it

- If the batch is small you are memory-bandwidth-bound anyway (see [[inference-batching]]), so recomputing spends idle flops and saves memory instead of costing time. The post says to consider dropping the cache at low load. (src: [[2026-11-12-transformer-inference-arithmetic]])
- It also adds memory traffic: on the benchmarked 13B model, storing to the cache was 18 µs per layer, 0.7 ms per token. (src: [[2026-11-12-transformer-inference-arithmetic]])

### Interaction with parallelism

- Under tensor parallelism the cache is split by attention head across GPUs, so each GPU stores only its heads' share. (src: [[2026-11-12-transformer-inference-arithmetic]])

### Optimizations built on the cache

- The handbook names two cache optimizations. **KV cache management**: reusing or offloading attention caches to handle long prompts efficiently. **Prefix caching**: caching common prompt segments to avoid redundant computation. Neither is explained in this source. Roadmap: [[inference-optimization]]. (src: [[2026-11-21-what-is-llm-inference]]) A typical shared prefix is a system prompt reused across requests. (general knowledge)
- Offloading moves cache out of GPU memory to cheaper, slower memory (CPU RAM, SSD), which relieves the capacity limit in [[gpu-memory-capacity]]. (general knowledge)

## Why it matters for a GPU cloud

- Cache size is why LLM serving memory is a function of *context length*, not just model size, so quota systems and schedulers that count "one replica = one model" will mispredict capacity. See [[gpu-memory-capacity]]. (src + general knowledge)
- Long-context requests can be refused or throttled purely on memory grounds, which makes max context length a real product limit per SKU. (general knowledge)
- Serving engines that page or share cache memory between requests (vLLM's PagedAttention and similar) exist precisely because this buffer fragments; a provider's scheduler must know how much of it a tenant may hold. (general knowledge — not in the source)

## Related

- [[gpu-memory-capacity]]: the cache competes with weights for the same DRAM.
- [[inference-batching]]: cache size caps how many requests can run together.
- [[inference-latency-model]]: cache reads appear as an extra memory term.
- [[model-parallelism]]: the cache is sharded with the heads.
- [[prefill-and-decode]]: prefill writes the cache, decode reads it.
- [[inference-optimization]]: prefix caching and cache offloading.
