---
type: concept
tags: [provisioning, lifecycle, career]
sources: [2026-11-21-what-is-llm-inference, 2026-11-22-training-vs-inference]
created: 2026-11-21
updated: 2026-11-22
---

# Inference frameworks (inference servers)

An **inference server** is the program that loads a model onto GPUs, accepts requests, runs the model, and returns the output. An **inference framework** is the broader toolkit (APIs, optimizations, integrations) that such a server is built from. In practice both names refer to the same software. Analogy: a database engine such as PostgreSQL, but for models instead of tables. For a GPU cloud, this is the workload running on each replica, and the control plane deploys, scales, and monitors it. (src: [[2026-11-21-what-is-llm-inference]])

## Details

- When a prompt arrives, the server allocates resources, executes the model, and returns the output. (src: [[2026-11-21-what-is-llm-inference]])
- Built-in features beyond request/response (src: [[2026-11-21-what-is-llm-inference]]):
  - **Batching**: combining multiple requests to improve GPU efficiency ([[inference-batching]]).
  - **Streaming**: sending tokens as they are generated, for lower perceived latency.
  - **Scaling**: spinning replicas up or down with demand.
  - **Monitoring**: exposing metrics for performance and debugging.
- *Server* emphasizes the runtime that receives requests and returns results. *Framework* emphasizes the library of APIs and optimizations. "Inference framework" and "serving framework" often mean the same systems, because modern tools do both jobs ([[inference-serving]]). (src: [[2026-11-21-what-is-llm-inference]])

### Named frameworks

| Framework | Source says | Notes |
| --- | --- | --- |
| vLLM | Listed as popular; runs inference, manages requests, exposes APIs | Open source; known for PagedAttention, a paged [[kv-cache]] (general knowledge) |
| SGLang | Same | Open source (general knowledge) |
| MAX | Same | Made by Modular, the handbook's publisher (general knowledge) |
| TensorRT LLM | Same | NVIDIA's; successor to FasterTransformer, which [[2026-11-12-transformer-inference-arithmetic]] benchmarked (general knowledge) |
| LMDeploy | Listed among inference tools in the training-vs-inference table (src: [[2026-11-22-training-vs-inference]]) | Open source, from the InternLM / Shanghai AI Lab ecosystem (general knowledge) |

(src: [[2026-11-21-what-is-llm-inference]] for the first four, [[2026-11-22-training-vs-inference]] for LMDeploy) Each will get its own entity page once a source covers it in depth.

These are *inference* tools. Training uses a different stack (PyTorch, JAX, DeepSpeed, Megatron), so a GPU cloud that serves both phases deploys different software for each ([[training-vs-inference]]). (src: [[2026-11-22-training-vs-inference]])

## Why it matters for a GPU cloud

- The framework is the unit the control plane deploys: a container image with a model and a GPU allocation. Its startup time (loading weights) sets cold-start time for autoscaling, and its health endpoint and metrics feed lifecycle and metering. (general knowledge)
- Framework choice changes how many requests fit on a GPU (batching and cache management), so the same SKU can yield different tokens/sec per dollar. (general knowledge; see [[inference-optimization]])

## Related

- [[inference-serving]]: the per-replica part of serving lives here; the cross-replica part is the platform's.
- [[inference-optimization]]: the techniques these frameworks implement.
- [[llm-inference]]: the computation they run.
- [[training-vs-inference]]: training tools vs inference tools.

## Open questions

- Which handbook chapter compares frameworks ("choosing the right inference framework"), and how even-handed is it, given that Modular sells MAX?
