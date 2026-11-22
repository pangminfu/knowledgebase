---
type: concept
tags: [lifecycle, provisioning, tenancy-quotas]
sources: [2026-11-22-k8s-viewing-pods-and-nodes]
created: 2026-11-22
updated: 2026-11-22
---

# Node (Kubernetes)

A **Node** is one worker machine in a [[kubernetes]] cluster, either a virtual machine or a physical server, that runs [[kubernetes-pod|Pods]]. The [[control-plane]] manages every Node and decides which Pods go on which Node. Analogy: if the cluster is a hotel, Nodes are the floors and Pods are the guests; the front desk (control plane) assigns rooms based on what is free, and each floor has a manager ([[kubelet]]) who actually lets guests in. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])

![[raw/assets/k8s-explore-nodes.svg]]
One Node (hexagon) running four Pods of different sizes, each with its own IP. The strip at the bottom edge is the Node's own processes: kubelet and the container runtime (drawn as "Docker"; the page text now says containerd). (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])

## Details

- A Pod always runs on a Node. A Node can run many Pods. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- A Node is virtual or physical "depending on the cluster". (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- The control plane schedules Pods across Nodes automatically, taking each Node's available resources into account ([[kubernetes-scheduling]]). (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- Every Node runs at least two things (src: [[2026-11-22-k8s-viewing-pods-and-nodes]]):
  - **[[kubelet]]**: the link between the control plane and the Node; manages the Pods and containers on that machine.
  - **A container runtime** such as [[containerd]]: pulls images from a registry, unpacks them, and runs them ([[container]]).
- If a Node fails, its Pods are rescheduled as identical new Pods on other available Nodes. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- The control plane notices a failed Node because its kubelet stops sending heartbeats; after a timeout the Node is marked not ready and its Pods are evicted. (general knowledge)
- `kubectl describe node <name>` shows a Node's capacity, what is allocated, and recent events ([[kubectl]]). (src for `describe` on Nodes: [[2026-11-22-k8s-viewing-pods-and-nodes]]; fields: general knowledge)

## GPU cloud bridge

All general knowledge; not from the source.

- **A GPU Node is usually a bare-metal server** with 8 GPUs connected by NVLink (e.g. an 8× [[nvidia-a100]] box). GPU clouds often skip the VM layer so customers get full GPU and network performance.
- **Extra software on GPU Nodes:** the NVIDIA driver, the NVIDIA container toolkit (so [[containerd]] can hand GPUs to containers), and the **device plugin**, which tells kubelet "this Node has 8 `nvidia.com/gpu`", so the scheduler can count them. The NVIDIA GPU Operator installs all of this, plus a DCGM metrics exporter ([[gpu-utilization-metrics]]).
- **Node lifecycle is the bare-metal half of the job:** rack and image the server, install drivers, run health checks ([[nvidia-smi]], burn-in tests), join it to the cluster, and later *cordon* (stop new Pods), *drain* (evict existing Pods), repair, and return it. A single bad GPU can take a whole 8-GPU Node out of service.
- **Node shape decides what fits.** A model that needs 4 GPUs on one NVLink domain can only run on Nodes with 4 free, connected GPUs ([[model-parallelism]], [[gpu-memory-capacity]]).
- **Tenancy:** customers needing hard isolation often get whole dedicated Nodes, so no other tenant's Pods share their machine.

## Related

- [[kubernetes-pod]]: what runs on a Node.
- [[kubelet]], [[containerd]]: the two processes every Node runs.
- [[control-plane]]: what manages Nodes.
- [[kubernetes-scheduling]]: how Pods are placed on Nodes.
- [[nvidia-smi]]: first-line GPU health check on a Node.

## Open questions

- How does a GPU cloud take a new bare-metal server to a schedulable GPU Node (provisioning, imaging, driver install, burn-in), and how long does that take?
- What exact signals mark a GPU Node unhealthy (XID errors, ECC errors, NVLink faults), and who drains it?
