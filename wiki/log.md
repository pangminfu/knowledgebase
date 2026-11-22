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

## [2026-11-22] ingest | Viewing Pods and Nodes (Kubernetes Basics, Module 3)
- Saved raw: `raw/2026-11-22-k8s-viewing-pods-and-nodes.md`; 2 diagrams as `raw/assets/k8s-explore-pods.svg`, `raw/assets/k8s-explore-nodes.svg`
- Emphasis (user): bridge to GPU cloud (marked general knowledge), beginner definitions and analogies, kubectl cheat sheet as its own page
- Created: [[2026-11-22-k8s-viewing-pods-and-nodes]], [[kubernetes-pod]], [[kubernetes-node]], [[kubernetes-deployment]], [[kubernetes-scheduling]], [[control-plane]], [[container]], [[kubernetes]], [[kubelet]], [[containerd]], [[kubectl]]
- Updated: [[inference-serving]], [[inference-frameworks]], [[model-parallelism]], [[training-vs-inference]], [[nvidia-smi]], [[overview]], [[index]]

## [2026-11-22] schema | Lifelong knowledge base; drop interview/career content
- AGENTS.md: subject/goal reframed from career switch to a lifelong knowledge base; `career` tag removed; new style rule "Knowledge, not exam prep"; source template gains optional `## Takeaways`
- Replaced "Interview talking points" with declarative "Takeaways": [[2026-11-10-deep-learning-brrrr-first-principles]], [[2026-11-12-transformer-inference-arithmetic]], [[2026-11-21-what-is-llm-inference]], [[2026-11-22-k8s-viewing-pods-and-nodes]], [[2026-11-22-training-vs-inference]]
- [[gpu-performance-regimes]]: talking points removed; compute-outgrowing-bandwidth trend moved into Details
- Reworded career framing: [[llm-inference]], [[control-plane]], [[kubernetes-node]], [[overview]] (Career coverage row removed), [[index]]
- `career` tag removed from frontmatter of 29 pages; [[operator-fusion]] now has no tags

## [2026-11-22] ingest | Using a Service to Expose Your App (Kubernetes Basics, Module 4)
- Saved raw: raw/2026-11-22-k8s-using-a-service-to-expose-your-app.md; image raw/assets/k8s-expose-labels.svg
- Created: [[2026-11-22-k8s-using-a-service-to-expose-your-app]], [[kubernetes-service]], [[kubernetes-labels-and-selectors]]
- Updated: [[kubernetes]], [[kubernetes-pod]], [[kubernetes-node]], [[kubernetes-deployment]], [[kubectl]], [[inference-serving]], [[2026-11-22-k8s-viewing-pods-and-nodes]] (question cross-link), [[overview]], [[index]]
- Emphasis (user): Service types, labels as a general primitive, inference-endpoint mapping, kubectl commands
