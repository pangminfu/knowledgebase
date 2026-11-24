---
type: concept
tags: [lifecycle, provisioning, tenancy-quotas]
sources: [2026-11-22-k8s-viewing-pods-and-nodes, 2026-11-22-k8s-using-a-service-to-expose-your-app]
created: 2026-11-22
updated: 2026-11-24
---

# Node (Kubernetes)

A **Node** is one worker machine in a [[kubernetes]] cluster, either a virtual machine or a physical server, that runs [[kubernetes-pod|Pods]]. The [[control-plane]] manages every Node and decides which Pods go on which Node. Analogy: if the cluster is a hotel, Nodes are the floors and Pods are the guests; the front desk (control plane) assigns rooms based on what is free, and each floor has a manager ([[kubelet]]) who actually lets guests in. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]]; analogy: general knowledge)

![[raw/assets/k8s-explore-nodes.svg]]
One Node (hexagon) running four Pods of different sizes, each with its own IP. The strip along the bottom-left edge is the Node's own processes: kubelet and the container runtime (drawn as "Docker"; the page text now says containerd). (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])

## Details

### What runs on a Node

- A Pod always runs on a Node. A Node can run many Pods. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- A Node is virtual or physical "depending on the cluster". (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- The control plane schedules Pods across Nodes automatically, taking each Node's available resources into account ([[kubernetes-scheduling]]). (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- Every Node runs at least two things (src: [[2026-11-22-k8s-viewing-pods-and-nodes]]):
  - **[[kubelet]]**: the link between the control plane and the Node; manages the Pods and containers on that machine.
  - **A container runtime** such as [[containerd]]: pulls images from a registry, unpacks them, and runs them ([[container]]).
- Most clusters also run **kube-proxy** on each Node (officially optional), which maintains the network rules that make [[kubernetes-service|Services]] work. (general knowledge; verified: [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/))
- `kubectl describe node <name>` shows a Node's status: conditions (e.g. `Ready`), capacity and allocatable resources, the resources already requested by its Pods, and recent events ([[kubectl]]). (src for `describe` on Nodes: [[2026-11-22-k8s-viewing-pods-and-nodes]]; fields: general knowledge; verified: [Node Status](https://kubernetes.io/docs/reference/node/node-status/))
- Nodes carry [[kubernetes-labels-and-selectors|labels]] too, and Pods can use node selectors to pick Nodes by label, e.g. to land on the right GPU type. On NVIDIA GPU Nodes, GPU Feature Discovery adds labels such as `nvidia.com/gpu.product`. (general knowledge; verified: [Schedule GPUs](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/), [NVIDIA k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin))

### When a Node fails

- If a Node fails, identical Pods are scheduled on other available Nodes. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]]) Precisely: the old Pods are not moved; their controller creates new replacement Pods ([[kubernetes-pod]]).
- **Heartbeats:** the kubelet renews its Node's **Lease** object (in the `kube-node-lease` namespace) every 10 s by default, and separately updates the Node's `.status`. (general knowledge; verified: [Leases](https://kubernetes.io/docs/concepts/architecture/leases/), [Node Status](https://kubernetes.io/docs/reference/node/node-status/))
- **Detection:** the node controller (part of the control plane's controller manager) checks Nodes every 5 s. If a Node sends no heartbeat for the grace period (`--node-monitor-grace-period`, default 50 s), it sets the Node's `Ready` condition to `Unknown` and taints the Node `node.kubernetes.io/unreachable`, so nothing new is scheduled there. (general knowledge; verified: [Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/), [kube-controller-manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/))
- **Eviction:** Pods tolerate that taint for 300 s by default, so about 5 minutes later they are evicted and their controllers create replacements elsewhere. Evictions are rate-limited (by default at most one Node's Pods every 10 s). (general knowledge; verified: [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/), [Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/))

### Node ports

- A **NodePort** [[kubernetes-service|Service]] makes an app reachable from outside at `NodeIP:NodePort`. The tutorial says it uses "the same port of each selected Node"; in the tutorial, the Service's external IP is the Node's IP. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]]) The Service reference is more precise: every Node in the cluster proxies that same port into the Service. (general knowledge; verified: [Service](https://kubernetes.io/docs/concepts/services-networking/service/))

## GPU cloud bridge

- **A GPU Node is usually a bare-metal server** with 8 GPUs linked by NVLink (e.g. an 8× [[nvidia-a100]] HGX board). GPU clouds often skip the VM layer so customers get full GPU and network performance. (general knowledge)
- **Extra software on GPU Nodes:** the GPU driver, the NVIDIA Container Toolkit (so [[containerd]] can hand GPUs to containers), and the **device plugin**, which registers with the kubelet so the Node advertises e.g. 8 `nvidia.com/gpu` and the scheduler can count them. The NVIDIA GPU Operator installs all of this, plus GPU Feature Discovery labels and a DCGM-based metrics exporter ([[gpu-utilization-metrics]]). (general knowledge; verified: [Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/), [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/overview.html))
- **Node lifecycle is the bare-metal half of the lifecycle domain:** rack and image the server, install drivers, run health checks ([[nvidia-smi]], burn-in tests), join it to the cluster, and later take it out for repair. (general knowledge)
  - `kubectl cordon` marks the Node unschedulable: no new Pods, existing Pods keep running.
  - `kubectl drain` cordons the Node, then evicts its Pods through the Eviction API, so each Pod terminates gracefully and PodDisruptionBudgets (limits on how many replicas may be down at once) are respected. DaemonSet Pods need `--ignore-daemonsets`.
  - `kubectl uncordon` returns it to service. (general knowledge; verified: [Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/), [kubectl drain](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_drain/))
  - A single bad GPU can take a whole 8-GPU Node out of service. (general knowledge)
- **Node shape decides what fits.** A model that needs 4 GPUs on one NVLink domain (the set of GPUs wired to each other by NVLink, NVIDIA's fast GPU-to-GPU link) can only run on Nodes with 4 free GPUs, ideally connected ones; picking a connected set needs topology-aware allocation ([[model-parallelism]], [[gpu-memory-capacity]]). (general knowledge)
- **Tenancy:** customers needing hard isolation often get whole dedicated Nodes, so no other tenant's Pods share their machine. (general knowledge)

## Related

- [[kubernetes-pod]]: what runs on a Node.
- [[kubelet]], [[containerd]]: the two processes every Node must run.
- [[control-plane]]: what manages Nodes.
- [[kubernetes-scheduling]]: how Pods are placed on Nodes.
- [[nvidia-smi]]: first-line GPU health check on a Node.
- [[kubernetes-service]]: NodePort Services open the same port on every Node.

## Open questions

- How does a GPU cloud take a new bare-metal server to a schedulable GPU Node (provisioning, imaging, driver install, burn-in), and how long does that take?
- What exact signals mark a GPU Node unhealthy (XID errors, ECC errors, NVLink faults), and who drains it?
