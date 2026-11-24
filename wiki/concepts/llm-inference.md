---
type: concept
tags: [provisioning, metering-billing]
sources: [2026-11-21-what-is-llm-inference, 2026-11-12-transformer-inference-arithmetic, 2026-11-22-training-vs-inference]
created: 2026-11-21
updated: 2026-11-24
---

# LLM inference

**LLM inference** is *running* a trained large language model: a prompt goes in and generated text (or code, a summary, a translation) comes out. Training is how the model learns its parameters (the billions of numbers that make up the model). Inference uses those parameters without changing them. Analogy: training writes the cookbook, inference cooks a dish from it on request. Strictly, inference is only the computation. The production system around it is [[inference-serving]]. (src: [[2026-11-21-what-is-llm-inference]]) Inference compute is spent on every request, so its cost is ongoing and grows with traffic, unlike the mostly one-time cost of training ([[training-vs-inference]]). (src: [[2026-11-22-training-vs-inference]])

## Details

### Vocabulary

- **Token:** the unit the model reads and writes, usually a word or part of a word. A **tokenizer** converts text to token IDs on the way in and back to text on the way out. (general knowledge)
- **Forward pass:** one run of the input through every layer of the model. Its output is a probability distribution over which token comes next. (src: [[2026-11-21-what-is-llm-inference]])
- **Prompt:** the user's input, usually natural language. (src: [[2026-11-21-what-is-llm-inference]])

### What happens to one request

The handbook's visualizer shows **Tokenizer → Scheduling → Prefill → Output**; its text adds that prefill is followed by "the decode loop", which produces the output (src: [[2026-11-21-what-is-llm-inference]]). The diagram below draws the decode loop explicitly:

```mermaid
flowchart LR
  A[Prompt text] --> B[Tokenizer]
  B --> C[Scheduling<br/>queue + batch]
  C --> D[Prefill<br/>read whole prompt]
  D --> E[Decode loop<br/>one token per step]
  E -->|streamed| F[Output text]
  E -->|next token| E
```

- Tokenizing runs on the CPU and is cheap compared with the GPU work. Scheduling is a serving decision: which requests share a GPU pass ([[inference-batching]]). (general knowledge)
- Prefill and decode are the GPU work. Prefill processes the whole prompt in parallel, fills the [[kv-cache]], and produces the first output token. Decode generates the rest one token at a time, reusing the KV cache. See [[prefill-and-decode]]. (src: [[2026-11-12-transformer-inference-arithmetic]], [[2026-11-21-what-is-llm-inference]]; first token from prefill: general knowledge; verified: [Sarathi-Serve (Agrawal et al., 2024)](https://arxiv.org/abs/2403.02310))
- Every token pushed through the model (prompt or generated) costs about 2 [[flops]] per model parameter; attention and other small operations add little at short context. With a small batch, a decode step's time is set by reading all the weights from memory rather than by compute ([[gpu-performance-regimes]]). (src: [[2026-11-12-transformer-inference-arithmetic]])

### Inference vs serving

| | Inference | Serving |
| --- | --- | --- |
| What it is | The computation: tokens in, next-token distribution out, repeated through prefill and decode | Everything needed to offer that computation as a reliable production service |
| Runs where | Anywhere, even a Jupyter notebook with no server | Behind an API (HTTP/gRPC), on a fleet of replicas |
| Concerns | Speed per token, memory, numerical precision | Queuing, batching across users, load balancing, autoscaling, model versions, health checks, observability, resource allocation |
| Who owns it | The model + inference engine | Inference framework + platform/control plane |

(src: [[2026-11-21-what-is-llm-inference]]; "who owns it" row: general knowledge)

## Related

- [[inference-serving]]: the production system around inference.
- [[prefill-and-decode]]: the two GPU phases of one request.
- [[inference-frameworks]]: software that performs inference and usually serving too (vLLM, SGLang, TensorRT LLM, MAX).
- [[inference-optimization]]: techniques to make inference faster and cheaper.
- [[inference-latency-model]]: how long one decode step takes, from first principles.
- [[training-vs-inference]]: how inference differs from training in workload, cost, and what the control plane does with each.
- [[fine-tuning]]: changes weights, so it is training, not inference.
