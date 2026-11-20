---
type: source
tags: [provisioning, tenancy-quotas, lifecycle, metering-billing, career]
sources: []
raw: raw/2026-11-21-what-is-llm-inference.md
url: https://handbook.modular.com/llm-inference-basics/what-is-llm-inference/
author: Modular (LLM Inference Handbook)
created: 2026-11-21
updated: 2026-11-21
---

# What is LLM inference? (LLM Inference Handbook)

Raw: [[raw/2026-11-21-what-is-llm-inference]] · Modular, *LLM Inference Handbook*, chapter "LLM inference basics". Undated web page, clipped 2026-11-21.

## Summary

This is the opening chapter of a vendor-written handbook (Modular makes the MAX framework), and it is mostly vocabulary. It defines [[llm-inference]] as using a trained model to turn a prompt into output, and an inference server as the runtime that loads models onto GPUs and handles requests ([[inference-frameworks]]). Its most useful point is the split between **inference** (the computation, a forward pass) and **serving** (everything that turns that computation into a production service: API, queuing, batching, load balancing, autoscaling, model versioning, health checks, observability, resource allocation). See [[inference-serving]]. It then lists the standard [[inference-optimization]] techniques and argues that teams eventually outgrow per-token serverless APIs and move to hybrid or self-hosted serving ([[per-token-pricing]]).

**Why a control-plane engineer should care:** the "serving" list on this page is close to a job description for a GPU cloud's inference control plane. Each item maps to one of the four target domains (mapping in [[inference-serving]]). (general knowledge)

There are no images. The page has an interactive visualizer of one request's path: **Tokenizer → Scheduling → Prefill → Output** (streamed). (src: [[2026-11-21-what-is-llm-inference]])

## Key points

### Definitions

- LLM inference means using a trained LLM (the page names GPT-5, GLM-5, DeepSeek-V3.2) to generate output from user input, usually a natural-language prompt. The model runs the prompt through its parameters to produce text, code, summaries, or translations. (src: [[2026-11-21-what-is-llm-inference]])
- An **inference server** loads models, connects them to hardware (GPUs), and processes application requests. When a prompt arrives it allocates resources, runs the model, and returns the output. (src: [[2026-11-21-what-is-llm-inference]])
- Beyond request/response, inference servers provide **batching** (combining requests for GPU efficiency), **streaming** (sending tokens as they are generated), **scaling** (adding or removing replicas with demand), and **monitoring** (exposing metrics). (src: [[2026-11-21-what-is-llm-inference]])
- "Inference server" and "inference framework" are used almost interchangeably. *Server* stresses the runtime that handles requests. *Framework* stresses the broader toolkit of APIs, optimizations, and integrations. Named examples: vLLM, SGLang, MAX, TensorRT LLM. (src: [[2026-11-21-what-is-llm-inference]])

### Inference vs serving

- **Inference is the computation:** tokens in, a probability distribution over the next token out, repeated through prefill and the decode loop ([[prefill-and-decode]]). You can run it in a notebook with no server at all. (src: [[2026-11-21-what-is-llm-inference]])
- **Serving is the production process around it:** HTTP/gRPC API, request queuing and scheduling, batching across concurrent requests, load balancing across replicas, autoscaling, model loading and version management, health checks, observability, resource allocation. (src: [[2026-11-21-what-is-llm-inference]])
- People blur the two because modern frameworks do both, but "the distinction matters the moment you start designing infrastructure." (src: [[2026-11-21-what-is-llm-inference]])

### Inference optimization (named, not explained, on this page)

- Goal: lower latency, higher throughput, and lower hardware cost without hurting model quality. (src: [[2026-11-21-what-is-llm-inference]])
- Techniques listed: continuous batching, KV cache reuse/offloading, speculative decoding, quantization (INT8, FP8), prefix caching, multi-GPU parallelism. Each has its own later chapter in the handbook. Roadmap: [[inference-optimization]]. (src: [[2026-11-21-what-is-llm-inference]])

### Business framing

- Serverless APIs (OpenAI, Anthropic, and others) hide infrastructure, optimization, and scaling; you "pay by the token." (src: [[2026-11-21-what-is-llm-inference]])
- As usage grows, teams hit limits on cost, latency, customization, or compliance that serverless APIs cannot fully address, and they move to hybrid or self-hosted setups. (src: [[2026-11-21-what-is-llm-inference]])
- Claim: a poorly optimized setup "can cost 10× more in GPU hours while delivering worse performance." The page gives no measurement behind this. Treat it as an illustrative order of magnitude, not data. (src: [[2026-11-21-what-is-llm-inference]])

## Interview talking points

- **"What's the difference between inference and serving?"** Inference is the forward pass that turns tokens into next-token probabilities. Serving is the system that makes it a reliable service: API, queue, scheduler, batching, replicas, autoscaling, model versions, health, metrics. A control plane owns most of the serving list. (src: [[2026-11-21-what-is-llm-inference]]; control-plane framing: general knowledge)
- Name the request path (tokenize → schedule → prefill → decode/stream) and say where the GPU-bound work happens (prefill and decode). (src: [[2026-11-21-what-is-llm-inference]], [[2026-11-12-transformer-inference-arithmetic]])
- Explain why per-token prices and GPU-hour prices are different products: the provider's margin depends on batching efficiency. See [[per-token-pricing]].

## Concepts & entities touched

[[llm-inference]] · [[prefill-and-decode]] · [[inference-serving]] · [[inference-frameworks]] · [[inference-optimization]] · [[per-token-pricing]] · [[inference-batching]] · [[kv-cache]] · [[model-parallelism]]

## Questions raised

- Source bias: Modular sells MAX, and the page lists it among the "popular" frameworks. Later handbook chapters on framework choice should be read with that in mind.
- The "10× more GPU hours" claim: what workload and baseline? Later chapters (continuous batching, quantization) may quantify it.
- Which serving responsibilities live inside the inference framework (batching, KV cache management) and which belong to the platform around it (autoscaling, routing, quotas, billing)? Where does a GPU cloud draw that line?
- How do serverless providers turn GPU-hours into per-token prices, and how do they handle the gap between input (prefill) and output (decode) token costs?
