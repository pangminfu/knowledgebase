---
type: overview
tags: [provisioning, tenancy-quotas, lifecycle, metering-billing, software-delivery, ai-agents]
sources: [2026-11-10-deep-learning-brrrr-first-principles, 2026-11-12-transformer-inference-arithmetic, 2026-11-21-what-is-llm-inference, 2026-11-22-training-vs-inference, 2026-11-22-k8s-viewing-pods-and-nodes, 2026-11-22-k8s-using-a-service-to-expose-your-app, 2026-11-23-k8s-running-multiple-instances-of-your-app, 2026-11-24-ai-native-sdlc-playbook]
created: 2026-11-10
updated: 2026-11-24
---

# Overview

A lifelong knowledge base, organized into areas. Each area below has its own big-picture synthesis.

| Area | Scope | Sources | Entry points |
| --- | --- | --- | --- |
| [[#GPU cloud]] (primary) | How GPU clouds are built and run: provisioning APIs, tenancy and quotas, Kubernetes/bare-metal lifecycle, metering and billing | 7 | [[control-plane]], [[inference-serving]], [[kubernetes]] |
| Software delivery ([[#Software delivery and AI agents]]) | How software is planned, built, tested, shipped, maintained | 1 | [[software-development-lifecycle]], [[deployment-gates]] |
| AI agents ([[#Software delivery and AI agents]]) | How coding agents work and are controlled | 1 (shared with software delivery) | [[agent-guardrails]], [[claude-code]] |

Where areas meet (general knowledge): a GPU cloud's control plane is software shipped through an SDLC; rollbacks and gated rollouts land on Kubernetes Deployments ([[deployment-gates]], [[kubernetes-deployment]]); alerting on control bands depends on choosing metrics that mean what you think ([[control-bands]], [[gpu-utilization-metrics]]).

## GPU cloud

**Goal:** working knowledge of GPU cloud control plane engineering across four domains: provisioning APIs, tenancy and quotas, Kubernetes/bare-metal lifecycle, and metering and billing.

### The product: what a GPU actually does

A GPU's real throughput is limited by one of three things: compute, memory bandwidth, or software overhead ([[gpu-performance-regimes]]). Headline numbers like "312 TFLOPS" hold only under narrow conditions ([[flops]], [[nvidia-a100]]). (src: [[2026-11-10-deep-learning-brrrr-first-principles]])

Control-plane consequences (general knowledge unless cited):

- **Metering-billing:** "busy" is ambiguous. `nvidia-smi` GPU-Util measures time-busy, not capacity used (src: [[2026-11-10-deep-learning-brrrr-first-principles]]), so dashboards and idle-detection have to choose their metrics deliberately. See [[gpu-utilization-metrics]].
- **Provisioning:** SKUs differ in form factor, memory size, and bandwidth. Which one fits depends on the customer's regime. See [[memory-bandwidth]].
- **Lifecycle:** [[nvidia-smi]] is the first-line tool for node health and "slow GPU" triage.

### Serving the product: inference arithmetic

The second source turns the same physics into a sizing procedure for model serving, which is where most GPU-cloud demand now comes from. (src: [[2026-11-12-transformer-inference-arithmetic]])

The chain is four arithmetic steps, laid out end to end in [[2026-11-12-sizing-a-gpu-inference-deployment]]:

1. **Weights → GPUs.** 2 bytes per parameter at 16-bit; a 52B model is 104 GB, so it cannot fit on a 40 GB card ([[gpu-memory-capacity]]).
2. **Leftover memory → concurrency.** The [[kv-cache]] costs ~2 MB per token for that model, so the remaining 56 GB on four cards is ~28,000 cached tokens (the source prints 23,000, an arithmetic slip; see [[gpu-memory-capacity]]).
3. **Concurrency → cost.** Below ~208 tokens per forward pass (at decode: a batch of ~208 sequences) the GPU is memory-bandwidth-bound; above it, flops-bound and cheaper per token ([[inference-batching]]).
4. **GPUs → latency floor.** Tensor parallelism cuts compute by `1/N` but adds a fixed four-per-layer communication cost, so more GPUs stop helping past a point ([[model-parallelism]], [[inference-latency-model]]).

Control-plane consequences (general knowledge unless cited):

- **Quotas are memory limits.** Concurrency follows from `(VRAM − weights) ÷ (cache per token × context)`, not from a GPU count (src: [[2026-11-12-transformer-inference-arithmetic]]).
- **Multi-GPU replicas are gang-scheduled.** A model too large for one card needs all the GPUs of a node, wired with NVLink — a topology constraint, not a quantity constraint. (general knowledge)
- **Tokens/second is the inference-native metering unit**, and it is not fixed per SKU: it depends on the batch the provider can pack (src: [[2026-11-12-transformer-inference-arithmetic]]; product framing: general knowledge).
- **Memory-bound replicas look busy.** GPU-Util can read high while the GPU does a fraction of its peak flops ([[gpu-utilization-metrics]]).

### The layer the control plane owns: serving

The third source adds vocabulary for the layer above the GPU arithmetic. **Inference** is the computation: tokens in, next-token probabilities out, through [[prefill-and-decode]] ([[llm-inference]]). **Serving** is everything that turns it into a production service: API, queuing and scheduling, batching across requests, load balancing, autoscaling, model versioning, health checks, observability, resource allocation ([[inference-serving]]). Engines like vLLM, SGLang, TensorRT-LLM, and MAX do both on each replica ([[inference-frameworks]]). (src: [[2026-11-21-what-is-llm-inference]])

```mermaid
flowchart TB
  CP["Control plane (platform)<br/>endpoints · quotas · autoscaling · routing · metering"] --> R1["Replica: inference framework<br/>batching · KV cache · streaming"]
  CP --> R2["Replica ..."]
  R1 --> G["GPU(s): prefill + decode<br/>bound by memory, bandwidth, flops"]
```

Control-plane consequences (general knowledge unless cited):

- **The serving list is the control plane's scope.** Each serving responsibility maps to one of the four domains; the mapping table is in [[inference-serving]].
- **Two products, two meters.** Serverless APIs bill per token and carry the utilization risk; self-hosted customers rent GPU-hours and carry it themselves (src for per-token serverless: [[2026-11-21-what-is-llm-inference]]). See [[per-token-pricing]].
- **Optimizations move capacity.** Continuous batching, quantization, and prefix caching raise tokens per GPU, so GPU-denominated quotas and token-denominated prices drift apart. See [[inference-optimization]] for the reading roadmap.

### Two workloads: training vs inference

A model's life has two phases. **Training** (build) runs once or periodically: it is a long job on multi-node clusters and is mostly a one-time cost. **Inference** (operate) runs on every request, in milliseconds to seconds, and costs money for as long as there is traffic, so it often becomes the larger expense over time. [[fine-tuning]] changes weights, so it counts as training. See [[training-vs-inference]]. (src: [[2026-11-22-training-vs-inference]])

Control-plane consequences (general knowledge):

- **Two job shapes.** Training is an all-or-nothing batch job: a gang-scheduled cluster, checkpoints, and reserved GPU-hours. Inference is an always-on service: autoscaled replicas, rolling upgrades, and per-token or endpoint billing. Every domain needs a version of each; the per-domain table is in [[training-vs-inference]].
- **The handoff is a model artifact.** Training or fine-tuning writes weights, and serving loads them. Storage, versioning, and cold start sit at that seam.

### Where workloads run: Kubernetes Pods and Nodes

The fifth source starts the lifecycle domain. In [[kubernetes]], a [[kubernetes-node|Node]] is a worker machine, virtual or physical, and a [[kubernetes-pod|Pod]] is the smallest unit it runs: one or more [[container|containers]] sharing an IP and storage, always on one Node. The [[control-plane]] schedules Pods onto Nodes by free resources ([[kubernetes-scheduling]]). Each Node runs a [[kubelet]] and a container runtime ([[containerd]]). When a Node fails, its Pods are not moved: identical replacements are scheduled elsewhere. A [[kubernetes-deployment|Deployment]] creates the Pods in the first place. [[kubectl]] is the tool for inspecting all of it. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])

Control-plane consequences (general knowledge):

- **"Control plane" has two levels.** Kubernetes has its own control plane (API server, scheduler, controllers). A GPU cloud's control plane sits above it and translates "create an endpoint for tenant A" into Kubernetes objects, checks quotas, and meters the result ([[control-plane]]).
- **Map the earlier pages onto Kubernetes objects.** A GPU Node is usually an 8-GPU bare-metal server. An inference replica is one Pod requesting GPUs, and an endpoint is a Deployment of such Pods. A Node failure costs a cold start while a replacement Pod reloads weights.
- **GPUs are scheduled as whole, reserved units.** A requested GPU is held even when idle, so GPU-hour billing and capacity follow Pod lifetimes, not utilization ([[gpu-utilization-metrics]]). Multi-Node jobs need gang scheduling beyond the default scheduler ([[kubernetes-scheduling]]).

### Reaching workloads: Services and labels

Pods get new IPs whenever they are replaced, and Pod IPs are not reachable from outside the cluster. A [[kubernetes-service|Service]] gives a set of Pods one stable address, load-balances across them, and lets clients discover them. Its `type` sets how far it reaches: ClusterIP (in-cluster) ⊂ NodePort (a port on every Node) ⊂ LoadBalancer (a cloud load balancer); ExternalName is only a DNS alias. A Service picks its Pods with a selector over key/value [[kubernetes-labels-and-selectors|labels]], Kubernetes' grouping primitive. Deleting a Service removes the route in, not the Pods. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])

Control-plane consequences (general knowledge):

- **An inference endpoint = Deployment + Service.** Replicas churn underneath; the customer's address does not. Readiness probes keep a Service from routing to a replica that is still loading weights.
- **Per-connection balancing is not enough for LLMs.** Long, streamed requests and per-replica [[kv-cache]] contents call for request-level, cache-aware routing above the Service ([[inference-serving]]).
- **Labels are the cross-domain tagging tool.** Node labels select GPU SKUs (provisioning), tenant labels group objects (tenancy), version labels drive rollouts (lifecycle), and cost tools attribute GPU-hours by label (metering). Because labels are mutable, billing-relevant ones must be set and guarded by the platform.

### Sizing the fleet: scaling replicas

[[kubernetes-scaling|Scaling]] means changing one number: a Deployment's replica count, its desired state. Its [[kubernetes-replicaset|ReplicaSet]] compares DESIRED with CURRENT and creates or terminates Pods to close the gap. New Pods are scheduled to Nodes with free resources, and scaling to zero terminates all Pods. The Service tracks running Pods through endpoints, so traffic spreads across new replicas automatically and the address never changes. Multiple replicas also make zero-downtime rolling updates easier. Autoscaling exists; it writes the same number automatically. (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]])

Control-plane consequences (general knowledge):

- **The replica count is the knob every domain touches.** Provisioning sets min/max replicas, quotas cap them (max replicas × GPUs per replica), autoscalers move them, and GPU-hour metering integrates them over time.
- **On GPUs, scale-out is not instant and not guaranteed.** New replicas stay Pending without whole free GPUs ([[kubernetes-scheduling]]), then pay a cold start loading weights. LLM autoscalers scale early and on queue/KV-cache signals, not CPU or GPU-Util ([[gpu-utilization-metrics]]).
- **Scale-to-zero is a pricing decision.** It saves idle GPU-hours at the cost of a cold start. Someone pays for minimum replicas: the customer on a dedicated endpoint, the provider on a serverless API ([[per-token-pricing]]).
- **Scale-down must drain.** Long streamed responses need a longer termination grace period than the 30 s default, or they are cut off mid-stream.

### Coverage so far

| Domain | Status |
| --- | --- |
| Provisioning APIs | GPU/SKU background, inference sizing rules, the serving responsibilities an endpoint API must cover, and training-cluster vs endpoint provisioning shapes (unsourced mapping); no actual API sources yet |
| Tenancy and quotas | Concurrency-as-memory from inference arithmetic; serving-side queuing/rate limits and training capacity blocks identified but unsourced |
| Kubernetes and bare-metal lifecycle | Kubernetes basics sourced (Pods, Nodes, kubelet, runtime, scheduling by free resources, replacement on Node failure, Services and Service types, labels/selectors, manual scaling via Deployment/ReplicaSet, endpoints, kubectl triage); autoscaling (HPA/cluster autoscaler), rolling updates, GPU device plugins, gang scheduling, Node health/drain, readiness probes, graceful termination, and bare-metal provisioning still unsourced |
| Metering and billing | Utilization metrics, tokens-vs-GPU-hours reasoning, per-token vs GPU-hour products, inference as the recurring cost; no billing-pipeline sources |

### Suggested next sources

- A provider's GPU instance docs and pricing page (e.g. AWS EC2 P-series, GCP A3, CoreWeave): provisioning, billing granularity, and how they present concurrency.
- Modular LLM Inference Handbook follow-on chapters: continuous batching, prefix caching, quantization, routing, serverless vs self-hosted (ordered list in [[inference-optimization]]).
- vLLM / TensorRT-LLM documentation on continuous batching and paged attention: the scheduling half of serving, from the engine authors.
- Kubernetes Basics Module 2 (Deployments) and Module 6 (rolling updates); Kubernetes autoscaling docs (HPA, cluster autoscaler) and an LLM autoscaler (KServe or KEDA); Kubernetes docs on device plugins, the NVIDIA GPU Operator, and gang scheduling (Kueue/Volcano): lifecycle.
- Gateway API Inference Extension / llm-d / KServe docs: how inference platforms route requests above a plain Service.
- Kubernetes ResourceQuota and cloud quota docs: tenancy and quotas.
- NVIDIA DCGM docs: real utilization metrics for metering.
- GPU cloud docs on training capacity (reserved clusters, capacity blocks, managed Slurm/Kubernetes): the training half of provisioning, quotas, and lifecycle.

## Software delivery and AI agents

**Goal:** understand how software gets from idea to production and how AI coding agents change that process, at a beginner level. The two areas share their first source.

### The bottleneck moves

The [[software-development-lifecycle]] (SDLC) runs six stages: plan, design, build, test, deploy, maintain. It was built when writing code was the slowest stage, and its controls assume a human does every step. Coding agents shrink build to hours, so the human-speed stages around it (requirements, review, release) become the constraint, and per-line human review stops scaling. (src: [[2026-11-24-ai-native-sdlc-playbook]])

![[raw/assets/sdlc-bottleneck-shift.png]]
Before agents, Build is the longest of six stages. After agents, it shrinks to a sliver and the other stages keep their length. (src: [[2026-11-24-ai-native-sdlc-playbook]])

### The AI-native answer: three mechanisms

```mermaid
flowchart LR
  I["intent.md"] --> S["spec.md"] --> P["plan.md"] --> D["diff + tests"] --> R["PR + review"]
  R -->|"code owner approves"| G{"production gate<br/>(human authorizes)"}
  G --> M["production"]
  M -->|"control band breached"| I
```

1. **[[artifact-chain]].** Each stage commits a file that starts the next. Git history becomes the trigger, the metric source, and the audit trail. (src: [[2026-11-24-ai-native-sdlc-playbook]])
2. **[[agent-guardrails]].** Advisory instructions (`CLAUDE.md`, skills) make good behaviour likely. Deterministic controls (hooks, permissions, sandbox, branch protection) make violations close to impossible. The agent that writes code never approves it, and the agent's own configuration is regression-tested with [[evals]]. (src: [[2026-11-24-ai-native-sdlc-playbook]])
3. **[[deployment-gates]].** The agent acts up to the production gate, never past it: free in development, prepare-only in production. Rollback is the most rehearsed path. (src: [[2026-11-24-ai-native-sdlc-playbook]])

The loop closes in maintenance: a deterministic script watches production against [[control-bands]], and the size of a breach decides whether the agent logs, diagnoses read-only, or proposes a fix through the gates. Its finding becomes the next `intent.md`. (src: [[2026-11-24-ai-native-sdlc-playbook]])

The single source so far is a vendor playbook built around [[claude-code]]. Its control design is general; its speed claims are the vendor's. (general knowledge)

### Coverage so far

| Topic | Status |
| --- | --- |
| SDLC fundamentals | Six stages and a beginner glossary, from an AI-native perspective only; no neutral source on traditional practice (agile, DevOps, DORA) |
| Agent controls | Advisory vs deterministic layers, managed settings, separation of duties; one vendor's implementation |
| Deploy and operations | Tiered autonomy, approval hooks, rollback, σ-band detection; no SRE or release-engineering source yet |
| Agent evaluation | Eval suite design for agent configuration; no general evals source |

### Suggested next sources

- DORA research / *Accelerate* (Forsgren, Humble, Kim): the delivery metrics the playbook measures against.
- Google SRE book, chapters on monitoring, alerting, and release engineering: control bands, SLOs, and rollouts from the operations side.
- An SPC / control-chart primer covering the Western Electric rules.
- Anthropic's companion articles: "How Anthropic secures its AI-native SDLC" and "The enterprise guide to securing coding agents".
- Claude Code docs on hooks, permissions, and sandboxing: the mechanics behind [[agent-guardrails]].
