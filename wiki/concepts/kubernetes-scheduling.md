---
type: concept
tags: [lifecycle, tenancy-quotas, provisioning]
sources: [2026-11-22-k8s-viewing-pods-and-nodes, 2026-11-23-k8s-running-multiple-instances-of-your-app]
created: 2026-11-22
updated: 2026-11-24
---

# Scheduling (Kubernetes)

**Scheduling** is deciding which [[kubernetes-node|Node]] each new [[kubernetes-pod|Pod]] should run on. In [[kubernetes]] the [[control-plane]] does this automatically, based on how much free capacity each Node has. Analogy: a parking attendant who knows which spots are free and how big each car is, and sends each arriving car to a spot it fits in. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]]; analogy: general knowledge)

## Details

### From the tutorials

- The control plane schedules Pods across the Nodes in a cluster, taking into account the available resources on each Node. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- All containers of a Pod are **co-scheduled**: they land on the same Node together. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- Once scheduled, a Pod stays on its Node until it terminates or is deleted. If the Node fails, identical Pods are scheduled on other Nodes. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]]) These are new Pods created by the Pod's controller; scheduling happens once per Pod and is never redone for an existing Pod. (general knowledge; verified: [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/))
- [[kubernetes-scaling|Scaling out]] a Deployment creates new Pods that are scheduled to Nodes with available resources. In the tutorial, scaling to 4 replicas gives 4 Pods with different IP addresses. (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]]) `kubectl get pods -o wide`, which the tutorial uses, also shows each Pod's Node. (general knowledge; verified: [kubectl get](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/))

### How the scheduler works

General knowledge; verified: [Kubernetes Scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/), [Node Status](https://kubernetes.io/docs/reference/node/node-status/).

- The **kube-scheduler** is a control-plane component that watches for Pods with no Node assigned yet and picks one for each.
- Each container declares **resource requests** (e.g. 4 CPUs, 32 GiB memory, 1 GPU). The scheduler adds up what is already requested on each Node and compares it with the Node's **allocatable** resources (capacity minus what is reserved for the operating system and Kubernetes' own daemons). It uses requests, not actual usage.
- **Filter, then score:** filtering drops Nodes where the Pod cannot run (not enough free resources, a taint it does not tolerate, a node selector it does not match); scoring ranks the remaining *feasible* Nodes and the Pod goes to the highest-scoring one.
- If no Node is feasible, the Pod stays unscheduled (phase **Pending**) until capacity frees up or a Node is added.
- **Taints and tolerations** keep ordinary Pods off special Nodes: a tainted Node only accepts Pods that tolerate the taint, e.g. so CPU-only Pods do not take up a GPU server. (verified: [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/))

## GPU cloud bridge

- **GPUs are scheduled as countable "extended resources".** A device plugin on each GPU Node advertises e.g. `nvidia.com/gpu: 8`; a container asks for `nvidia.com/gpu: 4` in its limits; the scheduler only considers Nodes with 4 unallocated GPUs. Extended resources are whole numbers and cannot be overcommitted: a requested GPU is reserved even if it sits idle ([[gpu-utilization-metrics]]). (general knowledge; verified: [Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/), [Schedule GPUs](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/))
- **Dynamic Resource Allocation (DRA)** is the newer, richer alternative to plain GPU counts: Pods reference *ResourceClaims* that can ask for devices by attributes (model, memory) and share them. DRA is stable since Kubernetes v1.35. (general knowledge; verified: [Dynamic Resource Allocation](https://kubernetes.io/docs/concepts/resource-management/dynamic-resource-allocation/))
- **Single-Node multi-GPU is easy; multi-Node is harder.** A Pod asking for 4 GPUs gets all 4 on one Node or stays Pending, so a single-Node tensor-parallel replica is naturally all-or-nothing. A job spanning many Nodes is many Pods, and by default the scheduler places them one by one: it can start half of them and leave the rest Pending, holding GPUs that do nothing. See [[model-parallelism]] and [[training-vs-inference]]. (general knowledge)
- **Gang scheduling** (place all Pods of a group, or at least `minCount` of them, or none) is now built into kube-scheduler, but only as a beta feature that is **disabled by default** in v1.37: it needs the `GenericWorkload` feature gate and the PodGroup API (`scheduling.k8s.io/v1beta1`). (general knowledge; verified: [Gang Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/)) Production clusters commonly use add-ons instead, such as Volcano (a batch scheduler with gang scheduling) or Kueue (job queueing with all-or-nothing admission). (general knowledge)
- **Fragmentation:** eight Nodes each with 1 free GPU cannot host one 8-GPU Pod. Packing small Pods tightly keeps whole Nodes free for big ones. This is a capacity and pricing problem, not only a technical one. (general knowledge)
- **Quotas sit in front of scheduling.** The scheduler only checks whether a Pod fits a Node, not whether a tenant may use that many GPUs. Per-tenant limits are enforced when the Pod is created, by admission checks such as a namespace's ResourceQuota (e.g. `requests.nvidia.com/gpu: 4`) or by the cloud's own quota system. (general knowledge; verified: [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/))
- **Scale-out is where Pending shows up.** Scaling an inference Deployment from 2 to 4 replicas needs free GPUs on Nodes for 2 more Pods. If there are none, the Deployment's `READY` column stays at `2/4` (ready/desired) until Nodes free up or are added ([[kubernetes-scaling]]). (READY = ready/desired: src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]]; scenario: general knowledge)

## Related

- [[kubernetes-pod]]: the unit being scheduled.
- [[kubernetes-node]]: where Pods are placed; its allocatable capacity is the input.
- [[kubernetes-scaling]]: the main source of new Pods to place.
- [[control-plane]]: the scheduler is one of its components.
- [[model-parallelism]]: why some replicas need several GPUs on one Node.
- [[2026-11-12-sizing-a-gpu-inference-deployment]]: where the GPU count per replica comes from.

## Open questions

- How do GPU clouds handle topology (which 4 of 8 GPUs, NVLink and network locality) when placing Pods?
- How do built-in gang scheduling, Kueue, and Volcano compare in practice, and which providers use which?
