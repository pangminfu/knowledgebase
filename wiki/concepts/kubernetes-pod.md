---
type: concept
tags: [lifecycle, provisioning, metering-billing]
sources: [2026-11-22-k8s-viewing-pods-and-nodes]
created: 2026-11-22
updated: 2026-11-22
---

# Pod (Kubernetes)

A **Pod** is the smallest thing [[kubernetes]] schedules and runs: a group of one or more [[container|containers]] that share one IP address, one port space, and any storage volumes, and that always run together on the same [[kubernetes-node|Node]]. Analogy: a Pod is a small apartment; the containers are roommates who share the street address, the front door, and the storage closet, and they always move in and out together. You never create containers directly. A controller such as a [[kubernetes-deployment|Deployment]] creates Pods, and the containers come with them. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])

![[raw/assets/k8s-explore-pods.svg]]
Four Pods, each with its own IP (10.10.10.1–4). They range from one container (Pod 1) to three containers and two volumes (Pod 4). Every Pod has exactly one IP, however many containers it holds. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])

## Details

- What a Pod bundles (src: [[2026-11-22-k8s-viewing-pods-and-nodes]]):
  - **Volumes:** shared storage the containers can all read and write.
  - **Networking:** one unique cluster IP address, shared by all its containers.
  - **Run information:** per container, which image version to run and which ports to use.
- A Pod is an application-specific "logical host". Its containers are co-located, co-scheduled, and share a context on one Node. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- Group containers into one Pod only when they are tightly coupled and must share resources such as disk. Example: a Node.js web server plus a helper container that feeds it data. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]]) The helper pattern is commonly called a **sidecar**. (general knowledge)
- **Lifecycle:** a Pod stays bound to the Node it was scheduled on until it terminates (per its restart policy) or is deleted. If the Node fails, identical Pods are scheduled on other Nodes. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]]) The replacements are new Pods with new names and IPs; a Pod itself never migrates. (general knowledge)
- **Restart policy** decides what happens when a container in the Pod exits: restart it always, only on failure, or never. Services use "always"; batch jobs use "on failure" or "never". (general knowledge)
- Pods live on a private cluster network, so reaching one from a laptop needs a proxy such as `kubectl proxy` ([[kubectl]]). (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])

## GPU cloud bridge

All general knowledge; not from the source.

- **An inference replica is usually one Pod.** Its main container runs an [[inference-frameworks|inference server]] such as vLLM. The Pod requests GPUs in its resource spec, for example `nvidia.com/gpu: 4`, and the scheduler only places it on a Node with 4 free GPUs ([[kubernetes-scheduling]]).
- **GPUs are whole units by default.** A Pod gets whole GPUs, and two Pods never share one unless the operator turns on sharing (MIG partitions or time-slicing).
- **A replica that fits on one Node is one Pod.** A tensor-parallel replica that needs 4 GPUs on one Node ([[model-parallelism]]) is one Pod asking for 4 GPUs, so it lands all-or-nothing on one machine. Only replicas spanning several Nodes become several Pods that must be started together, which is the harder *gang scheduling* problem.
- **Pods are natural metering records.** "Pod X held 4 GPUs on Node Y from 10:02 to 11:47" is the raw fact behind GPU-hour billing ([[per-token-pricing]] covers the token-based alternative).
- **Replacement means cold start.** When a Node dies, the replacement inference Pod must pull its image and reload model weights before it can serve, which can take minutes for large models.

## Related

- [[kubernetes-node]]: the machine a Pod runs on.
- [[kubernetes-deployment]]: the controller that creates and replaces Pods.
- [[kubernetes-scheduling]]: how a Pod gets assigned to a Node.
- [[container]]: what a Pod holds.
- [[kubelet]]: the Node agent that actually starts a Pod's containers.
- [[inference-serving]]: the replicas a GPU cloud runs as Pods.

## Open questions

- How do multi-node inference replicas (and training jobs) get all their Pods started together? Needs a source on gang scheduling (e.g. Kueue, Volcano, LeaderWorkerSet).
