---
type: concept
tags: [provisioning]
sources: [2026-11-10-deep-learning-brrrr-first-principles, 2026-11-12-transformer-inference-arithmetic]
created: 2026-11-10
updated: 2026-11-22
---

# FLOPS

A **FLOP** is one floating-point operation: a single add or multiply on a decimal number. **FLOPS** is how many of those a chip does per second, and **TFLOPS** means trillions per second. GPU spec sheets advertise *peak* FLOPS, the best case under ideal conditions. Real programs reach *achieved* FLOPS, which is usually much lower. The gap between the two is the main subject of [[gpu-performance-regimes]]. (general knowledge)

## Details

- Peak FLOPS depends on the kind of math. Modern accelerators have special hardware for matrix multiplication ("matmul"), e.g. NVIDIA **Tensor Cores**. An [[nvidia-a100]] reaches 312 TFLOPS only on matmuls; everything else gets 19.5 TFLOPS. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])
- Even the 19.5 assumes fused multiply-add (FMA: multiply then add in one instruction, counted as 2 FLOPs). Fully general code reaches about 9.75 TFLOPS on an A100. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])
- TPUs (Google's accelerators) are even more specialized for matmul than GPUs. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])
- Matmuls ("tensor contractions") dominate FLOP counts but not runtime:

![[raw/assets/brrr-bert_flops.png]]
BERT operator classes: tensor contraction = 99.80% of FLOPs / 61.0% of runtime; normalization = 0.17% / 25.5%; element-wise = 0.03% / 13.5%. Non-matmul ops are tiny in math terms but spend their time waiting on [[memory-bandwidth]]. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

- So normalization achieves ~250× fewer FLOPS than matmuls, and pointwise ops ~700× fewer. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])
- Compute capacity grows faster than memory bandwidth:

![[raw/assets/brrr-bw_moores_law.png]]
Table from "Latency Lags Bandwidth" (CACM 2004): processor bandwidth doubles every ~1.6 years, memory modules every ~2.7 years. Each hardware generation is therefore harder to keep fed. (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

- Achieved FLOPS as a % of peak is a common measure of how compute-bound a workload is. (src: [[2026-11-10-deep-learning-brrrr-first-principles]]) In large-model training this is commonly called **MFU** (model FLOPS utilization). (general knowledge)

## Counting flops for a transformer

- A matrix-vector multiply of an `m×n` matrix by a vector costs `2mn` flops; matrix-matrix `2mnp`. The factor 2 is multiply + add. (src: [[2026-11-12-transformer-inference-arithmetic]])
- One token through a transformer costs `2 · P` flops, where `P` is the parameter count — one pass over every weight. Verified by summing per-layer terms: `n_layers · (6·d_model² for q,k,v + 2·d_model² for the output projection + 16·d_model² for the MLP)` = `24 · n_layers · d_model²`. (src: [[2026-11-12-transformer-inference-arithmetic]])
- For a 52B model (`d_model`=8192, 64 layers) that is 103 GFLOP/token, implying 51.5B parameters; the 0.5B difference is token embeddings, under 2%. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Attention's `softmax(q·k)·v` step is vector-vector work, scaling with `d_model`, so it never shows up in flop budgets — but it does show up in *runtime* because it is memory-bound. (src: [[2026-11-12-transformer-inference-arithmetic]])

## Why it matters for a GPU cloud

- Customers compare SKUs on headline TFLOPS. Those numbers depend on precision (FP64/FP32/TF32/BF16/FP16) and Tensor Core use. The A100 sheet's starred, doubled figures assume structured sparsity. (sparsity: general knowledge)
- Billing is per GPU-hour, not per FLOP, so poorly optimized workloads cost the customer more for the same result. (general knowledge)

## Related

- [[gpu-performance-regimes]]: FLOPS is the "compute" regime.
- [[memory-bandwidth]]: why achieved FLOPS falls short.
- [[nvidia-a100]]: spec-sheet numbers used here.
- [[gpu-utilization-metrics]]: % of peak FLOPS vs GPU-Util.
