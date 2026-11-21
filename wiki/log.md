# Log

Append-only. Entry format: `## [YYYY-MM-DD] <ingest|query|lint|schema|init> | <title>`

## [2026-11-10] init | Wiki scaffolded

- Created AGENTS.md, wiki/index.md, wiki/log.md, wiki/overview.md
- Created folders: raw/, raw/assets/, wiki/sources/, wiki/concepts/, wiki/entities/, wiki/syntheses/

## [2026-11-10] schema | Topic set

- Subject: career switch to GPU cloud control plane engineer
- Focus: provisioning APIs, tenancy & quotas, Kubernetes & bare-metal lifecycle, metering & billing
- Level: beginner; ingest style: supervised
- Added domain tags: provisioning, tenancy-quotas, lifecycle, metering-billing, career

## [2026-11-10] schema | Image convention

- Added Images rule to AGENTS.md Conventions: embed meaningful source images with a text description; download URL images to raw/assets/

## [2026-11-10] ingest | Making Deep Learning Go Brrrr From First Principles
- Emphasis (user): control-plane angle, beginner fundamentals, interview talking points; lean page set
- Created: wiki/sources/2026-11-10-deep-learning-brrrr-first-principles.md
- Created: wiki/concepts/gpu-performance-regimes.md, flops.md, memory-bandwidth.md, operator-fusion.md, framework-overhead.md, gpu-utilization-metrics.md
- Created: wiki/entities/nvidia-a100.md, nvidia-smi.md
- Updated: wiki/overview.md, wiki/index.md

## [2026-11-12] ingest | Transformer Inference Arithmetic
- Saved source to `raw/2026-11-12-transformer-inference-arithmetic.md` (full text, 14 display equations) and 5 images as `raw/assets/tia-*.png`
- Created `wiki/sources/2026-11-12-transformer-inference-arithmetic.md`
- Created concepts: `kv-cache`, `gpu-memory-capacity`, `inference-batching`, `model-parallelism`, `inference-latency-model`
- Created synthesis: `2026-11-12-sizing-a-gpu-inference-deployment`
- Updated `overview`, `index`, `gpu-performance-regimes`, `flops`, `memory-bandwidth`, `operator-fusion`, `gpu-utilization-metrics`, `nvidia-a100`

## [2026-11-21] ingest | What is LLM inference? (Modular LLM Inference Handbook)
- Saved raw: `raw/2026-11-21-what-is-llm-inference.md` (no images; page has an interactive visualizer only)
- Emphasis (user): serving → control-plane domain mapping; per-token billing and serverless vs self-hosted; optimization reading roadmap; interview vocabulary. Frameworks kept on one page, not split into entities.
- Created: [[2026-11-21-what-is-llm-inference]], [[llm-inference]], [[prefill-and-decode]], [[inference-serving]], [[inference-frameworks]], [[inference-optimization]], [[per-token-pricing]]
- Updated: [[inference-batching]], [[kv-cache]], [[overview]], [[index]]

## [2026-11-22] ingest | Training vs. inference (Modular LLM Inference Handbook)
- Saved raw: `raw/2026-11-22-training-vs-inference.md` (no images; the interactive lifecycle visualizer is described in frontmatter)
- Emphasis (user): control-plane implications of training vs inference workloads; fine-tuning as a separate stub page
- Created: [[2026-11-22-training-vs-inference]], [[training-vs-inference]], [[fine-tuning]]
- Updated: [[llm-inference]], [[inference-serving]], [[inference-frameworks]] (added LMDeploy), [[per-token-pricing]], [[overview]], [[index]]
