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

A worked calculation of how fast a large language model can *serve* text, done with arithmetic instead of experiments. It walks through four things you can count on paper: the [[kv-cache]] that makes sampling cheap, the GPU memory that cache and the model weights compete for ([[gpu-memory-capacity]]), the [[model-parallelism]] needed when the model does not fit on one GPU, and the resulting [[inference-latency-model]]. Every quantity is in units of [[flops]] and bytes, so the same handful of ratios decides whether a given request is compute-bound or [[memory-bandwidth]]-bound ([[gpu-performance-regimes]]). The predictions are then checked against NVIDIA FasterTransformer benchmarks on a 13B model: the naive per-token estimates come in 24% (1 GPU) and 38% (2 GPUs) below measured latency, and measured bandwidth efficiency, intermediate ops, and comms account for nearly all of the gap.

**Why a control-plane engineer should care:** this post is the arithmetic behind three product decisions. How many concurrent requests fit on one GPU (capacity → quotas and packing). How big a batch to run (throughput vs latency → SLA and price). How many GPUs one model needs and how they must be wired (→ node shape, NVLink, multi-GPU scheduling). It also explains why "tokens per second" is the natural metering unit for inference, in contrast to the GPU-hours model used for training. (general knowledge)

![[raw/assets/tia-kvdiagram.png]]
Sampling loop with a [[kv-cache]]: the context tokens `a b c d e` go through attention (`attn`) and the feed-forward block (`mlp`) all in parallel (the "forwards pass") and fill the cache; the sampled tokens `f` (first sample) and `g` (second sample) each run through the transformer alone, reading the cache and appending their own `k`/`v` vectors, which concatenate because they have a `seq_len` dimension.

## Key points

### The kv cache

- Inference has two phases: process the prompt in parallel, then sample tokens one at a time, each depending on all previous ones ("autoregressive", like finishing a sentence one word at a time). The kv cache stores the attention key/value vectors for every token already seen so they are not recomputed per step. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Cost per cached token, at 16-bit precision: `2 · 2 · n_layers · n_heads · d_head` bytes (2 for k and v, 2 bytes per number). For a 52B model (`d_model` 8192, 64 layers; Anthropic's model from Askell et al. 2021) that is 2,097,152 bytes ≈ **2 MB per token**. (src: [[2026-11-12-transformer-inference-arithmetic]]) The dimensions match the paper's `d_model = 128 · n_layer` rule for its 52B model. (general knowledge; verified: [A General Language Assistant as a Laboratory for Alignment](https://arxiv.org/abs/2112.00861))
- Computing k and v for one token costs `2 · 2 · n_layers · d_model²` flops = 17.2 GFLOP on the 52B model, i.e. **1/6 of a full forward pass**; without the cache, sampling cost grows quadratically with sequence length. (src: [[2026-11-12-transformer-inference-arithmetic]]) The 1/6 counts only the k/v projections; see [[kv-cache#Cost]] for why the real saving is larger.
- The cache is not always worth it: at small batch you are memory-bound anyway, so the post suggests recomputing instead, spending flops you were not using. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Detail: [[kv-cache]].

### The one ratio that decides everything: 208 flops/byte

- The post's A100 constants are 312e12 flops/s and 1.5e12 bytes/s, so it has **208 flops per byte** of bandwidth. Below that arithmetic intensity you are memory-bound; above, compute-bound. (src: [[2026-11-12-transformer-inference-arithmetic]]) NVIDIA's datasheet gives 312 TFLOPS dense FP16/BF16 Tensor Core and 1,555 GB/s for the 40 GB card (the author later notes he rounded 1.555 to 1.5). (general knowledge; verified: [NVIDIA A100 datasheet](https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/a100/pdf/nvidia-a100-datasheet-us-nvidia-1758950-r4-web.pdf))
- For inference, the axis is tokens processed in one forward pass: computing for up to ~208 tokens takes the same time as for one, because reading the weights dominates; beyond ~208, time grows with tokens. At decode each sequence adds one token per pass, so the threshold becomes a batch size of ~208. (src: [[2026-11-12-transformer-inference-arithmetic]])

![[raw/assets/tia-roofline.png]]
The post's roofline: time versus number of context tokens (batch size) in one pass. Memory time is flat (all weights must be read once regardless of batch); compute time rises linearly; they cross at 208 tokens.

### Capacity sets batch size, batch size sets cost

- A 52B model at 16 bits is 104 GB of weights. It does not fit on one 40 GB A100, so at least 3 (sensibly 4) GPUs are needed; see [[gpu-memory-capacity]]. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Three A100-40GB leave 120 − 104 = 16 GB for the cache, which the post rounds to ≈ 8,000 tokens (batch 4 × 2048-token requests). Four leave 56 GB; the post prints "≈ 23,000" tokens, which does not follow from its own numbers (see Contradictions). (src: [[2026-11-12-transformer-inference-arithmetic]])
- Because weights are a fixed cost per replica, bigger batches amortize them better; but batch size cannot grow past what memory allows. (src: [[2026-11-12-transformer-inference-arithmetic]])

### Model parallelism and its communication floor

- Weights and flops divide by the number of GPUs, but the GPUs must synchronize: 4 communications per layer, each a send/receive of `d_model/N`-sized pieces `N−1` times, so `4 · (N−1) · d_model / N` values per layer (×2 bytes at 16-bit; the post writes "bytes" but its large-batch formula includes the factor 2). (src: [[2026-11-12-transformer-inference-arithmetic]])
- Communication is a *floor* on latency: compute time shrinks as you add GPUs, comms time does not. That is why adding GPUs eventually stops helping. (src: [[2026-11-12-transformer-inference-arithmetic]])
- A100 link bandwidth used here: 300 GB/s per direction (NVIDIA's 600 GB/s counts both directions). (src: [[2026-11-12-transformer-inference-arithmetic]]) The datasheet lists NVLink at 600 GB/s for SXM. (general knowledge; verified: [NVIDIA A100 datasheet](https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/a100/pdf/nvidia-a100-datasheet-us-nvidia-1758950-r4-web.pdf))
- Each of the four comms steps must individually be cheaper than the compute it overlaps, so the governing ratio is flops per byte of comms: 312e12/300e9 = **1040**. Per-layer values are `3·d_model`, `d_model`, `4·d_model`, `4·d_model`, so the post calls any model with an embedding dimension over 1024 "(per chip)" safe, and 512 "a little awkward". (src: [[2026-11-12-transformer-inference-arithmetic]])
- Pipeline parallelism (each GPU holds a fraction of the layers) communicates less (the post: `d_model` per accelerator versus `N · d_model` per layer for tensor parallelism), but for a single request all but one GPU idle. Detail: [[model-parallelism]]. (src: [[2026-11-12-transformer-inference-arithmetic]])

![[raw/assets/tia-model-parallel.png]]
Tensor-parallel split for a toy model (`d_model` 4, 2 heads, `d_head` 2, 2 accelerators): each GPU holds one attention head's slice of `W_q`, `W_k`, `W_v`, `W_o`, and a shard of the MLP matrices `W_1`, `W_2`. The green arrows mark the two send/receive rounds after the attention output and after the MLP, each passing `d_model/N`-sized vectors `N−1` times, followed by a concat before the next layer.

### The latency equations

- Small batch (memory-bound): `compute = 2 bytes · P / (N · A_bm)` seconds per token, where `P` = parameters, `N` = GPUs, `A_bm` = per-GPU bandwidth; comms adds `4 · n_layers · 8 µs` of fixed latency. (src: [[2026-11-12-transformer-inference-arithmetic]]) The 8 µs is the V100 NVLink peer-to-peer latency (8.55 µs) measured by Citadel's Volta microbenchmark report. (general knowledge; verified: [Dissecting the NVIDIA Volta GPU Architecture via Microbenchmarking, Table 5.1](https://arxiv.org/abs/1804.06826))
- Large batch (compute-bound): `compute = B · 2·P / (N · A_f)`; comms become a throughput term `B · 2·4·n_layers·d_model / A_c` bytes over link bandwidth. Take the larger of the two, assuming they overlap. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Worked "Gopher sized" example (260B parameters, 80 layers, `d_model` 16384, 16 GPUs): (src: [[2026-11-12-transformer-inference-arithmetic]])
  - Batch 1: 22 ms per decode step (2·260e9 / (16·1.5e12) = 21.7 ms). Comms latency `4 · 80 · 8 µs` = 2.56 ms (the post rounds to ≈ 3 ms); comms throughput at batch 1 ≈ 35 µs, small enough to drop.
  - Batch 512: compute 53 ms and comms throughput ≈ 18 ms per decode step, so ≈ 53 ms for a step that produces 512 tokens (one per sequence). The post also writes "in the 62ms 512 tokens are generated", which does not match its own 53 ms (see Contradictions).
  - Gopher itself is 280B parameters; the post's 260B matches `12 · 80 · 16384²` ≈ 258B. (general knowledge; verified: [Gopher paper](https://arxiv.org/abs/2112.11446))
- A forward pass costs `2·P` flops per token (each parameter is used in one multiply-add), i.e. `24 · n_layers · d_model²`; for the 52B model that is 103 GFLOP, implying 51.5B parameters. The post blames the 0.5B gap on token (un)embeddings and calls the difference under 2%. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Detail: [[inference-latency-model]].

### Small ops: real, then irrelevant

- Softmax, layernorm, biases, ReLU, dropout, and the `q·kᵀ` product are memory-bound and cost far more time than their flop share. In a 336M-parameter BERT layer they are still **43% of latency** even with the paper's optimised (fused) kernels; in plain PyTorch an unfused softmax (453 µs) costs more than the q/k/v projection (333 µs). (src: [[2026-11-12-transformer-inference-arithmetic]]) The 43% re-sums from Table A.1: memory-bound "Ours" times 1,028 µs of 2,385 µs. (arithmetic check, this wiki)
- Scaling helps: per-layer memory traffic grows ~`d_model` while flops grow ~`d_model²`, so at 52B (`d_model` 8× larger) the post estimates `0.43 · 8 / 64` ≈ 5% of latency. Rescaling only the non-memory-bound 57% by 64 gives `3.44 / (3.44 + 36.5)` ≈ 9%; either way small. (src: [[2026-11-12-transformer-inference-arithmetic]]; recomputation: arithmetic check, this wiki) The fix is [[operator-fusion]].

![[raw/assets/tia-table-a1.png]]
Table A.1 from "Data Movement Is All You Need" (one BERT encoder layer; the post gives the model as 336M parameters, `d_model` 1024, 24 layers): per-operator Gflop, input/output size, PyTorch time and % of peak, and the paper's optimised ("Ours") time, % of peak, and memory-use efficiency (MUE). Scaled softmax takes 453 µs in PyTorch for ~1 Gflop, versus 333 µs for Q/K/V at 25.8 Gflop — the memory-bound signature.

### Checks against real hardware

- 13B model (40 layers, 40 heads, `d_head` 128, `d_model` 5120), FasterTransformer, 512-token context, batch 1, 10 output tokens. Predicted per decode step: 16.8 ms on one GPU, 8.4 ms (plus ~1 ms comms) on two. Measured: **22.0 ms** and **13.5 ms**. (src: [[2026-11-12-transformer-inference-arithmetic]])
- The decode gap is accounted for: ~90% (1 GPU) and ~87% (2 GPUs) of peak bandwidth achieved, ~2.2 / 2 ms of intermediate ops, and on 2 GPUs ~1.7 ms of comms (40–50 µs per layer) not overlapped with compute. (src: [[2026-11-12-transformer-inference-arithmetic]])
- The context (prefill) pass is predicted at `decode step × 512/231` (231 = 312e12 / (1.5e12 · 0.9)): ≈ 48 ms on 1 GPU and ≈ 30 ms on 2, versus measured 63.17 ms and 45.45 ms. On 1 GPU, storing kv costs 18 µs per layer (≈ 0.7 ms) and memsets ~0.2 ms. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Achieved flops fall short for these matrix sizes: an MLP matmul expected at 344 µs took 476 µs (72% of peak); an attention projection expected at 86 µs took 159 µs (54%). (src: [[2026-11-12-transformer-inference-arithmetic]])

## The post's central tradeoff table

![[raw/assets/tia-batch-size.png]]
Three consequences of batch size, left to right: memory ("capacity") is mostly weights at small batch and mostly past (kv) cache at large batch; compute moves from memory-bandwidth-bound to flops-bound; communication cost moves from a latency term to a throughput term.

| Axis | Small batch | Large batch |
| --- | --- | --- |
| GPU memory | mostly weights | mostly kv cache |
| Bottleneck | memory bandwidth | flops |
| Comms | latency-dominated | throughput-dominated |
| Cache strategy (post's advice) | maybe drop the cache and recompute | keep cache, maximize tokens in flight |

(src: [[2026-11-12-transformer-inference-arithmetic]])

## Takeaways

- Any SKU's flops-per-byte threshold is peak FLOPS ÷ memory bandwidth (208 for the post's A100 numbers). It is the number of tokens per forward pass (at decode, the batch size) at which serving stops being memory-bound and becomes compute-bound, so it sets the useful batch size. (src: [[2026-11-12-transformer-inference-arithmetic]])
- A 52B fp16 model (104 GB) cannot run on one 40 GB GPU. Whatever memory the weights leave free becomes kv cache, which caps how many tokens can be in flight. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Adding GPUs stops lowering latency because the communication floor does not shrink, and pipeline parallelism does not speed up a single request because only one GPU works on it at a time. (src: [[2026-11-12-transformer-inference-arithmetic]])

## Concepts & entities touched

[[kv-cache]] · [[gpu-memory-capacity]] · [[model-parallelism]] · [[inference-batching]] · [[inference-latency-model]] · [[gpu-performance-regimes]] · [[flops]] · [[memory-bandwidth]] · [[operator-fusion]] · [[nvidia-a100]]

## Contradictions

- **Cache tokens on 4 GPUs.** The post (raw line 96, unchanged in the live post) says `56/0.002 ≈ 23000`. Its own inputs give 56 ÷ 0.002 = 28,000, and exactly 56e9 ÷ 2,097,152 ≈ 26,700 tokens (≈ 13 requests of 2048 tokens). The same exact division gives ≈ 7,600 (not 8,000) tokens on 3 GPUs, slightly under 4 × 2048 = 8,192. (arithmetic check, this wiki; live text: [Transformer Inference Arithmetic](https://kipp.ly/p/transformer-inference-arithmetic))
- **Batch-512 step time.** The post gives 53 ms compute and ≈ 18 ms comms for Gopher-260B at batch 512, then writes "in the 62ms 512 tokens are generated" (also in the live post). With overlap the step is max(53, 18) ≈ 53 ms; 62 ms is not derivable from the stated terms. (arithmetic check, this wiki)
- **"Still 11% latency."** The post says comms for "our 512 batch on 52B" were still 11% latency, but it never computes a 52B batch-512 case. For its 260B batch-512 example, latency is 2.56 ms of 2.56 + 17.9 ms total comms ≈ 12.5%. (arithmetic check, this wiki)
- **51.5B vs 52B.** The post attributes the gap to token (un)embeddings, but Askell et al. describe the 52B as counting *non-embedding* parameters, with a 2^16-token vocabulary (≈ 0.54B per embedding matrix at `d_model` 8192). The gap is rounding in the model's name; the "under 2%" conclusion stands. (general knowledge; verified: [Askell et al. 2021](https://arxiv.org/abs/2112.00861))

## Questions raised

- The numbers are A100-era (2022) and 16-bit. Flops-per-byte has not risen monotonically: H100 SXM is ≈ 989 dense BF16 TFLOPS ÷ 3.35 TB/s ≈ 295, while H200 has the same compute and 4.8 TB/s, ≈ 206 — about the A100's 208. (general knowledge; verified: [NVIDIA H100](https://www.nvidia.com/en-us/data-center/h100/), [NVIDIA H200](https://www.nvidia.com/en-us/data-center/h200/); NVIDIA lists the sparse figures 1,979 TFLOPS, halved here for dense.) How does FP8/INT8 serving shift the memory budget and these thresholds?
- The model assumes one uniform batch. How much of real serving efficiency comes from scheduling *across* requests with different lengths? Iteration-level ("continuous") batching was introduced by Orca, and vLLM's PagedAttention manages cache memory in blocks so more requests fit; the post models neither. (general knowledge; verified: [Orca, OSDI '22](https://www.usenix.org/conference/osdi22/presentation/yu), [PagedAttention](https://arxiv.org/abs/2309.06180))
- What should a control plane expose so a tenant can reason this way: per-replica memory, tokens/sec, context-length limits, queue depth?
- Where do multi-node links replace NVLink, and at what point does cross-node comms kill tensor parallelism? The post's exercise cites AWS's 400 Gb/s (gigabits, = 50 GB/s) instance networking against 600 GB/s NVSwitch inside a p4d node. (general knowledge; verified: [Amazon EC2 P4 instances](https://aws.amazon.com/ec2/instance-types/p4/))
- The 8 µs per message is a 2018 V100 measurement. What are current per-message latencies, and do they dominate small-model serving?
