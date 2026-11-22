---
type: concept
tags: [lifecycle, provisioning]
sources: [2026-11-22-training-vs-inference]
created: 2026-11-22
updated: 2026-11-22
---

# Fine-tuning

**Fine-tuning** takes an existing trained model and updates some of its weights using new data, so that it fits a specific task or domain. It is a form of training, not inference, because it changes weights. Inference only uses weights. Analogy: hiring an experienced cook and teaching them your restaurant's menu, instead of training someone from scratch. (src: [[2026-11-22-training-vs-inference]])

## Details

- In the model lifecycle it comes after base training and alignment, and before evaluation and deployment. It is optional. (src: [[2026-11-22-training-vs-inference]])
- Most companies should fine-tune an existing open-source model rather than train their own. Training from scratch needs massive datasets, specialized hardware, and a research team. It is justified only when existing models cannot handle the problem, or when control requirements cannot be met by fine-tuning. (src: [[2026-11-22-training-vs-inference]])
- Common variants update only a small set of extra weights, an **adapter** (LoRA is the best-known method). The adapter is small, and many adapters can share one base model when serving. (general knowledge)

## Why it matters for a GPU cloud

- A fine-tuning run is shaped like training: a batch job with GPU-hours, a fixed duration, and checkpoints. Fine-tuning is usually far smaller than pretraining, often one node instead of a cluster. (general knowledge)
- Its output is a new model artifact that the inference side must deploy. Platforms that serve many customer adapters on shared base-model replicas have to route each request to the right adapter and meter usage per adapter. (general knowledge)

## Related

- [[training-vs-inference]]: fine-tuning sits on the training side of that split.
- [[inference-serving]]: where the fine-tuned model ends up.

## Open questions

- The handbook has a fine-tuning section (model preparation). Ingest it if fine-tuning-as-a-service becomes a focus.
- How do serving frameworks handle many LoRA adapters on one replica, and what does that do to batching and memory? Needs vLLM or similar docs.
