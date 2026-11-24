---
type: entity
tags: [lifecycle]
sources: [2026-11-22-k8s-viewing-pods-and-nodes]
created: 2026-11-22
updated: 2026-11-24
---

# kubelet

The **kubelet** is the [[kubernetes]] agent process that runs on every [[kubernetes-node|Node]]. It takes orders from the [[control-plane]] ("run this Pod here") and makes them happen on its machine by telling the container runtime what to start and stop. Analogy: the floor manager in a hotel who takes the front desk's room assignments and actually checks guests in and out. (src for role: [[2026-11-22-k8s-viewing-pods-and-nodes]]; analogy: general knowledge)

## Details

- Every Node runs a kubelet. It is responsible for communication between the control plane and the Node, and manages the [[kubernetes-pod|Pods]] and containers running on the machine. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- It does not run containers itself; it delegates to a container runtime such as [[containerd]] over the Container Runtime Interface (CRI), a gRPC API. (src for the kubelet/runtime split: [[2026-11-22-k8s-viewing-pods-and-nodes]]; CRI: general knowledge; verified: [Container Runtime Interface](https://kubernetes.io/docs/concepts/containers/cri/))
- **Heartbeats and status:** it renews the Node's Lease object every 10 s by default and reports the Node's status (conditions, capacity, allocatable resources) to the API server. (general knowledge; verified: [Leases](https://kubernetes.io/docs/concepts/architecture/leases/), [Node Status](https://kubernetes.io/docs/reference/node/node-status/))
- **Restarts:** when a container exits, the kubelet restarts it in place according to the Pod's restart policy. (general knowledge; verified: [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/))
- **Probes:** it runs containers' health checks (general knowledge; verified: [Liveness, Readiness, and Startup Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)):
  - *liveness* (is it still alive? if not, the kubelet restarts the container);
  - *readiness* (can it take traffic yet? if not, the Pod is removed from [[kubernetes-service|Service]] endpoints, but not restarted);
  - *startup* (has a slow app finished starting? liveness and readiness checks wait until it succeeds).

## Why it matters for a GPU cloud

- A GPU device plugin registers with the kubelet over a gRPC `Registration` service. The kubelet then reports the Node's GPU count in the Node status, where the scheduler sees it ([[kubernetes-scheduling]]), and when a Pod lands, the kubelet picks specific free GPUs for the container and calls the plugin's `Allocate` to set them up. (general knowledge; verified: [Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/))
- Missing kubelet heartbeats are how the control plane decides a Node is gone, which leads to eviction and Pod replacement elsewhere ([[kubernetes-node]]). (general knowledge)
- A startup probe gives an inference Pod time to load model weights without being killed by its liveness probe, and a readiness probe keeps it out of the load balancer until it can serve ([[inference-serving]]). (general knowledge)

## Related

- [[kubernetes-node]]: where it runs.
- [[containerd]]: the runtime it drives.
- [[control-plane]]: who it reports to.
- [[kubernetes-pod]]: what it starts, restarts, and probes.
