---
type: concept
tags: [provisioning, tenancy-quotas, lifecycle, metering-billing]
sources: [2026-11-21-what-is-llm-inference, 2026-11-12-transformer-inference-arithmetic, 2026-11-22-training-vs-inference, 2026-11-22-k8s-using-a-service-to-expose-your-app, 2026-11-23-k8s-running-multiple-instances-of-your-app]
created: 2026-11-21
updated: 2026-11-24
---

# Inference serving

**Serving** is everything required to turn [[llm-inference]] (the computation) into a reliable production service that applications can call. (src: [[2026-11-21-what-is-llm-inference]]) Analogy: inference is the kitchen cooking one dish. Serving is the restaurant around it: taking orders, seating guests, deciding which orders cook together, adding cooks at rush hour, and handing out the bill. For a GPU cloud, most of the serving list is control-plane work. (general knowledge)

## Details

### The serving responsibilities (the handbook's list)

API surface (HTTP/gRPC), request queuing and scheduling, batching across concurrent requests, load balancing across replicas, autoscaling, model loading and version management, health checks, observability, resource allocation. (src: [[2026-11-21-what-is-llm-inference]])

A second handbook page defines model serving as the production process of making a trained model available to applications: packaging it, loading it onto suitable hardware, exposing it through an API, monitoring it, and keeping **latency, reliability, security, and cost** under control. This adds packaging and security to the list above. (src: [[2026-11-22-training-vs-inference]])

A **replica** is one running copy of the model on its own GPU(s). Serving at scale means many replicas behind a load balancer. (general knowledge) On Kubernetes, a replica is typically one [[kubernetes-pod|Pod]], a dedicated endpoint is a [[kubernetes-deployment|Deployment]] of such Pods, and the endpoint's stable address is a [[kubernetes-service|Service]]. (general knowledge) A Service is Kubernetes' built-in way to load-balance across interchangeable replicas and give clients one address while replicas are lost and recreated. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]]) Its default proxy (kube-proxy) picks a backend Pod per connection, not per request, so it ignores request length and cache contents, and one long-lived HTTP/2 or gRPC connection sticks to one replica; LLM platforms add request-level routing on top. (general knowledge; verified: [Kubernetes docs: Virtual IPs and Service Proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/)) Adding or removing replicas is a change to the Deployment's replica count, and the Service sends traffic only to available Pods, tracked through its endpoints ([[kubernetes-scaling]]). (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]])

### Mapped to the four control-plane domains

The handbook gives the list; the mapping below is this wiki's (general knowledge). Some items are usually handled *inside* the [[inference-frameworks|inference framework]] on a single replica, and others by the *platform* across replicas and tenants.

| Responsibility | Domain | Usually inside the framework or in the platform? | What the control plane has to decide |
| --- | --- | --- | --- |
| API surface (HTTP/gRPC) | Provisioning APIs | Both: the framework exposes an endpoint, the platform adds auth and routing | Create/delete endpoint APIs, model + SKU choice, API keys |
| Resource allocation | Provisioning; tenancy-quotas | Platform | How many GPUs of which SKU per replica. Sizing from [[gpu-memory-capacity]] and [[model-parallelism]] |
| Model loading and version management | Lifecycle | Platform orchestrates, framework loads | Where weights are stored, cold-start time, rolling upgrades between model versions |
| Health checks | Lifecycle | Both | Restarting failed replicas, draining bad GPUs/nodes ([[nvidia-smi]]) |
| Autoscaling | Lifecycle; metering-billing | Platform | Scale on which signal (queue depth, tokens/sec, latency), min/max replicas, who pays for idle replicas. Mechanism and GPU pitfalls: [[kubernetes-scaling]] |
| Load balancing across replicas | Provisioning; lifecycle | Platform | Routing policy. A Kubernetes [[kubernetes-service\|Service]] gives basic per-connection balancing; cache-aware routing matters once replicas hold different prefixes ([[inference-optimization]]) |
| Request queuing and scheduling | Tenancy-quotas | Both: framework schedules within a replica, platform queues across tenants | Per-tenant rate limits, priority, fairness, what happens when a queue is full |
| Batching across concurrent requests | Tenancy-quotas; metering-billing | Framework | Tenants share a batch, so isolation and per-tenant accounting are required ([[inference-batching]]) |
| Observability | Metering-billing; lifecycle | Both | Per-tenant token counts for billing, latency/throughput SLOs, GPU metrics ([[gpu-utilization-metrics]]) |

### Serverless vs self-hosted

- **Serverless API:** the provider runs all of the above; the customer sends prompts and pays per token. (src: [[2026-11-21-what-is-llm-inference]])
- **Self-hosted / hybrid:** the customer (or a GPU cloud on their behalf) runs the serving stack. Teams move here when they hit limits on cost, latency, customization, or compliance. (src: [[2026-11-21-what-is-llm-inference]])
- A GPU cloud can sell at either level: raw GPUs (customer runs serving), managed endpoints (platform runs serving on dedicated GPUs), or a serverless per-token API (platform runs serving on shared GPUs). Each level moves more of the table above into the provider's control plane. (general knowledge) Pricing consequences: [[per-token-pricing]].

## Related

- [[llm-inference]]: the computation being served.
- [[inference-frameworks]]: engines that do both inference and the per-replica part of serving.
- [[inference-batching]]: the main lever for serving cost.
- [[per-token-pricing]]: how serving is billed.
- [[2026-11-12-sizing-a-gpu-inference-deployment]]: sizing arithmetic for the "resource allocation" row.
- [[training-vs-inference]]: serving is the "operate" half of the model lifecycle, and the always-on workload the control plane runs.
- [[kubernetes-pod]]: how a replica is packaged and placed on a GPU Node.
- [[kubernetes-service]]: the stable address and basic load balancer in front of replica Pods.
- [[kubernetes-scaling]]: how the replica count changes, and what limits it on GPUs.
- [[control-plane]]: the platform side of the framework/platform split, defined.

## Open questions

- Where exactly do real platforms (e.g. KServe, NVIDIA Triton/Dynamo, managed endpoints at GPU clouds) draw the framework/platform boundary? Needs sources.
- How do multi-tenant shared replicas isolate tenants (noisy neighbors, data separation in a shared batch and shared cache)?
