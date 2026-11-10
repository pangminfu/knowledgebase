# Index

Catalog of every page in the wiki. One line per page: link — summary.

## Overview

- [[overview]] — big picture: GPU performance basics and their control-plane consequences; domain coverage table; next sources

## Sources

- [[2026-11-10-deep-learning-brrrr-first-principles]] — Horace He (2022): compute vs memory-bandwidth vs overhead regimes on GPUs; why spec-sheet TFLOPS and GPU-Util mislead

## Concepts

- [[gpu-performance-regimes]] — compute-, memory-, overhead-bound: how to diagnose each and what fixes it
- [[flops]] — floating-point ops/sec; peak vs achieved; Tensor Cores; why matmul dominates
- [[memory-bandwidth]] — DRAM↔SRAM data movement, memory-bound ops, compute intensity
- [[operator-fusion]] — merging ops into one kernel to skip memory round-trips
- [[framework-overhead]] — Python/framework/kernel-launch time that idles the GPU; how to detect it
- [[gpu-utilization-metrics]] — GPU-Util vs % of peak FLOPS vs DCGM; implications for metering

## Entities

- [[nvidia-a100]] — Ampere datacenter GPU; spec table, SKU variants (PCIe/SXM, 40/80 GB)
- [[nvidia-smi]] — NVIDIA's GPU status CLI; DRAM usage, GPU-Util, health triage

## Syntheses
