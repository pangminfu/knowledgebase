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

## [2026-11-23] ingest | Running Multiple Instances of Your App (Kubernetes Basics, Module 5)
- Saved raw: raw/2026-11-23-k8s-running-multiple-instances-of-your-app.md; images raw/assets/k8s-scale-1.svg, raw/assets/k8s-scale-2.svg
- Created: [[2026-11-23-k8s-running-multiple-instances-of-your-app]], [[kubernetes-scaling]], [[kubernetes-replicaset]]
- Updated: [[kubernetes-deployment]], [[kubernetes-service]], [[kubernetes-scheduling]], [[kubernetes-labels-and-selectors]], [[kubernetes]], [[kubectl]], [[control-plane]], [[inference-serving]], [[2026-11-22-k8s-using-a-service-to-expose-your-app]] (question cross-link), [[overview]], [[index]]
- Emphasis (user): desired state and reconcile loop, GPU scaling implications, Service load balancing and endpoints, kubectl commands and status columns

## [2026-11-24] schema | Multi-area knowledge base
- AGENTS.md Topic: single subject replaced by an Areas table (scope, level, tags per area). GPU cloud stays the primary area with its four domain tags; new areas `software-delivery` and `ai-agents`, both Beginner
- Tag rule: the Areas table is the complete tag vocabulary; new areas are proposed to the user and logged as schema changes
- `overview.md` becomes a map of areas followed by per-area synthesis; ingest step 5 and lint checks (knowledge gaps per area, thin areas) reworded to match
- Existing pages keep their domain tags; no retagging

## [2026-11-24] ingest | The AI-native SDLC playbook (Anthropic)
- Saved raw: raw/2026-11-24-ai-native-sdlc-playbook.md; images raw/assets/sdlc-bottleneck-shift.png, raw/assets/sdlc-line-vs-loop.png, raw/assets/sdlc-play-dependencies.png, raw/assets/sdlc-claude-tag-incident.png
- First source in the new `software-delivery` and `ai-agents` areas
- Created: [[2026-11-24-ai-native-sdlc-playbook]], [[software-development-lifecycle]], [[artifact-chain]], [[agent-guardrails]], [[deployment-gates]], [[control-bands]], [[evals]], [[claude-code]]
- Updated: [[overview]] (restructured: area map, GPU cloud sections under one heading, new software delivery / AI agents section), [[index]], [[kubernetes-deployment]], [[kubernetes-scaling]], [[control-plane]], [[gpu-utilization-metrics]] (cross-area links)
- Emphasis (user): all four equally: deployment gates and rollback, control bands, advisory vs enforced controls, artifact chain as audit trail; beginner level

## [2026-11-24] lint | Full wiki fact-check against raw; 50+ fixes, no structural issues
- Mechanical: all 52 pages in index, no orphans, no broken wikilinks. Fixed: [[operator-fusion]] had empty `tags` (now `metering-billing`); unescaped `|` in table wikilinks in [[inference-serving]], [[training-vs-inference]]
- Source error: kipply's 56 GB ÷ 2 MB is ≈28,000 tokens, not the printed 23,000; "batch 4 × 2048" belongs to the 3-GPU case. Corrected with a note in [[gpu-memory-capacity]], [[inference-batching]], [[overview]], [[2026-11-12-transformer-inference-arithmetic]], [[2026-11-12-sizing-a-gpu-inference-deployment]]
- Concept fix: the 208 threshold counts tokens per forward pass (batch size at decode), not batch × context length. Fixed in [[inference-batching]], [[memory-bandwidth]], [[per-token-pricing]], [[overview]], the source page, [[index]]
- Misstated numbers: "10–35%" prediction gap → naive estimates 24–38% under measured ([[inference-latency-model]], [[gpu-performance-regimes]], synthesis, source page); 43%/5% memory-bound share is *after* fusion ([[operator-fusion]]); 0.7 ms kv store is for the context pass ([[kv-cache]]); loaded-API batch advice; 69 ms single-GPU 52B marked hypothetical; synthesis step 4 misused the 1040 ratio; autoscaling row contradicted [[gpu-utilization-metrics]]
- Citation scope: split general knowledge from sourced claims in [[kubernetes-deployment]], [[control-plane]], [[inference-serving]], [[inference-frameworks]], [[fine-tuning]], [[flops]], [[nvidia-a100]], [[nvidia-smi]], [[inference-optimization]], k8s source pages (`logs` was never demonstrated)
- Contradictions resolved: "all GPUs on a node" ([[model-parallelism]]) vs 4-of-8 Pods ([[kubernetes-pod]], [[kubernetes-node]]); rolling updates "need" >1 replica ([[kubernetes-deployment]], [[kubernetes-scaling]], [[overview]])
- Stale general knowledge: Endpoints API deprecated (v1.33), kube-proxy nftables mode, per-connection balancing mode ([[kubernetes-service]]); namespaced vs cluster-scoped objects ([[kubernetes-labels-and-selectors]]); DORA now five metrics ([[deployment-gates]], playbook source page); Claude Code `auto` vs `acceptEdits` modes ([[claude-code]])
- SDLC source misreads: plan mode interviews the engineer, not the reverse; invented "spec→plan time" metric replaced with the playbook's plan-approval→merged-PR ([[artifact-chain]]); "passing checks" on branch protection marked general knowledge ([[agent-guardrails]])
- Beginner glosses added: readiness/liveness probes, layer 4, kube-proxy, MIG/time-slicing, OCI, admission policies, NVLink domain, MCP, OpenTelemetry, MDM, `PreToolUse`
- Open: no page for transformer basics (`d_model`, heads, attention), quotas/multi-tenancy, or metering pipelines though each is mentioned on 15+ pages; software-delivery and ai-agents areas rest on one vendor source
