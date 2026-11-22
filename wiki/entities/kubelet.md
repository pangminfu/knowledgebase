---
type: entity
tags: [lifecycle]
sources: [2026-11-22-k8s-viewing-pods-and-nodes]
created: 2026-11-22
updated: 2026-11-22
---

# kubelet

The **kubelet** is the [[kubernetes]] agent process that runs on every [[kubernetes-node|Node]]. It takes orders from the [[control-plane]] ("run this Pod here") and makes them happen on its machine by telling the container runtime what to start and stop. Analogy: the floor manager in a hotel who takes the front desk's room assignments and actually checks guests in and out. (src for role: [[2026-11-22-k8s-viewing-pods-and-nodes]]; analogy: general knowledge)

## Details

- Every Node runs a kubelet. It is responsible for communication between the control plane and the Node, and manages the [[kubernetes-pod|Pods]] and containers running on the machine. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- It does not run containers itself; it delegates to a container runtime such as [[containerd]]. (src for the two-part split: [[2026-11-22-k8s-viewing-pods-and-nodes]]; delegation detail: general knowledge)
- It reports the Node's status and capacity back to the control plane, sends heartbeats, and runs containers' health checks (liveness/readiness probes), restarting failed containers. (general knowledge)

## Why it matters for a GPU cloud

General knowledge.

- A GPU device plugin registers with the kubelet, which then advertises the Node's GPU count to the scheduler ([[kubernetes-scheduling]]) and assigns specific GPUs to each Pod.
- Missing kubelet heartbeats are how the control plane decides a Node is gone, which triggers Pod replacement elsewhere.
- Readiness probes keep an inference Pod out of the load balancer until it has finished loading model weights ([[inference-serving]]).

## Related

- [[kubernetes-node]]: where it runs.
- [[containerd]]: the runtime it drives.
- [[control-plane]]: who it reports to.
