---
type: concept
tags: [lifecycle, tenancy-quotas, provisioning]
sources: [2026-11-22-k8s-viewing-pods-and-nodes, 2026-11-23-k8s-running-multiple-instances-of-your-app]
created: 2026-11-22
updated: 2026-11-23
---

# Scheduling (Kubernetes)

**Scheduling** is deciding which [[kubernetes-node|Node]] each new [[kubernetes-pod|Pod]] should run on. In [[kubernetes]] the [[control-plane]] does this automatically, based on how much free capacity each Node has. Analogy: a parking attendant who knows which spots are free and how big each car is, and sends each arriving car to a spot it fits in. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]]; analogy: general knowledge)

## Details

- The control plane schedules Pods across the Nodes in a cluster, taking into account the available resources on each Node. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- All containers of a Pod are **co-scheduled**: they land on the same Node together. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- Once scheduled, a Pod stays on its Node until it terminates or is deleted. If the Node fails, identical Pods are scheduled on other Nodes. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- [[kubernetes-scaling|Scaling out]] a Deployment creates new Pods that are scheduled to Nodes with available resources. In the tutorial, 4 replicas get 4 different Pod IPs (`kubectl get pods -o wide` shows the Node of each). (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]])
- How it works (general knowledge):
  - Each Pod declares **resource requests** (e.g. 4 CPUs, 32 GiB memory, 1 GPU). The scheduler adds up what is already requested on each Node and compares it with the Node's capacity.
  - **Filter, then score:** first drop Nodes that cannot fit the Pod or that the Pod is not allowed on, then rank the rest and pick the best.
  - If no Node fits, the Pod stays **Pending** until capacity frees up or a Node is added.
  - **Taints and tolerations** keep ordinary Pods off special Nodes, e.g. so CPU-only Pods do not waste a GPU server.

## GPU cloud bridge

General knowledge.

- **GPUs are scheduled as countable "extended resources".** A device plugin on each GPU Node advertises e.g. `nvidia.com/gpu: 8`; a Pod requests `nvidia.com/gpu: 4`; the scheduler only considers Nodes with 4 unallocated GPUs. GPUs are not overcommitted: requested means reserved, even if idle ([[gpu-utilization-metrics]]).
- **Single-Node multi-GPU is easy; multi-Node is not.** A Pod asking for 4 GPUs gets all 4 on one Node or stays Pending, so a single-Node tensor-parallel replica is naturally all-or-nothing. A job spanning many Nodes is many Pods, and the default scheduler places them one by one. It can start half of them and leave the rest Pending, holding GPUs that do nothing. **Gang scheduling** (all Pods or none) needs add-ons such as Kueue or Volcano. See [[model-parallelism]] and [[training-vs-inference]].
- **Fragmentation:** eight Nodes each with 1 free GPU cannot host one 8-GPU Pod. Packing small Pods tightly keeps whole Nodes free for big ones. This is a capacity and pricing problem, not only a technical one.
- **Quotas sit in front of scheduling.** The scheduler only checks whether a Pod fits a Node, not whether a tenant is allowed that many GPUs. Per-tenant limits come from a separate admission step (Kubernetes ResourceQuota, or the cloud's own quota system).
- **Scale-out is where Pending shows up.** Scaling an inference Deployment from 2 to 4 replicas needs free GPUs on Nodes for 2 more Pods. If there are none, `READY` stays at `2/4` until Nodes free up or are added ([[kubernetes-scaling]]).

## Related

- [[kubernetes-pod]]: the unit being scheduled.
- [[kubernetes-node]]: where Pods are placed; its capacity is the input.
- [[kubernetes-scaling]]: the main source of new Pods to place.
- [[control-plane]]: the scheduler is one of its components.
- [[model-parallelism]]: why some replicas need several GPUs on one Node.
- [[2026-11-12-sizing-a-gpu-inference-deployment]]: where the GPU count per replica comes from.

## Open questions

- How do GPU clouds handle topology (which 4 of 8 GPUs, NVLink and network locality) when placing Pods?
- How do Kueue / Volcano gang scheduling and queueing work, and which providers use them?
