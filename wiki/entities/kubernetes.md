---
type: entity
tags: [lifecycle, provisioning, tenancy-quotas]
sources: [2026-11-22-k8s-viewing-pods-and-nodes, 2026-11-22-k8s-using-a-service-to-expose-your-app, 2026-11-23-k8s-running-multiple-instances-of-your-app]
created: 2026-11-22
updated: 2026-11-24
---

# Kubernetes

**Kubernetes** (often "K8s": the 8 stands for the eight letters between "K" and "s") is open-source software that runs [[container|containerized]] applications across a cluster of machines. You describe what you want running ("3 copies of this app, each with 1 GPU"), and Kubernetes places it on machines, restarts what crashes, and replaces what is lost. Google open-sourced it in 2014, and it is now maintained under the Cloud Native Computing Foundation (CNCF). (general knowledge; verified: [Overview](https://kubernetes.io/docs/concepts/overview/)) Many GPU clouds either run customer workloads on Kubernetes or offer managed Kubernetes clusters, which makes it core knowledge for the lifecycle domain. (general knowledge)

## Details

- **Versions:** the project ships about three minor releases a year (general knowledge) and supports the three most recent (as of 2026-11: v1.37, v1.36, v1.35) (general knowledge; verified: [Releases](https://kubernetes.io/releases/)). Many features move through alpha, beta, and stable across releases, so version-specific facts on this wiki name the version they apply to.
- **Core objects so far:** [[kubernetes-pod|Pod]] (the unit it runs and schedules), [[kubernetes-node|Node]] (a worker machine, virtual or physical), [[kubernetes-deployment|Deployment]] (creates and replaces Pods, through a [[kubernetes-replicaset|ReplicaSet]]), [[kubernetes-service|Service]] (a stable address and load balancer for a set of Pods). (src: [[2026-11-22-k8s-viewing-pods-and-nodes]], [[2026-11-22-k8s-using-a-service-to-expose-your-app]], [[2026-11-23-k8s-running-multiple-instances-of-your-app]])
- **Labels and selectors** are the grouping primitive: objects carry key/value [[kubernetes-labels-and-selectors|labels]], and other objects find them by label query instead of by name. Object manifests are written in YAML or JSON. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])
- **Two halves:** the [[control-plane]] manages Nodes and schedules Pods; every Node runs a [[kubelet]] and a container runtime such as [[containerd]]. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- **Interface:** everything is an API object on the control plane's API server, e.g. `/api/v1/namespaces/default/pods/<name>`. [[kubectl]] is the command-line client for it. (URL path: src: [[2026-11-22-k8s-viewing-pods-and-nodes]]; "everything is an API object": general knowledge)
- **Declarative model:** you store *desired state*; controllers continuously reconcile actual state toward it. This is why a lost Pod that belongs to a Deployment comes back without anyone asking. (general knowledge) [[kubernetes-scaling|Scaling]] shows it directly: you change a Deployment's replica count, and `kubectl get rs` reports the ReplicaSet's DESIRED (desired replicas) and CURRENT (running replicas) counts. (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]])
- **Namespaces** partition one cluster into named groups of objects (the tutorial's Pod lives in `default`). They are the basic unit for per-team or per-tenant access control and quotas. (namespace in URL: src: [[2026-11-22-k8s-viewing-pods-and-nodes]]; role: general knowledge)

## Why it matters for a GPU cloud

General knowledge.

- **Lifecycle:** Node health, draining, and replacement; Pod restarts; rolling upgrades of inference replicas.
- **Provisioning:** a cloud "create endpoint" or "create cluster" API ends up creating Kubernetes objects, e.g. a Deployment plus a Service for an inference endpoint.
- **Tenancy and quotas:** namespaces, ResourceQuota, and dedicated Node pools are the Kubernetes-level isolation tools; labels tag objects by tenant.
- **Metering:** Pod start/stop times and GPU requests are the raw data behind GPU-hour billing, attributed to tenants by namespace and label.
- **GPUs need add-ons:** the vendor's device plugin (for NVIDIA, usually installed by the GPU Operator). Multi-Node jobs need gang scheduling: built in only as a beta, off-by-default feature in v1.37, so add-ons such as Volcano or Kueue are common ([[kubernetes-scheduling]]). (verified: [Schedule GPUs](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/), [Gang Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/))

## Related

- [[control-plane]]: Kubernetes' control plane vs a GPU cloud's.
- [[kubectl]]: how you talk to it.
- [[kubernetes-service]], [[kubernetes-labels-and-selectors]]: how workloads are addressed and grouped.
- [[kubernetes-scaling]], [[kubernetes-replicaset]]: how the number of Pods is set and enforced.
- [[2026-11-22-k8s-viewing-pods-and-nodes]], [[2026-11-22-k8s-using-a-service-to-expose-your-app]], [[2026-11-23-k8s-running-multiple-instances-of-your-app]]: the tutorial modules this page is based on.
