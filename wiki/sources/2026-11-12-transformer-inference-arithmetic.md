---
type: source
tags: [provisioning, tenancy-quotas, metering-billing]
sources: []
raw: raw/2026-11-12-transformer-inference-arithmetic.md
url: https://kipp.ly/p/transformer-inference-arithmetic
author: Carol Chen (kipply)
created: 2026-11-12
updated: 2026-11-24
---

# Transformer Inference Arithmetic

Raw: [[raw/2026-11-12-transformer-inference-arithmetic]] · Carol Chen ("kipply"), 2022-03-30.

## Summary

A worked calculation of how fast a large language model can *serve* text, done with arithmetic instead of experiments. It walks through four things you can count on paper: the [[kv-cache]] that makes sampling cheap, the GPU memory that cache and the model weights compete for ([[gpu-memory-capacity]]), the [[model-parallelism]] needed when the model does not fit on one GPU, and the resulting [[inference-latency-model]]. Every quantity is in units of [[flops]] and bytes, so the same handful of ratios decides whether a given request is compute-bound or [[memory-bandwidth]]-bound ([[gpu-performance-regimes]]). The predictions are then checked against NVIDIA FasterTransformer benchmarks on a 13B model: the naive estimates come in 24% (1 GPU) and 38% (2 GPUs) below measured latency, and measured bandwidth efficiency, intermediate ops, and comms account for nearly all of the gap.

**Why a control-plane engineer should care:** this post is the arithmetic behind three product decisions. How many concurrent requests fit on one GPU (capacity → quotas and packing). How big a batch to run (throughput vs latency → SLA and price). How many GPUs one model needs and how they must be wired (→ node shape, NVLink, multi-GPU scheduling). It also explains why "tokens per second" is the natural metering unit for inference, in contrast to the GPU-hours model used for training. (general knowledge)

![[raw/assets/tia-kvdiagram.png]]
Sampling loop with a [[kv-cache]]: the prompt (`a b c d e f g`, bottom left) is processed in parallel through attention (`attn`) and the feed-forward block (`mlp`) and fills the cache; each new token then runs through the transformer alone, appending its `k`/`v` vectors (dimension `seq_len`, so they concatenate).

## Key points

### The kv cache

- Inference has two phases: process the prompt in parallel, then sample tokens one at a time, each depending on all previous ones ("autoregressive" — like finishing a sentence one word at a time). The kv cache stores the attention key/value vectors for every token already seen so they are not recomputed per step. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Cost per cached token, at 16-bit precision: `2 · 2 · n_layers · n_heads · d_head` bytes (2 for k and v, 2 bytes per number). For a 52B model (`d_model`=8192, 64 layers) that is 2,097,152 bytes ≈ **2 MB per token**. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Recomputing k and v from scratch for one token costs `2 · 2 · n_layers · d_model²` flops = 17.2 GFLOP on the 52B model, i.e. **1/6 of a full forward pass**; without the cache, sampling cost grows quadratically with context length. (src: [[2026-11-12-transformer-inference-arithmetic]])
- The cache is not always worth it: if batch size is small, you are memory-bound anyway, so recomputing spends flops you were not using and saves memory. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Detail: [[kv-cache]].

### The one ratio that decides everything: 208 flops/byte

- An A100 does 312e12 flops/s and 1.5e12 bytes/s, so it has **208 flops per byte** of bandwidth. Below that arithmetic intensity you are memory-bound; above, compute-bound. (src: [[2026-11-12-transformer-inference-arithmetic]])
- For inference, the axis is tokens processed in one forward pass: computing for up to ~208 tokens takes the same time as for one, because reading the weights dominates; beyond ~208, time grows with tokens. At decode each sequence adds one token per pass, so the threshold becomes a batch size of ~208. (src: [[2026-11-12-transformer-inference-arithmetic]])

![[raw/assets/tia-roofline.png]]
The post's roofline: time for a forward pass versus tokens in the pass. Memory time is flat (all weights must be read once regardless of batch); compute time rises linearly; they cross at 208 tokens.

### Capacity sets batch size, batch size sets cost

- A 52B model at 16 bits is 104 GB of weights — it does not fit on one 40 GB A100, so at least 3 (sensibly 4) GPUs are needed; see [[gpu-memory-capacity]]. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Four A100-40GB give 160 GB: 56 GB left for the cache. The post says ≈ 23,000 cached tokens, but its own numbers give 56 ÷ 0.002 ≈ 28,000 (arithmetic check, this wiki). Three GPUs leave only 16 GB ≈ 8,000 tokens, i.e. batch 4 × 2048-token requests. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Because weights are a fixed cost per replica, bigger batches amortize them better; but batch size cannot grow past what memory allows. (src: [[2026-11-12-transformer-inference-arithmetic]])

### Model parallelism and its communication floor

- Weights and flops divide by the number of GPUs, but the GPUs must synchronize: 4 communications per layer, each of a `d_model`-sized vector, so `4 · (N-1) · d_model / N` bytes per layer across `N` GPUs. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Communication is a *floor* on latency: compute time shrinks as you add GPUs, comms time does not. That is why adding GPUs stops helping. (src: [[2026-11-12-transformer-inference-arithmetic]])
- A100 link bandwidth used here: 300 GB/s unidirectional (NVIDIA's 600 GB/s counts both directions). (src: [[2026-11-12-transformer-inference-arithmetic]])
- Each of the four comms steps must individually be cheaper than the compute it overlaps, so the governing ratio is flops per byte of comms: 312e12/300e9 = **1040**. Per-layer values are `3·d_model`, `d_model`, `4·d_model`, `4·d_model`, so any model with `d_model` above ~1024 per GPU stays comfortable. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Pipeline parallelism (one GPU per layer group) sends only `d_model` total, but idles all but one GPU per request; tensor parallelism uses all GPUs but talks more. Detail: [[model-parallelism]].

![[raw/assets/tia-model-parallel.png]]
Tensor parallel split for a 2-GPU, 2-head toy model: each GPU holds half the query/key/value/output weights and one attention head, then the MLP is split by columns and rows (`W_1`, `W_2` shards). The green arrows mark the two send/receive rounds per block, each passing `d_model/N`-sized vectors `N-1` times.

### The latency equations

- Small batch (memory-bound): `compute = 2 bytes · P / (N · A_bm)` seconds per token, where `P` = parameters, `N` = GPUs, `A_bm` = per-GPU bandwidth; comms adds `4 · n_layers · 8 µs` of fixed latency. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Large batch (compute-bound): `compute = B · 2·P / (N · A_f)`; comms become a throughput term `B · 2·4·n_layers·d_model / A_c`. Take the larger of the two, assuming they overlap. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Worked Gopher-size example (260B params, 16 GPUs): 22 ms per token at batch 1; 53 ms per token at batch 512 (so 62 ms for the whole batch), of which comms latency is 2.5 ms and throughput cost ~35 µs. (src: [[2026-11-12-transformer-inference-arithmetic]])
- A full forward pass is 2 flops per parameter (`2·P`), i.e. `24 · n_layers · d_model²`; for the 52B model that is 103 GFLOP, and 51.5B parameters — the 0.5B gap is the token embeddings, under 2%. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Detail: [[inference-latency-model]].

### Small ops: real, then irrelevant

- Softmax, layernorm, biases, ReLU, dropout are memory-bound and cost far more time than their flop share. In a 336M-parameter model they are still **43% of latency** even with the paper's optimised (fused) kernels; in plain PyTorch an unfused softmax (453 µs) costs more than the q/k/v projection (333 µs). (src: [[2026-11-12-transformer-inference-arithmetic]])
- Scaling saves you: per-layer memory traffic grows ~`d_model` while flops grow ~`d_model²`, so at 52B (with the same optimisations) these ops are ~5% of latency. Fix is [[operator-fusion]]. (src: [[2026-11-12-transformer-inference-arithmetic]])

![[raw/assets/tia-table-a1.png]]
Table A.1 from "Data Movement Is All You Need" (336M BERT, `d_model`=1024, 24 layers), reproduced by the post: per-operator FLOPs, memory traffic, PyTorch time, and optimised time. Scaled softmax spends 453 µs against its 0.2 GFLOP, versus 333 µs for Q/K/V at 25.8 GFLOP — the memory-bound signature.

### Checks against real hardware

- 13B model (40 layers, 40 heads, `d_head` 128, `d_model` 5120), FasterTransformer, 512-token context, one output token: predicted 16.8 ms on one GPU, 8.4 ms on two. Measured: **22.0 ms** and **13.5 ms**. (src: [[2026-11-12-transformer-inference-arithmetic]])
- The decode gap is accounted for: ~90% (1 GPU) and ~87% (2 GPUs) of peak bandwidth achieved, ~2 ms of intermediate ops, and ~1.7 ms of un-overlapped comms on the 2-GPU run. Separately, the 512-token context pass spends 0.7 ms storing kv and ~0.2 ms on memsets. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Achieved flops also fall short for small matrices: an MLP matmul expected at 344 µs took 476 µs (72% of peak); an attention projection expected at 86 µs took 159 µs (54%). (src: [[2026-11-12-transformer-inference-arithmetic]])

## The post's central tradeoff table

![[raw/assets/tia-batch-size.png]]
Three consequences of batch size, left to right: memory is mostly weights at small batch and mostly cache at large batch; the regime moves from memory-bandwidth-bound to flops-bound; communication cost moves from a latency term to a throughput term.

| Axis | Small batch | Large batch |
| --- | --- | --- |
| GPU memory | mostly weights | mostly kv cache |
| Bottleneck | memory bandwidth | flops |
| Comms | latency-dominated | throughput-dominated |
| Right choice | maybe drop the cache and recompute | keep cache, maximize tokens in flight |

(src: [[2026-11-12-transformer-inference-arithmetic]])

## Takeaways

- Any SKU's flops-per-byte threshold is peak FLOPS ÷ memory bandwidth (208 for an A100). It is the number of tokens per forward pass (at decode, the batch size) at which serving stops being memory-bound and becomes compute-bound, so it sets the useful batch size. (src: [[2026-11-12-transformer-inference-arithmetic]])
- A 52B fp16 model (104 GB) cannot run on one 40 GB GPU. Whatever memory the weights leave free becomes kv cache, which caps how many tokens can be in flight. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Adding GPUs stops lowering latency because the communication floor does not shrink, and pipeline parallelism does not speed up a single request because only one GPU works on it at a time. (src: [[2026-11-12-transformer-inference-arithmetic]])

## Concepts & entities touched

[[kv-cache]] · [[gpu-memory-capacity]] · [[model-parallelism]] · [[inference-batching]] · [[inference-latency-model]] · [[gpu-performance-regimes]] · [[flops]] · [[memory-bandwidth]] · [[operator-fusion]] · [[nvidia-a100]]

## Questions raised

- The numbers are A100-era (2022) and 16-bit. How do H100/H200/B200 (higher flops per byte, so a higher threshold; HBM3 vs HBM2e) change each threshold, and how does FP8/INT8 serving shift the memory budget?
- The model assumes one request per sequence slot. How much of real serving efficiency comes from batching *across* requests with different context lengths (continuous batching), which the post does not model? (general knowledge: this is what vLLM's PagedAttention and similar schedulers address)
- What should a control plane expose so a tenant can reason this way: per-replica memory, tokens/sec, context-length limits, queue depth?
- Where do multi-node links (InfiniBand, 400 GB/s-class interconnects) replace NVLink, and at what point does cross-node comms kill tensor parallelism?
- The post asserts 8 µs per message from a V100 NVLink paper. What are current per-message latencies, and do they dominate small-model serving?
