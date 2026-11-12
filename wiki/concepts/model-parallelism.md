---
type: concept
tags: [provisioning, lifecycle, career]
sources: [2026-11-12-transformer-inference-arithmetic]
created: 2026-11-12
updated: 2026-11-12
---

# Model parallelism

**Model parallelism** means splitting one model across several GPUs, as opposed to data parallelism (which replicates the whole model and splits the *requests*). You need it when the weights do not fit on one GPU — a 52B-parameter model at 16 bits is 104 GB against 40 GB per card ([[gpu-memory-capacity]]). Two shapes matter: **tensor parallelism**, which splits every layer's matrices across GPUs and synchronizes after each block, and **pipeline parallelism**, which gives each GPU a contiguous set of layers and passes the intermediate values along like a production line. (src: [[2026-11-12-transformer-inference-arithmetic]])

## Details

- The payoff: per-GPU work (weight loading and flops) divides by the number of GPUs `N`. (src: [[2026-11-12-transformer-inference-arithmetic]])
- **Tensor parallelism** is the post's assumption. Each GPU holds a shard of each weight matrix, computes with its shard, and communicates whenever the shards need to be recombined. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Attention splits naturally by **head** (an attention head is an independent copy of the attention computation). Each GPU projects to its own heads, multiplies by its shard of the output projection `W_o`, then sends its result to all other GPUs and receives theirs; each adds an even share and the results are concatenated. (src: [[2026-11-12-transformer-inference-arithmetic]])
- The MLP block behaves the same way: `W_1` (expands to 4× `d_model`) and `W_2` (projects back) are sharded, requiring the same two communication rounds. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Total communication: **4 rounds per layer**, each moving a `d_model`-sized vector, so `4 · (N-1) · d_model / N` bytes per layer. The [[kv-cache]] is split by head, so each GPU keeps only its share. (src: [[2026-11-12-transformer-inference-arithmetic]])
- **Pipeline parallelism** evens out weight loading but leaves all but one GPU idle for a single request; it only works well when several requests can be pipelined through. It also fails to use the GPUs' full memory bandwidth, which is acceptable only if you are compute-bound. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Pipeline's one advantage is communication: it sends `d_model` per GPU in total, versus tensor parallelism's `N · d_model` per layer. (src: [[2026-11-12-transformer-inference-arithmetic]])

### The communication floor

- Compute time per token falls as `1/N`, but the four per-layer communication rounds do not shrink. Past some `N`, communication dominates and adding GPUs stops reducing latency. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Governing hardware ratio: flops per byte of link bandwidth. With 312e12 flops/s and 300 GB/s the A100 gives **1040 flops per byte**. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Per-layer flops per byte of comms are `3·d_model` (q,k,v), `d_model` (output projection), `4·d_model` (W1), `4·d_model` (W2), so models with `d_model` above ~1024 **per GPU** stay safely compute-bound; at 512 the margin is thin. (src: [[2026-11-12-transformer-inference-arithmetic]])
- The comparison must hold at *each* of the four steps, not on average, if compute and comms are to overlap. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Link numbers used: A100 NVLink 300 GB/s unidirectional — NVIDIA's 600 GB/s figure counts both directions. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Fixed per-message latency (~8 µs, from a V100 NVLink measurement) matters at small batch: Gopher-260B on 16 GPUs pays `4 · 80 · 8 µs` ≈ 2.5 ms per token, which the source counts as "somewhat significant" even at batch 512. It separately cites 11% residual comms latency for a 52B model on 4 GPUs at batch 512. (src: [[2026-11-12-transformer-inference-arithmetic]])

## Why it matters for a GPU cloud

- Multi-GPU inference requires NVLink-class links inside a host; scheduling such a job means allocating *all* the GPUs on a node (a gang/topology constraint), not one GPU from a pool. (general knowledge; the bandwidth numbers are src)
- The comms floor is why "more GPUs" is not a linear speedup claim. Quota policies and sales material that imply otherwise will not survive a benchmark. (general knowledge)
- Pipeline parallelism behaves like a different product: it wants many requests in flight, i.e. a batch service, not an interactive one. (src)
- Node shape (8-GPU NVLink domains, PCIe vs SXM) determines which models a customer can run at all on one node. (general knowledge)

## Related

- [[inference-latency-model]]: the equations whose comms terms come from here.
- [[gpu-memory-capacity]]: the reason to shard at all.
- [[kv-cache]]: sharded alongside the heads.
- [[memory-bandwidth]]: the per-GPU bandwidth that comms is weighed against.
- [[flops]]: peak flops used in the 1040 ratio.
