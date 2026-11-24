---
type: concept
tags: [lifecycle, provisioning, metering-billing]
sources: [2026-11-22-k8s-viewing-pods-and-nodes, 2026-11-22-k8s-using-a-service-to-expose-your-app]
created: 2026-11-22
updated: 2026-11-24
---

# Pod (Kubernetes)

A **Pod** is the smallest thing [[kubernetes]] schedules and runs: a group of one or more [[container|containers]] that share one IP address, one port space, and any storage volumes, and that always run together on the same [[kubernetes-node|Node]]. Analogy: a Pod is a small apartment; the containers are roommates who share the street address, the front door, and the storage closet, and they always move in and out together. You do not create containers directly: a controller such as a [[kubernetes-deployment|Deployment]] creates Pods, and the containers come with them. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])

![[raw/assets/k8s-explore-pods.svg]]
Four Pods, each with its own IP (10.10.10.1–4). They range from one container (Pod 1) to three containers and two volumes (Pod 4). Every Pod has exactly one IP, however many containers it holds. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])

## Details

### What a Pod is

- What a Pod bundles (src: [[2026-11-22-k8s-viewing-pods-and-nodes]]):
  - **Volumes:** shared storage the containers can all read and write.
  - **Networking:** one unique cluster IP address, shared by all its containers.
  - **Run information:** per container, which image version to run and which ports to use.
- A Pod is an application-specific "logical host". Its containers are co-located, co-scheduled, and share a context on one Node. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- Group containers into one Pod only when they are tightly coupled and must share resources such as disk. Example: a Node.js web server plus a helper container that feeds it data. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- A helper container like that is commonly called a **sidecar**. Since v1.33 Kubernetes also has *native* sidecar containers (stable; on by default since v1.29): an entry in the Pod's `initContainers` with `restartPolicy: Always`, which starts before the main containers and keeps running alongside them. (general knowledge; verified: [Sidecar Containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/))

### Lifecycle

- A Pod stays bound to the Node it was scheduled on until it terminates (per its restart policy) or is deleted. If the Node fails, identical Pods are scheduled on other Nodes. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- More precisely: a Pod is never moved. The replacement is a new, near-identical Pod with a new UID, created by the Pod's controller (e.g. a Deployment's [[kubernetes-replicaset|ReplicaSet]]); a bare Pod with no controller is not replaced. (general knowledge; verified: [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)) Under a Deployment the replacement also gets a new name and, in practice, a new IP. (general knowledge)
- **Restart policy** (`spec.restartPolicy`) decides what the [[kubelet]] does when a container in the Pod exits; it restarts the container in place on the same Node: `Always` (the default; restart after any exit), `OnFailure` (only after a non-zero exit), or `Never`. Deployments allow only `Always`; Jobs use `OnFailure` or `Never`. (general knowledge; verified: [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/))

### Networking

- Pods live on a private cluster network, so reaching one from a laptop needs a proxy such as `kubectl proxy` ([[kubectl]]). (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- **Every Pod has its own IP, even Pods on the same Node.** Those IPs are not exposed outside the cluster without a Service. Clients therefore reach Pods through a [[kubernetes-service|Service]], which usually picks them by [[kubernetes-labels-and-selectors|label selector]]. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]]) Because replacement Pods get new IPs, nothing should depend on a Pod's IP. (general knowledge)
- Labels can be added to a running Pod at any time, e.g. `kubectl label pods "$POD_NAME" version=v1`. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])

## GPU cloud bridge

- **An inference replica is usually one Pod.** Its main container runs an [[inference-frameworks|inference server]] such as vLLM. (general knowledge)
- **GPUs are requested as whole units.** The container sets e.g. `nvidia.com/gpu: 4` under `resources.limits` (the request defaults to the limit; if both are set they must be equal), and the scheduler only places the Pod on a Node with 4 unallocated GPUs ([[kubernetes-scheduling]]). Such device resources are integers, cannot be overcommitted, and a device is not shared between containers. (general knowledge; verified: [Schedule GPUs](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/), [Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/))
- **GPU sharing is opt-in.** Two Pods only share one physical GPU if the operator enables it: MIG (Multi-Instance GPU, which splits a supported GPU into isolated hardware slices, each schedulable like a small GPU), or the NVIDIA device plugin's time-slicing or MPS modes, which advertise one GPU as several "replicas". Time-slicing gives no memory or fault isolation (one crash takes down all sharers); MPS partitions memory and compute per workload. (general knowledge; verified: [NVIDIA k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin))
- **A replica that fits on one Node is one Pod.** A tensor-parallel replica that needs 4 GPUs on one Node ([[model-parallelism]]) is one Pod asking for 4 GPUs, so it lands all-or-nothing on one machine. Only replicas spanning several Nodes become several Pods that must start together: *gang scheduling* (see [[kubernetes-scheduling]]). (general knowledge)
- **Pods are natural metering records.** "Pod X held 4 GPUs on Node Y from 10:02 to 11:47" is the raw fact behind GPU-hour billing ([[per-token-pricing]] covers the token-based alternative). (general knowledge)
- **Replacement means cold start.** When a Node dies, the replacement inference Pod must pull its image and reload model weights before it can serve, which can take minutes for large models. (general knowledge)

## Related

- [[kubernetes-node]]: the machine a Pod runs on.
- [[kubernetes-deployment]]: the controller that creates and replaces Pods.
- [[kubernetes-scheduling]]: how a Pod gets assigned to a Node.
- [[container]]: what a Pod holds.
- [[kubelet]]: the Node agent that actually starts and restarts a Pod's containers.
- [[kubernetes-service]]: the stable address in front of Pods that come and go.
- [[kubernetes-labels-and-selectors]]: the tags that decide which Service routes to a Pod.
- [[inference-serving]]: the replicas a GPU cloud runs as Pods.

## Open questions

- How do multi-node inference replicas (and training jobs) get all their Pods started together in practice: built-in gang scheduling (beta, off by default in v1.37) or add-ons such as Kueue, Volcano, and LeaderWorkerSet? Needs a dedicated source.
