# Index

Catalog of every page in the wiki. One line per page: link — summary.

## Overview

- [[overview]] — big picture: GPU physics, then the four-step inference sizing chain; domain coverage table; next sources

## Sources

- [[2026-11-10-deep-learning-brrrr-first-principles]] — Horace He (2022): compute vs memory-bandwidth vs overhead regimes on GPUs; why spec-sheet TFLOPS and GPU-Util mislead
- [[2026-11-12-transformer-inference-arithmetic]] — kipply (2022): closed-form LLM serving math — kv cache size, 208 flops/byte, capacity → batch size, tensor-parallel comms floor; validated against FasterTransformer

## Concepts

- [[gpu-performance-regimes]] — compute-, memory-, overhead-bound: how to diagnose each and what fixes it; why single-request inference is memory-bound
- [[flops]] — floating-point ops/sec; peak vs achieved; Tensor Cores; flop counting for a transformer (2 flops per parameter)
- [[memory-bandwidth]] — DRAM↔SRAM data movement, memory-bound ops, compute intensity, the 208 flops/byte SKU ratio
- [[operator-fusion]] — merging ops into one kernel to skip memory round-trips; what unfused softmax costs
- [[framework-overhead]] — Python/framework/kernel-launch time that idles the GPU; how to detect it
- [[gpu-utilization-metrics]] — GPU-Util vs % of peak FLOPS vs DCGM; implications for metering
- [[kv-cache]] — the per-token attention cache: why sampling needs it, what it costs (~2 MB/token at 52B), when to skip it
- [[gpu-memory-capacity]] — weights vs kv cache vs activations; how leftover memory sets max concurrency
- [[inference-batching]] — tokens in flight as the batch axis; 208-token crossover; batch size trades latency, throughput, cost
- [[model-parallelism]] — tensor vs pipeline parallelism; 4 comms/layer, the 1040 flops-per-byte test, and the latency floor
- [[inference-latency-model]] — the two-regime equations (memory-bound / flops-bound) for per-token latency, and why they land within 10–35%

## Entities

- [[nvidia-a100]] — Ampere datacenter GPU; spec table, SKU variants (PCIe/SXM, 40/80 GB), NVLink 300 GB/s
- [[nvidia-smi]] — NVIDIA's GPU status CLI; DRAM usage, GPU-Util, health triage

## Syntheses

- [[2026-11-12-sizing-a-gpu-inference-deployment]] — weights → GPUs, leftover memory → concurrency, concurrency → cost; worked 13B/52B example and what it implies for quotas, scheduling, metering
