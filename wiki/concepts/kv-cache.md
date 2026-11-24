---
type: concept
tags: [provisioning, tenancy-quotas, metering-billing]
sources: [2026-11-12-transformer-inference-arithmetic, 2026-11-21-what-is-llm-inference]
created: 2026-11-12
updated: 2026-11-24
---

# KV cache

The **kv cache** (also "past cache") is a GPU-memory buffer holding the attention **key** and **value** vectors of every token a language model has already read or generated, so the model does not recompute them when it generates the next token. Analogy: instead of re-reading the whole book each time you write the next sentence, you keep notes on every page. It trades memory for compute. (src: [[2026-11-12-transformer-inference-arithmetic]]) It is the main reason a chat request's memory grows as the conversation gets longer. (general knowledge)

## Details

### Why it is needed

- A transformer generates text one token at a time ("autoregressive": each output becomes part of the next input). Producing each token runs the whole model once, and attention needs the key/value vectors of every earlier token. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Without a cache, each step recomputes those vectors from the start, so total sampling cost grows **quadratically** in the number of tokens. (src: [[2026-11-12-transformer-inference-arithmetic]])
- The prompt is processed in parallel and fills the cache; sampling then adds one token per step. (src: [[2026-11-12-transformer-inference-arithmetic]]) These phases are commonly called **prefill** and **decode**; see [[prefill-and-decode]]. (general knowledge)

### Cost

- Stored shape, per layer: `[batch, 2, n_heads, seq_len, d_head]` (the post writes `features` for `d_head`): two vectors (k and v) per head, per token. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Bytes per token at 16-bit precision: `2 · 2 · n_layers · n_heads · d_head`, which equals `4 · n_layers · d_model` because `n_heads · d_head = d_model` in standard multi-head attention. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Many newer models use grouped-query or multi-query attention (GQA/MQA): several query heads share one key/value head, so the cache is `2 · 2 · n_layers · n_kv_heads · d_head` bytes per token, smaller than the formula above by `n_heads / n_kv_heads`. (general knowledge; verified: [GQA paper](https://arxiv.org/abs/2305.13245))
- 52B example (`d_model` 8192, 64 layers): `4 · 64 · 8192` = 2,097,152 bytes ≈ **2 MB per token**, so a 2048-token sequence holds 4,294,967,296 bytes ≈ 4.3 GB (exactly 4 GiB). (src: [[2026-11-12-transformer-inference-arithmetic]]; product: arithmetic check, this wiki)
- What it saves, per the post: computing k,v for one token is `4 · n_layers · d_model²` flops = 17.2 GFLOP on the 52B model, 1/6 of a full forward pass, so each step saves 1/6 of a pass per earlier token. (src: [[2026-11-12-transformer-inference-arithmetic]]) That count assumes each earlier token's per-layer input is still available. A model with no cache at all must rerun the *whole* forward pass over every earlier token to rebuild them (what generation libraries do with caching switched off), so the real saving per step is closer to a full pass per earlier token. (general knowledge)

### When not to use it

- If the batch is small you are memory-bandwidth-bound anyway (see [[inference-batching]]), so recomputing spends idle flops and saves memory instead of costing time. The post says to consider dropping the cache at low load. (src: [[2026-11-12-transformer-inference-arithmetic]]) This only holds while the recomputation fits in the idle compute of a memory-bound step (a budget of roughly 208 token-passes per step on the post's A100 numbers); for long contexts it is not free, and production engines keep the cache. (general knowledge)
- It also adds memory traffic: on the benchmarked 13B model, storing to the cache cost 18 µs per layer, ≈ 0.7 ms in total for the 512-token context pass on one GPU. (src: [[2026-11-12-transformer-inference-arithmetic]]) Reading the cache at every decode step is a further memory-bandwidth cost the post leaves out of its equations ([[inference-latency-model]]). (src: [[2026-11-12-transformer-inference-arithmetic]])

### Interaction with parallelism

- Under tensor parallelism the cache is split by attention head across GPUs, so each GPU stores only its heads' share. (src: [[2026-11-12-transformer-inference-arithmetic]])

### Optimizations built on the cache

- The handbook names two cache optimizations. **KV cache management**: reusing or offloading attention caches to handle long prompts efficiently. **Prefix caching**: caching common prompt segments to reduce redundant computation. Neither is explained in this source. Roadmap: [[inference-optimization]]. (src: [[2026-11-21-what-is-llm-inference]])
- Prefix caching keeps the cache blocks of already-processed prompts and reuses them when a new request starts with the same tokens (a shared system prompt, for example); it does not change model outputs. vLLM identifies reusable blocks by hashing each block's tokens together with all tokens before it. (general knowledge; verified: [vLLM: Automatic Prefix Caching](https://docs.vllm.ai/en/latest/design/prefix_caching/))
- Offloading moves cache out of GPU memory to cheaper, slower memory (CPU RAM, SSD), which relieves the capacity limit in [[gpu-memory-capacity]]. (general knowledge)
- **PagedAttention** (the core of vLLM) stores each request's cache in fixed-size blocks, like operating-system memory pages, instead of one contiguous buffer sized for the maximum length. This cuts the memory wasted to fragmentation to near zero and lets requests share blocks, so more requests fit in a batch; the paper reports 2–4× throughput over FasterTransformer and Orca at the same latency. It is a memory manager, not a batch scheduler; scheduling across requests is continuous batching ([[inference-batching]]). (general knowledge; verified: [PagedAttention paper](https://arxiv.org/abs/2309.06180))

## Why it matters for a GPU cloud

- Cache size is why LLM serving memory is a function of *context length*, not just model size, so quota systems and schedulers that count "one replica = one model" will mispredict capacity. See [[gpu-memory-capacity]]. (src: [[2026-11-12-transformer-inference-arithmetic]] for the memory arithmetic; general knowledge for the quota consequence)
- Long-context requests can be refused or throttled purely on memory grounds, which makes max context length a real product limit per SKU. (general knowledge)
- Because the cache is the per-request memory that limits batch size, serving engines manage it in blocks (PagedAttention, above); a provider's scheduler must know how much of it a tenant may hold. (general knowledge)

## Related

- [[gpu-memory-capacity]]: the cache competes with weights for the same GPU memory.
- [[inference-batching]]: cache size caps how many requests can run together.
- [[inference-latency-model]]: cache reads appear as an extra memory term.
- [[model-parallelism]]: the cache is sharded with the heads.
- [[prefill-and-decode]]: prefill writes the cache, decode reads it.
- [[inference-optimization]]: prefix caching and cache offloading.
