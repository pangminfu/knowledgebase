---
type: entity
tags: [lifecycle, provisioning, tenancy-quotas]
sources: [2026-11-22-k8s-viewing-pods-and-nodes]
created: 2026-11-22
updated: 2026-11-22
---

# Kubernetes

**Kubernetes** (often "K8s") is open-source software that runs [[container|containerized]] applications across a cluster of machines. You describe what you want running ("3 copies of this app, each with 1 GPU"), and Kubernetes places it on machines, restarts what crashes, and replaces what is lost. It came out of Google in 2014 and is now maintained under the Cloud Native Computing Foundation. Most GPU clouds either run customer workloads on Kubernetes or offer managed Kubernetes clusters, which makes it core knowledge for the lifecycle domain. (general knowledge)

## Details

- **Core objects so far:** [[kubernetes-pod|Pod]] (the unit it runs and schedules), [[kubernetes-node|Node]] (a worker machine, virtual or physical), [[kubernetes-deployment|Deployment]] (creates and replaces Pods). (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- **Two halves:** the [[control-plane]] manages Nodes and schedules Pods; every Node runs a [[kubelet]] and a container runtime such as [[containerd]]. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- **Interface:** everything is an API object on the control plane's REST API, e.g. `/api/v1/namespaces/default/pods/<name>`. [[kubectl]] is the command-line client for it. (URL path: src: [[2026-11-22-k8s-viewing-pods-and-nodes]]; "everything is an API object": general knowledge)
- **Declarative model:** you store *desired state*; controllers continuously reconcile actual state toward it. This is why a lost Pod comes back without anyone asking. (general knowledge)
- **Namespaces** partition one cluster into named groups of objects (the tutorial's Pod lives in `default`). They are the basic unit for per-team or per-tenant access control and quotas. (namespace in URL: src; role: general knowledge)

## Why it matters for a GPU cloud

General knowledge.

- **Lifecycle:** Node health, draining, and replacement; Pod restarts; rolling upgrades of inference replicas.
- **Provisioning:** a cloud "create endpoint" or "create cluster" API ends up creating Kubernetes objects.
- **Tenancy and quotas:** namespaces, ResourceQuota, and dedicated Node pools are the Kubernetes-level isolation tools.
- **Metering:** Pod start/stop times and GPU requests are the raw data behind GPU-hour billing.
- GPUs need add-ons: the NVIDIA device plugin / GPU Operator, and a gang scheduler for multi-Node jobs ([[kubernetes-scheduling]]).

## Related

- [[control-plane]]: Kubernetes' control plane vs a GPU cloud's.
- [[kubectl]]: how you talk to it.
- [[2026-11-22-k8s-viewing-pods-and-nodes]]: the tutorial module this page is based on.
