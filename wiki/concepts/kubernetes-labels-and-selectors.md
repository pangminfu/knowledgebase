---
type: concept
tags: [tenancy-quotas, metering-billing, lifecycle, provisioning]
sources: [2026-11-22-k8s-using-a-service-to-expose-your-app]
created: 2026-11-22
updated: 2026-11-24
---

# Labels and selectors (Kubernetes)

A **label** is a key/value tag attached to a [[kubernetes]] object, such as `app=kubernetes-bootcamp` or `version=v1`. A **selector** is a query over labels ("every Pod with `app=A`"). Together they are Kubernetes' grouping primitive. Objects don't hold lists of the other objects they relate to; they find them by matching labels. Analogy: colored stickers on folders. You don't keep a list of "all red folders"; you just ask for every folder with a red sticker, including ones added after you last looked. (definition: src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]]; analogy: general knowledge)

## Details

- Labels are key/value pairs attached to objects. They are used to mark environments (development, test, production), embed version tags, and classify objects. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])
- Labels can be attached at creation time or later, and modified at any time. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])
- A [[kubernetes-service|Service]] finds its Pods by label selector. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]]) Membership is evaluated continuously, so relabeling a Pod adds it to or removes it from a Service's traffic. (general knowledge)
- A [[kubernetes-deployment|Deployment]] labels the Pods it creates automatically; in the tutorial the key is `app`. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]]) The Deployment and its ReplicaSet also use a selector to find "their" Pods. (general knowledge)
- With [[kubectl]], labels filter any listing: `kubectl get pods -l app=kubernetes-bootcamp`, `kubectl get services -l ...`, `kubectl delete service -l ...`. `kubectl label pods <pod> version=v1` adds a label. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])
- Two kinds of selector: **equality-based** (`app=A`, `tier!=cache`) and **set-based** (`env in (prod, staging)`, `!gpu`). (general knowledge)
- Syntax: a key is an optional prefix (a DNS subdomain, ≤253 characters, then `/`) plus a name of ≤63 characters; the value is ≤63 characters and may be empty. Names and values start and end with an alphanumeric character and may contain `-`, `_`, `.` in between. The `kubernetes.io/` and `k8s.io/` prefixes are reserved for Kubernetes. (general knowledge; verified: [Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#syntax-and-character-set))
- Labels are for identifying and selecting. Arbitrary non-identifying metadata (build info, owner contact, tool config) goes in **annotations**, which cannot be selected on. (general knowledge)
- Labels vs **namespaces**: a namespace is a hard partition with its own access control and quotas, and each *namespaced* object (Pod, Service, Deployment) lives in exactly one. Some kinds, such as Nodes, are cluster-scoped and belong to no namespace. Labels are soft, overlapping tags; an object can carry many. (general knowledge)

## GPU cloud bridge

All general knowledge; not from the source.

| Domain | How labels are used |
| --- | --- |
| Provisioning | Node labels describe hardware. NVIDIA GPU Feature Discovery adds labels like `nvidia.com/gpu.product=NVIDIA-A100-SXM4-80GB`, and a Pod's `nodeSelector` or node affinity uses them to land on the right SKU ([[kubernetes-scheduling]], [[nvidia-a100]]). |
| Tenancy and quotas | Tenant, project, and endpoint IDs go on every object the platform creates, so it can find "everything belonging to tenant X" for cleanup, policy, and support. Hard isolation still comes from namespaces and dedicated Nodes ([[kubernetes-node]]). |
| Lifecycle | `version` labels drive rollouts: a Service whose selector includes `version=v2` switches traffic in one edit (blue/green), or two Deployments share an `app` label for a canary split. Cordoned/repair state is often mirrored in Node labels. |
| Metering and billing | Cost tools (e.g. OpenCost) aggregate Pod GPU-hours by namespace and label. A missing or wrong tenant label means unattributed spend, so platforms set labels themselves and don't trust user-supplied ones for billing ([[gpu-utilization-metrics]], [[per-token-pricing]]). |

- **Labels are mutable, which is a risk.** Anyone with edit rights on a Pod can change its labels and so move it into or out of a Service, or change how it is billed. Platforms restrict who can set billing- and routing-relevant labels with admission policies (rules the API server checks before it accepts an object).

## Related

- [[kubernetes-service]]: the main consumer of label selectors in this source.
- [[kubernetes-deployment]]: auto-labels its Pods and selects them.
- [[kubernetes-replicaset]]: selects its Pods by label, including `pod-template-hash`.
- [[kubernetes-pod]], [[kubernetes-node]]: the objects most often labeled.
- [[kubectl]]: `-l` filtering and `kubectl label`.
- [[kubernetes]]: namespaces, the hard-partition counterpart.

## Open questions

- Which labels do real GPU clouds put on customer workloads, and how do they keep them tamper-proof for billing?
- How do ResourceQuota scopes and namespaces combine with labels for per-tenant GPU limits? (Needs a ResourceQuota source.)
