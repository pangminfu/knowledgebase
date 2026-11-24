---
type: concept
tags: [lifecycle, provisioning]
sources: [2026-11-22-training-vs-inference]
created: 2026-11-22
updated: 2026-11-24
---

# Fine-tuning

**Fine-tuning** takes an existing trained model and trains it further on new data, so that it fits a specific task or domain. It is a form of training, not inference, because it changes the model's learned weights; inference only uses weights. (src: [[2026-11-22-training-vs-inference]]) Analogy: hiring an experienced cook and teaching them your restaurant's menu, instead of training someone from scratch. (general knowledge)

## Details

- In the model lifecycle it comes after base training and alignment, and before evaluation and deployment. It is optional. (src: [[2026-11-22-training-vs-inference]])
- Most companies should start from an existing open-source model and fine-tune or otherwise customize it, rather than train their own. Training from scratch needs massive datasets, specialized hardware, and a dedicated research team. It makes sense only for a problem existing models cannot handle, or for strict control requirements fine-tuning cannot meet. (src: [[2026-11-22-training-vs-inference]])

### Which weights change

The handbook says fine-tuning updates "some of the model's weights" (src: [[2026-11-22-training-vs-inference]]). In practice that depends on the method:

| Method | What is trained | What stays frozen | Output artifact |
| --- | --- | --- | --- |
| **Full fine-tuning** | All of the base model's weights | Nothing | A complete new copy of the model, same size as the base |
| **LoRA** (Low-Rank Adaptation) | Small added low-rank matrices next to chosen layers (the **adapter**) | All original base weights | A small adapter file, used together with the unchanged base model |
| **QLoRA** | LoRA adapters | Base weights, stored quantized to 4 bits to save GPU memory | A LoRA adapter |

- LoRA is the best-known **parameter-efficient fine-tuning** (PEFT) method. For GPT-3 175B, the LoRA paper reports about 10,000× fewer trainable parameters and about 3× less GPU memory than full fine-tuning with Adam. (general knowledge; verified: [LoRA: Low-Rank Adaptation of Large Language Models](https://arxiv.org/abs/2106.09685))
- QLoRA back-propagates through a frozen 4-bit base model into LoRA adapters; the paper fine-tunes a 65B model on a single 48 GB GPU. (general knowledge; verified: [QLoRA: Efficient Finetuning of Quantized LLMs](https://arxiv.org/abs/2305.14314))
- Because a LoRA adapter is small and the base stays unchanged, one serving replica can hold one base model and apply different adapters per request; vLLM supports this (`--enable-lora`, `--max-loras` = adapters per batch). (general knowledge; verified: [vLLM docs: LoRA Adapters](https://docs.vllm.ai/en/latest/features/lora/))

## Why it matters for a GPU cloud

- A fine-tuning run is shaped like training: a batch job billed in GPU-hours, with a bounded duration and checkpoints. It is usually far smaller than pretraining; PEFT runs often fit on one node or even one GPU. (general knowledge)
- Its output is an artifact the inference side must deploy: a full model (full fine-tuning) or an adapter (LoRA). Platforms that serve many customer adapters on shared base-model replicas must route each request to the right adapter and meter usage per adapter. (general knowledge)

## Related

- [[training-vs-inference]]: fine-tuning sits on the training side of that split.
- [[inference-serving]]: where the fine-tuned model or adapter ends up.
- [[inference-frameworks]]: engines such as vLLM that can serve multiple adapters on one base model.

## Open questions

- The handbook has a fine-tuning section (model preparation). Ingest it if fine-tuning-as-a-service becomes a focus.
- What do many LoRA adapters on one replica cost in batching throughput and GPU memory? Needs measurements.
