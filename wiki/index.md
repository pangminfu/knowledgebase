# Index

Catalog of every page in the wiki. One line per page: link — summary.

## Overview

- [[overview]] — big picture: GPU physics, the four-step inference sizing chain, and the serving layer the control plane owns; domain coverage table; next sources

## Sources

- [[2026-11-10-deep-learning-brrrr-first-principles]] — Horace He (2022): compute vs memory-bandwidth vs overhead regimes on GPUs; why spec-sheet TFLOPS and GPU-Util mislead
- [[2026-11-12-transformer-inference-arithmetic]] — kipply (2022): closed-form LLM serving math — kv cache size, 208 flops/byte, capacity → batch size, tensor-parallel comms floor; validated against FasterTransformer
- [[2026-11-21-what-is-llm-inference]] — Modular LLM Inference Handbook intro: inference vs serving, inference servers/frameworks, optimization techniques list, serverless per-token vs self-hosted
- [[2026-11-22-training-vs-inference]] — Modular handbook: training (build, mostly one-time, multi-node, hours–weeks) vs inference (operate, ongoing, ms–s, scales with traffic); comparison table; fine-tuning is training
- [[2026-11-22-k8s-viewing-pods-and-nodes]] — Kubernetes Basics Module 3: Pods (shared IP/volumes, atomic unit), Nodes (VM or physical; kubelet + containerd), scheduling by free resources, kubectl get/describe/logs/exec/proxy

## Concepts

- [[gpu-performance-regimes]] — compute-, memory-, overhead-bound: how to diagnose each and what fixes it; why single-request inference is memory-bound
- [[flops]] — floating-point ops/sec; peak vs achieved; Tensor Cores; flop counting for a transformer (2 flops per parameter)
- [[memory-bandwidth]] — DRAM↔SRAM data movement, memory-bound ops, compute intensity, the 208 flops/byte SKU ratio
- [[operator-fusion]] — merging ops into one kernel to skip memory round-trips; what unfused softmax costs
- [[framework-overhead]] — Python/framework/kernel-launch time that idles the GPU; how to detect it
- [[gpu-utilization-metrics]] — GPU-Util vs % of peak FLOPS vs DCGM; implications for metering
- [[kv-cache]] — the per-token attention cache: why sampling needs it, what it costs (~2 MB/token at 52B), when to skip it, offload/prefix-caching
- [[gpu-memory-capacity]] — weights vs kv cache vs activations; how leftover memory sets max concurrency
- [[inference-batching]] — tokens in flight as the batch axis; 208-token crossover; batch size trades latency, throughput, cost; continuous batching pointer
- [[model-parallelism]] — tensor vs pipeline parallelism; 4 comms/layer, the 1040 flops-per-byte test, and the latency floor
- [[inference-latency-model]] — the two-regime equations (memory-bound / flops-bound) for per-token latency, and why they land within 10–35%
- [[llm-inference]] — what running a model means; tokens, forward pass, request path (tokenize → schedule → prefill → decode); inference vs serving interview table
- [[prefill-and-decode]] — the two GPU phases of a request: parallel prompt pass (compute-heavy) vs one-token-at-a-time decode (memory-bound); pricing asymmetry
- [[inference-serving]] — everything around inference that makes it a service; serving responsibilities mapped to the four control-plane domains; serverless vs self-hosted
- [[inference-frameworks]] — inference servers/frameworks (vLLM, SGLang, TensorRT-LLM, MAX): what they do per replica and why the control plane cares
- [[inference-optimization]] — hub and reading roadmap: which bottleneck each technique (batching, caching, quantization, speculative decoding, parallelism, routing) attacks
- [[per-token-pricing]] — per-token vs GPU-hour products; why token cost depends on batching; what each metering pipeline must capture
- [[training-vs-inference]] — the build vs operate lifecycle; comparison table; how each workload differs for provisioning, quotas, lifecycle, and billing
- [[fine-tuning]] — adapting an existing model by updating some weights; training-shaped job whose output the serving side deploys
- [[control-plane]] — control plane vs data plane at two levels: Kubernetes' own (API server, scheduler, controllers) and the GPU cloud's (the four domains)
- [[kubernetes-pod]] — smallest unit Kubernetes runs: containers sharing IP and volumes on one Node; replaced not moved; an inference replica = a Pod requesting GPUs
- [[kubernetes-node]] — a worker machine (VM or bare metal) running Pods, kubelet, and a runtime; GPU Node software and bare-metal lifecycle (cordon/drain)
- [[kubernetes-deployment]] — keeps N identical Pods running and replaces lost ones; rolling updates; maps to a dedicated inference endpoint
- [[kubernetes-scheduling]] — placing Pods on Nodes by resource requests; GPUs as whole extended resources; Pending, fragmentation, gang scheduling, quotas in front
- [[container]] — packaged app as an isolated process; image vs container vs registry; Docker vs containerd; GPU images and cold start

## Entities

- [[nvidia-a100]] — Ampere datacenter GPU; spec table, SKU variants (PCIe/SXM, 40/80 GB), NVLink 300 GB/s
- [[nvidia-smi]] — NVIDIA's GPU status CLI; DRAM usage, GPU-Util, health triage
- [[kubernetes]] — container orchestration system; core objects, declarative desired state, namespaces; what it means for each control-plane domain
- [[kubelet]] — per-Node agent that runs Pods for the control plane; heartbeats, probes, GPU device-plugin registration
- [[containerd]] — the container runtime that pulls and runs images on each Node; replaced Docker in Kubernetes; GPU access via NVIDIA toolkit
- [[kubectl]] — Kubernetes CLI cheat sheet: get/describe/logs/exec/proxy from the tutorial plus GPU-node triage commands

## Syntheses

- [[2026-11-12-sizing-a-gpu-inference-deployment]] — weights → GPUs, leftover memory → concurrency, concurrency → cost; worked 13B/52B example and what it implies for quotas, scheduling, metering
