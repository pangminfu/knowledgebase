---
type: source
tags: [provisioning, tenancy-quotas, lifecycle, metering-billing]
sources: []
raw: raw/2026-11-22-training-vs-inference.md
url: https://handbook.modular.com/llm-inference-basics/training-inference-differences/
author: Modular (LLM Inference Handbook)
created: 2026-11-22
updated: 2026-11-24
---

# Training vs. inference (LLM Inference Handbook)

Raw: [[raw/2026-11-22-training-vs-inference]] · Modular, *LLM Inference Handbook*, chapter "LLM inference basics". Undated web page, clipped 2026-11-22. It follows [[2026-11-21-what-is-llm-inference]].

## Summary

This short introductory page contrasts the two phases of a model's life. **Training** builds the model: it adjusts the model's parameters (weights) using huge datasets, often on expensive GPU/TPU clusters; the page's table puts it at hours to weeks and calls it mostly a one-time cost. **Inference** uses the finished model on new inputs. It runs continuously, answers each request in milliseconds to seconds, and costs money for as long as traffic keeps coming, so over time it can become the larger expense. The page gives a side-by-side table covering purpose, data, compute, cost model, hardware, time, and tools. Its FAQs define model serving, place training and inference in the model lifecycle, and classify [[fine-tuning]] as training because it changes weights. Concept page: [[training-vs-inference]], which also notes where the table's time ranges understate real workloads ([[training-vs-inference#Contradictions]]).

**Why a control-plane engineer should care:** a GPU cloud sells to both phases, and they are different products. Training is a large, fixed-duration, all-or-nothing batch job. Inference is an always-on, autoscaled, latency-sensitive service. Each needs its own provisioning flow, quota shape, lifecycle handling, and billing meter. The mapping is in [[training-vs-inference]]. (general knowledge)

There are no images. The page has an interactive "LLM Lifecycle Visualizer" with two stages. **Build** is data → train → evaluate. **Operate** is deploy → infer → optimize. Live usage and fresh data feed the next build cycle. Its captions say "build once, then operate continuously" and "quality while building; latency & cost in production." (src: [[2026-11-22-training-vs-inference]])

## Key points

### Training

- Training teaches the model to recognize patterns and make predictions. It exposes the model to large amounts of data and adjusts its parameters based on what it sees. (src: [[2026-11-22-training-vs-inference]])
- Three techniques are named. **Supervised learning** shows inputs paired with correct outputs. **Reinforcement learning** learns by trial and error from feedback or rewards. **Self-supervised learning** predicts missing or corrupted parts of the data, with no explicit labels. (src: [[2026-11-22-training-vs-inference]])
- Training is computationally intensive and often needs expensive GPU or TPU clusters. The cost is "more or less a one-time expense". Retraining happens periodically to update or improve the model. (src: [[2026-11-22-training-vs-inference]])

### Inference

- Inference applies the trained model to new data. It runs "continuously and in real-time", responding to user input as it arrives. (src: [[2026-11-22-training-vs-inference]])
- Each request consumes GPU resources. A single inference step is smaller than training, but cumulative demand grows with traffic and can lead to large operating expenses. (src: [[2026-11-22-training-vs-inference]])
- Better-trained and better-fine-tuned models generally give more accurate inference. (src: [[2026-11-22-training-vs-inference]])

### The comparison table (verbatim content)

| Item | Training | Inference |
| --- | --- | --- |
| Purpose | Teach the model | Use the model |
| Data | Huge datasets | New, user-provided inputs |
| Compute | Long, expensive GPU/TPU jobs | Real-time, repeated workloads |
| Cost model | Mostly one-time | Ongoing and scales with traffic |
| Hardware | Multi-node clusters | Smaller clusters, optimized runtimes and cache usage |
| Time | Hours to weeks | Milliseconds to seconds |
| Tools | PyTorch, JAX, DeepSpeed, Megatron | vLLM, SGLang, TensorRT LLM, MAX, LMDeploy |

(src: [[2026-11-22-training-vs-inference]])

### FAQs

- **Model serving** is the production process of making a trained model available to applications, users, or systems so it can run inference on new inputs. It includes packaging the model, loading it onto suitable hardware, exposing it through an API or service, monitoring its behavior, and keeping latency, reliability, security, and cost under control. See [[inference-serving]]. (src: [[2026-11-22-training-vs-inference]])
- **Lifecycle order:** training → alignment → optional fine-tuning → evaluation → deployment → inference. In the page's words, training is "building the model" and inference is "putting the model to work." (src: [[2026-11-22-training-vs-inference]])
- **Inference often costs more than training over a model's life.** Training happens once. Inference runs on every request, and each request uses GPU compute, memory, and network bandwidth. Heavy usage and long prompts make this worse. The page gives no numbers. (src: [[2026-11-22-training-vs-inference]])
- **Should you train your own LLM?** In most cases, no. Training from scratch needs massive datasets, specialized hardware, and a dedicated research team. Most companies get better results by starting from an existing open-source model and fine-tuning or customizing it. Full training makes sense only for a problem existing models cannot handle, or for strict control requirements fine-tuning cannot meet. (src: [[2026-11-22-training-vs-inference]])
- **Fine-tuning is training.** In the page's words it updates "some of the model's weights" using new data. Inference never changes weights. (src: [[2026-11-22-training-vs-inference]]) Which weights depends on the method: full fine-tuning updates all of them, LoRA freezes them and trains a small adapter ([[fine-tuning#Which weights change]]). (general knowledge)

## Takeaways

- For a cloud, training is a scheduled batch job: many GPUs, all at once, for hours to months, paid for as reserved GPU-hours. Inference is a service: replicas that scale with traffic, judged on latency, and billed per token or per endpoint-hour. Each domain handles the two differently (table in [[training-vs-inference]]). (src for the contrast: [[2026-11-22-training-vs-inference]]; cloud framing: general knowledge)
- Inference cost keeps growing after launch: training cost is paid roughly once, while inference cost scales with traffic and never stops while the product is live. (src: [[2026-11-22-training-vs-inference]])
- Fine-tuning is training-shaped work, and its output is a model artifact that the inference side then has to deploy ([[fine-tuning]]). (src: [[2026-11-22-training-vs-inference]]; artifact handoff: general knowledge)

## Concepts & entities touched

[[training-vs-inference]] · [[fine-tuning]] · [[llm-inference]] · [[inference-serving]] · [[inference-frameworks]] · [[per-token-pricing]] · [[inference-optimization]]

## Questions raised

- The page asserts that inference often costs more than training but gives no figures. What is the actual split for a frontier model or a typical company? Look for provider or analyst estimates.
- "Smaller clusters" for inference holds for small models. Large models need multi-GPU or multi-node replicas ([[model-parallelism]]). Where is the crossover?
- How do GPU clouds sell training capacity: reserved blocks, spot/preemptible capacity, or cluster-as-a-service with Slurm or Kubernetes? Which quota and billing units go with each?
- The page names training tools (PyTorch, JAX, DeepSpeed, Megatron) without explaining them. Only needed if the wiki covers training clusters in depth.
