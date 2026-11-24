---
type: concept
tags: [provisioning, tenancy-quotas, metering-billing]
sources: [2026-11-12-transformer-inference-arithmetic, 2026-11-21-what-is-llm-inference]
created: 2026-11-12
updated: 2026-11-24
---

# Inference batching

**Batching** in model inference means serving several requests in one pass through the model, so the cost of reading the weights is shared instead of paid per request. The axis that decides memory-bound vs compute-bound is **tokens processed per pass**: during prefill, the prompt's tokens; during decode, one token per sequence, so it equals the batch size ([[prefill-and-decode]]). Batch size is the knob that trades latency against throughput against cost. (src: [[2026-11-12-transformer-inference-arithmetic]]) Batch size × context length is a different quantity: it sets how much [[kv-cache]] memory the batch holds (capacity, below), not which regime the weight matmuls are in. (src: [[2026-11-12-transformer-inference-arithmetic]])

## Details

### Why bigger batches are cheaper

- Reading the weights costs the same regardless of batch size: `2 · P` bytes per pass for `P` parameters at 16-bit. Compute costs `B · 2 · P` flops for batch `B`. So throughput per request improves with `B` until compute becomes the limit. (src: [[2026-11-12-transformer-inference-arithmetic]])
- The crossover is the hardware's flops-per-byte ratio: 312e12 flops/s ÷ 1.5e12 bytes/s = **208** for an [[nvidia-a100]] (the post's constants). Below ~208 tokens per pass (at decode: a batch below ~208 sequences) you are [[memory-bandwidth]]-bound; above, compute-bound ([[gpu-performance-regimes]]). (src: [[2026-11-12-transformer-inference-arithmetic]])
- Concretely: a 52B model's forward pass would take ~69 ms on one GPU for up to 208 tokens (hypothetical, since the weights do not fit on one 40 GB card), ~17 ms across four GPUs, and twice as long at 416 tokens. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Past the crossover, enlarging the batch does not make anything faster per token; it only increases total work per step. (src: [[2026-11-12-transformer-inference-arithmetic]])
- The 208 crossover covers the weight matmuls only. Reading each sequence's own kv cache is extra memory traffic that grows with batch × context and is not shared across the batch, so long-context decode stays partly memory-bound even past 208. The post names this cache read as an omitted term. (src: [[2026-11-12-transformer-inference-arithmetic]] for the omitted term; the consequence is general knowledge)

### What batch size changes, per the source's own summary

![[raw/assets/tia-batch-size.png]]
Left to right: small batch means memory is mostly weights, the bottleneck is bandwidth, and communication shows up as latency; large batch means memory is mostly the (past) cache, the bottleneck is flops, and communication shows up as throughput.

| Axis | Small batch | Large batch |
| --- | --- | --- |
| GPU memory | mostly weights | mostly [[kv-cache]] |
| Bottleneck | memory bandwidth | flops |
| Comms | latency term | throughput term |
| Cache worth it? (post's advice) | maybe not, recompute instead | yes |

(src: [[2026-11-12-transformer-inference-arithmetic]])

### Capacity is the upper bound

- Batch size cannot exceed what memory allows: `(GPU memory − weights) ÷ (bytes per cached token × context length)`. On 4×40 GB A100s with a 52B model, that is ≈ 26,700 tokens, about 13 requests of 2048 tokens (the source prints 23,000; see [[gpu-memory-capacity]]). (src: [[2026-11-12-transformer-inference-arithmetic]]; corrected figure: arithmetic check, this wiki)
- That is why the same model can be cheap (big batch) on large-memory GPUs and expensive (small batch) on smaller cards. (general knowledge)

### Choosing a batch size

- **Low-load API (interactive):** batches are small, so memory-bound; dropping the [[kv-cache]] and recomputing can be the right call. (src: [[2026-11-12-transformer-inference-arithmetic]])
- **Loaded API:** serve the smallest batch size that reaches flops-bound, even if memory would allow more, to keep per-request latency low. (src: [[2026-11-12-transformer-inference-arithmetic]])
- **Mass-inference job (e.g. AlphaCode-style generation):** use as many chips as available and the largest batch that fits. (src: [[2026-11-12-transformer-inference-arithmetic]])
- The post says it hedges with "may" but thinks these three cases are absolute: the load profile determines the batch strategy. (src: [[2026-11-12-transformer-inference-arithmetic]])

### Batching across requests in practice

- The post's model is one static, uniform batch. Real traffic mixes requests that arrive at different times and finish at different lengths. With static batching a finished request's slot sits idle until the whole batch ends, and new requests wait. **Continuous (iteration-level) batching**, introduced by Orca (OSDI '22), re-forms the batch at every decode step: finished requests leave and queued ones join immediately. NVIDIA's TensorRT LLM calls the same idea "in-flight batching". (general knowledge; verified: [Orca, OSDI '22](https://www.usenix.org/conference/osdi22/presentation/yu), [TensorRT LLM: Paged Attention, IFB, and Request Scheduling](https://nvidia.github.io/TensorRT-LLM/latest/features/paged-attention-ifb-scheduler.html))
- Continuous batching only helps if the memory for more concurrent requests exists. vLLM's PagedAttention provides that by allocating the [[kv-cache]] in small blocks with near-zero fragmentation. It is a memory manager that enables larger batches, not the scheduler itself. (general knowledge; verified: [PagedAttention paper](https://arxiv.org/abs/2309.06180))
- The handbook lists continuous batching ("dynamically grouping requests for better GPU utilization") first among inference optimizations, and batching across concurrent requests is a built-in feature of [[inference-frameworks]] and a core [[inference-serving]] responsibility. Roadmap: [[inference-optimization]]. (src: [[2026-11-21-what-is-llm-inference]])

## Why it matters for a GPU cloud

- **Metering and billing:** cost per token falls with batch size, so a provider selling "tokens" is really selling *batching efficiency*; GPU-hours and tokens-per-second are different products. (general knowledge; the arithmetic is from [[2026-11-12-transformer-inference-arithmetic]])
- **Tenancy and quotas:** concurrency limits are batch-size limits. A tenant allowed "one replica" gets whatever concurrency memory permits, not a fixed request count. (general knowledge)
- **SLA design:** latency and throughput are the same knob at different settings, so an SLA needs to say which one is guaranteed. (general knowledge, from the source's batch-size tradeoff)

## Open questions

- How much efficiency is lost when sequences in a batch have different lengths (padding, fragmentation), and how much do continuous batching and paged caches recover in practice?
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
