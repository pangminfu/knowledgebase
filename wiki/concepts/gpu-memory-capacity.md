---
type: concept
tags: [provisioning, tenancy-quotas, metering-billing]
sources: [2026-11-12-transformer-inference-arithmetic]
created: 2026-11-12
updated: 2026-11-24
---

# GPU memory capacity

**GPU memory capacity** is how many bytes of on-board memory (HBM, a kind of DRAM) a GPU, or a set of GPUs, can hold at once. For inference it is the hard limit on *how much model and how many concurrent requests* a replica can serve: the weights and the per-request [[kv-cache]] must both fit. (src: [[2026-11-12-transformer-inference-arithmetic]]) Unlike speed, it cannot be traded for time: if they do not fit, requests must be queued or refused, or the model must use fewer bytes (quantisation) or more GPUs. Analogy: a warehouse's floor area, not its loading docks. (general knowledge)

## Details

- Three occupants of GPU memory during inference: **weights** (the model's parameters), the **kv cache** (one entry per token in flight), and **intermediate activations** (temporaries). The source treats the third as negligible. (src: [[2026-11-12-transformer-inference-arithmetic]])
- Weights at 16-bit precision: parameters × 2 bytes. A 52B model is 104 GB, so it does **not** fit on a 40 GB [[nvidia-a100]]: three GPUs minimum, four in practice. The source wants four for larger batches and because "it's silly to divide powers of two over three GPUs". (src: [[2026-11-12-transformer-inference-arithmetic]])
- The leftover after weights is the cache budget, at ≈ 2 MB (2,097,152 bytes) per token for this model:

  | GPUs (A100-40GB) | Memory | Left for cache | Tokens (post's rounding: ÷ 0.002 GB) | Tokens (exact: ÷ 2,097,152 B) | 2048-token requests |
  | --- | --- | --- | --- | --- | --- |
  | 3 | 120 GB | 16 GB | 8,000 | ≈ 7,600 | ≈ 3.7 (post: "up to 4") |
  | 4 | 160 GB | 56 GB | 28,000 (post prints "≈ 23,000") | ≈ 26,700 | ≈ 13 |

  (src: [[2026-11-12-transformer-inference-arithmetic]]; exact columns and the 23,000 discrepancy: arithmetic check, this wiki — see Contradictions)
- So for a fixed model and GPU count, **max concurrency is a memory calculation**: `(total GPU memory − weights − overhead) ÷ bytes per cached token ÷ context length`. (src: [[2026-11-12-transformer-inference-arithmetic]], generalising its worked example)
- At high volume you run several replicas (copies) of the model. Each replica pays for its own copy of the weights, so each should run as large a batch as it can. (src: [[2026-11-12-transformer-inference-arithmetic]])
- The post notes the 80 GB A100 variant (2e12 bytes/s) was not offered by large clouds at the time (March 2022), so it treats capacity as 40 GB per GPU. (src: [[2026-11-12-transformer-inference-arithmetic]]) Both variants are listed by NVIDIA (40 GB HBM2 at 1,555 GB/s; 80 GB HBM2e at 2,039 GB/s SXM), and AWS now offers the 80 GB card in p4de instances. (general knowledge; verified: [NVIDIA A100 datasheet](https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/a100/pdf/nvidia-a100-datasheet-us-nvidia-1758950-r4-web.pdf), [Amazon EC2 P4](https://aws.amazon.com/ec2/instance-types/p4/))

## Why it matters for a GPU cloud

- **Provisioning:** GPU memory size is a SKU axis (40 vs 80 GB, HBM generations), and the *quotable* fact for a customer is "this SKU holds models up to X and contexts up to Y at concurrency Z". (general knowledge)
- **Tenancy and quotas:** concurrency limits are memory limits in disguise. Quota systems that count GPUs but not context length will overcommit and hit out-of-memory at admission time. (general knowledge)
- **Scheduling/lifecycle:** a serving replica must reserve weights plus worst-case cache, or be prepared to reject requests; memory pressure is a first-class scheduling signal, not just an alert. (general knowledge)
- **Billing:** cache held for an idle conversation occupies memory other requests could use. Per-token APIs therefore free it between turns and either recompute the prefix or keep it briefly under a priced cache. Anthropic's prompt caching, for example, keeps a prefix for 5 minutes by default (1 hour optional), bills cache writes at 1.25× (5-minute) or 2× (1-hour) the base input price and cache reads at 0.1×. (general knowledge; verified: [Prompt caching — Claude docs](https://docs.claude.com/en/docs/build-with-claude/prompt-caching))

## Related

- [[kv-cache]]: the per-request consumer of leftover memory.
- [[inference-batching]]: batch size is bounded above by capacity.
- [[model-parallelism]]: how a model too large for one GPU is spread across several.
- [[nvidia-a100]]: the 40/80 GB SKU examples.
- [[memory-bandwidth]]: the other, speed-related memory property.
- [[2026-11-12-sizing-a-gpu-inference-deployment]]: the whole chain in one worked example.

## Contradictions

- The source prints `56/0.002 ≈ 23000` cached tokens for four A100-40GB (raw line 96; the live post still says 23,000 as of 2026-11-24). Its own inputs give 28,000, or ≈ 26,700 with the exact 2,097,152 bytes per token. (src: [[2026-11-12-transformer-inference-arithmetic]]; arithmetic check, this wiki; live text: [Transformer Inference Arithmetic](https://kipp.ly/p/transformer-inference-arithmetic))
