---
type: concept
tags: [provisioning, lifecycle]
sources: [2026-11-21-what-is-llm-inference, 2026-11-22-training-vs-inference]
created: 2026-11-21
updated: 2026-11-24
---

# Inference frameworks (inference servers)

An **inference server** is the program that loads a model onto GPUs, accepts requests, runs the model, and returns the output. An **inference framework** is the broader toolkit (APIs, optimizations, integrations) that such a server is built from. People use the two names somewhat interchangeably, though the server emphasizes the runtime and the framework the toolkit. (src: [[2026-11-21-what-is-llm-inference]]) Analogy: a database engine such as PostgreSQL, but for models instead of tables. For a GPU cloud, this is the workload running on each replica, and the control plane deploys, scales, and monitors it. (general knowledge)

## Details

- When a prompt arrives, the server allocates resources, executes the model, and returns the output. (src: [[2026-11-21-what-is-llm-inference]])
- Built-in features beyond request/response, per the handbook (src: [[2026-11-21-what-is-llm-inference]]):
  - **Batching**: combining multiple requests to improve GPU efficiency ([[inference-batching]]).
  - **Streaming**: sending tokens as they are generated, so the user sees output sooner (the source says "for lower latency"; total generation time is unchanged, general knowledge).
  - **Scaling**: spinning replicas up or down with demand.
  - **Monitoring**: exposing metrics for performance and debugging.
- In practice, an open-source engine such as vLLM runs on the fixed set of GPUs it was started with; adding and removing replicas is usually done by the platform around it, e.g. [[kubernetes-scaling]]. (general knowledge)
- *Server* emphasizes the runtime that receives requests and returns results. *Framework* emphasizes the library of APIs and optimizations. "Inference framework" and "serving framework" often mean the same systems, because modern tools do both jobs ([[inference-serving]]). (src: [[2026-11-21-what-is-llm-inference]])

### Named frameworks

| Framework | Who makes it | What it is / known for | License |
| --- | --- | --- | --- |
| vLLM | Started at UC Berkeley's Sky Computing Lab; now a community project | "Library for LLM inference and serving"; introduced PagedAttention, which stores the [[kv-cache]] in fixed-size pages like OS virtual memory; also continuous batching, chunked prefill, prefix caching, OpenAI-compatible server | Apache 2.0 |
| SGLang | Hosted by LMSYS (non-profit) | Serving framework for LLMs and multimodal models; introduced RadixAttention for automatic KV-cache (prefix) reuse | Apache 2.0 |
| MAX | Modular, the handbook's publisher | Modular's AI framework and inference server (OpenAI-compatible endpoint), alongside the Mojo language | Mixed: parts Apache 2.0 with LLVM exceptions, parts Modular's own MAX license |
| TensorRT LLM | NVIDIA | Optimized kernels and runtime for LLM inference on NVIDIA GPUs; NVIDIA moved FasterTransformer development to it, and FasterTransformer is the engine [[2026-11-12-transformer-inference-arithmetic]] benchmarked. Older docs spell it "TensorRT-LLM". | Apache 2.0 |
| LMDeploy | InternLM GitHub org; developed by OpenMMLab's MMRazor and MMDeploy teams | "Toolkit for compressing, deploying, and serving LLMs" (TurboMind and PyTorch engines) | Apache 2.0 |

Sources: the first four are named by [[2026-11-21-what-is-llm-inference]] ("popular"; they run inference, manage requests, and expose APIs); LMDeploy is listed among inference tools by [[2026-11-22-training-vs-inference]]. Columns 2–4 are general knowledge; verified: [vLLM repo](https://github.com/vllm-project/vllm), [vLLM PagedAttention paper](https://arxiv.org/abs/2309.06180), [SGLang repo](https://github.com/sgl-project/sglang), [SGLang paper](https://arxiv.org/abs/2312.07104), [Modular repo](https://github.com/modular/modular), [TensorRT LLM repo](https://github.com/NVIDIA/TensorRT-LLM), [FasterTransformer repo](https://github.com/NVIDIA/FasterTransformer), [LMDeploy repo](https://github.com/InternLM/lmdeploy). Each will get its own entity page once a source covers it in depth.

These are *inference* tools. Training uses a different stack (PyTorch, JAX, DeepSpeed, Megatron), so a GPU cloud that serves both phases deploys different software for each ([[training-vs-inference]]). (src: [[2026-11-22-training-vs-inference]])

## Why it matters for a GPU cloud

- The framework is the unit the control plane deploys: a [[container]] image with a model and a GPU allocation, run as a [[kubernetes-pod|Pod]]. Its startup time (loading weights) sets cold-start time for autoscaling, and its health endpoint and metrics feed lifecycle and metering. (general knowledge)
- Framework choice changes how many requests fit on a GPU (batching and cache management), so the same SKU can yield different tokens/sec per dollar. (general knowledge; see [[inference-optimization]])

## Related

- [[inference-serving]]: the per-replica part of serving lives here; the cross-replica part is the platform's.
- [[inference-optimization]]: the techniques these frameworks implement.
- [[llm-inference]]: the computation they run.
- [[training-vs-inference]]: training tools vs inference tools.

## Open questions

- Which handbook chapter compares frameworks ("choosing the right inference framework"), and how even-handed is it, given that Modular sells MAX?
