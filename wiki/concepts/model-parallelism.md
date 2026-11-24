---
type: concept
tags: [provisioning, lifecycle]
sources: [2026-11-12-transformer-inference-arithmetic]
created: 2026-11-12
updated: 2026-11-24
---

# Model parallelism

**Model parallelism** means splitting one model across several GPUs. You need it when the weights do not fit on one GPU: a 52B-parameter model at 16 bits is 104 GB against 40 GB per card ([[gpu-memory-capacity]]). Two shapes matter: **tensor parallelism**, which splits every layer's weight matrices across GPUs and synchronizes inside each layer, and **pipeline parallelism**, which gives each GPU a fraction of the layers and passes intermediate values along like a production line. (src: [[2026-11-12-transformer-inference-arithmetic]]) It contrasts with data parallelism, which puts a full copy of the model on each GPU (or group) and splits the *requests*; for serving, that is simply running more replicas. (general knowledge)

## Details

- The payoff: per-GPU work (weight loading and flops) divides by the number of GPUs `N`. (src: [[2026-11-12-transformer-inference-arithmetic]])

### Tensor parallelism

- The post's assumption. Each GPU holds a shard of each weight matrix, computes with its shard, and communicates whenever the shards need to be recombined. (src: [[2026-11-12-transformer-inference-arithmetic]]) This is the scheme of Megatron-LM, one of the post's references. (general knowledge; verified: [Megatron-LM paper](https://arxiv.org/abs/1909.08053))
- Attention splits naturally by **head** (an attention head is an independent copy of the attention computation). Each GPU computes its own heads with no communication, multiplies by its shard of the output projection `W_o`, then sends its partial result to all other GPUs and receives theirs; each GPU sums an even share and the shares are exchanged again and concatenated. (src: [[2026-11-12-transformer-inference-arithmetic]])
- The MLP block works the same way: `W_1` (expands to 4× `d_model`) and `W_2` (projects back) are sharded, with the same two communication rounds at the end. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Total communication: **4 rounds per layer**, each a send/receive of `d_model/N`-sized pieces `N−1` times, so `4 · (N−1) · d_model / N` values per layer, ×2 bytes at 16-bit. The post calls this "bytes"; its own large-batch formula includes the factor 2. (src: [[2026-11-12-transformer-inference-arithmetic]]) Megatron-LM describes the same scheme as two all-reduce operations per layer in the forward pass. (general knowledge; verified: [Megatron-LM paper](https://arxiv.org/abs/1909.08053)) An all-reduce is commonly done as a reduce-scatter followed by an all-gather, which gives the post's two rounds per all-reduce. (general knowledge)
- The [[kv-cache]] is split by head, so each GPU keeps only its share. (src: [[2026-11-12-transformer-inference-arithmetic]])

### Pipeline parallelism

- Evens out weight loading but leaves all but one GPU idle for a single request. It works when several requests (or, in training, micro-batches) can be pipelined through. It also does not use all GPUs' memory bandwidth at once, which the post calls acceptable only if you are flops-bound anyway. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Its one advantage is communication: the post puts it at `d_model` per accelerator, versus `N · d_model` per layer for tensor parallelism. (src: [[2026-11-12-transformer-inference-arithmetic]])

### The communication floor

- Compute time per token falls as `1/N`, but the per-message latency of the four per-layer rounds does not. Past some `N`, communication dominates and adding GPUs stops reducing latency. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Governing hardware ratio: flops per byte of link bandwidth. With 312e12 flops/s and 300 GB/s the A100 gives **1040 flops per byte**. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Per-layer flops per byte of comms are `3·d_model` (q,k,v), `d_model` (output projection), `4·d_model` (W1), `4·d_model` (W2), so the post calls models with an embedding dimension over 1024 "(per chip)" safe, and 512 "a little awkward". (src: [[2026-11-12-transformer-inference-arithmetic]])
- The comparison must hold at *each* of the four steps, not on average, if compute and comms are to overlap. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Link numbers used: A100 NVLink 300 GB/s per direction; NVIDIA's 600 GB/s figure counts both directions. (src: [[2026-11-12-transformer-inference-arithmetic]]) NVIDIA's datasheet lists 600 GB/s NVLink for A100 SXM and for a two-GPU NVLink Bridge on PCIe cards, versus 64 GB/s over PCIe Gen4. (general knowledge; verified: [NVIDIA A100 datasheet](https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/a100/pdf/nvidia-a100-datasheet-us-nvidia-1758950-r4-web.pdf))
- Fixed per-message latency (~8 µs, from a V100 NVLink measurement) matters at small batch: Gopher-size 260B on 16 GPUs pays `4 · 80 · 8 µs` = 2.56 ms per decode step (the post rounds to ≈ 3 ms) against 22 ms of compute. The post calls this "somewhat significant" even at batch 512, but fine if comms and compute overlap. (src: [[2026-11-12-transformer-inference-arithmetic]]) The measurement is 8.55 µs peer-to-peer on V100 NVLink. (general knowledge; verified: [Dissecting the NVIDIA Volta GPU Architecture via Microbenchmarking, Table 5.1](https://arxiv.org/abs/1804.06826))
- The post also says comms for "our 512 batch on 52B" were still 11% latency, but it never shows that calculation; its worked batch-512 example (260B) gives 2.56 ms of 20.5 ms comms ≈ 12.5%. (src: [[2026-11-12-transformer-inference-arithmetic]]; arithmetic check, this wiki)

## Why it matters for a GPU cloud

- Multi-GPU inference needs NVLink-class links between the GPUs to scale well; over PCIe it works but comms cost more (64 GB/s vs 600 GB/s on A100). So a tensor-parallel replica belongs on several GPUs of *one* Node, ideally ones that share NVLink, not on GPUs pooled from anywhere. (general knowledge; the post assumes GPUs on the same host)
- In Kubernetes such a replica is one Pod requesting `N` GPUs: the default scheduler places it all-or-nothing on a single Node, but plain GPU counts do not guarantee *which* GPUs (NVLink-connected or not); that needs topology-aware setup. Gang scheduling is the separate problem of starting several Pods together, which arises when a replica spans Nodes. See [[kubernetes-scheduling]]. (general knowledge)
- The comms floor is why "more GPUs" is not a linear speedup claim. Quota policies and sales material that imply otherwise will not survive a benchmark. (general knowledge, from the source's comms floor)
- Pipeline parallelism behaves like a different product: it wants many requests in flight, i.e. a batch service, not an interactive one. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Node shape (8-GPU NVLink domains, PCIe vs SXM) determines which models a customer can run at all on one node. (general knowledge)

## Related

- [[inference-latency-model]]: the equations whose comms terms come from here.
- [[gpu-memory-capacity]]: the reason to shard at all.
- [[kv-cache]]: sharded alongside the heads.
- [[memory-bandwidth]]: the per-GPU bandwidth that comms is weighed against.
- [[flops]]: peak flops used in the 1040 ratio.
- [[kubernetes-scheduling]]: how a multi-GPU replica is placed on a Node, and why multi-Node gangs need extra tooling.
