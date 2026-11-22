---
type: source
tags: [lifecycle, provisioning]
sources: []
raw: raw/2026-11-22-k8s-viewing-pods-and-nodes.md
url: https://kubernetes.io/docs/tutorials/kubernetes-basics/explore/explore-intro/
author: Kubernetes documentation (kubernetes.io)
created: 2026-11-22
updated: 2026-11-22
---

# Viewing Pods and Nodes (Kubernetes Basics, Module 3)

Raw: [[raw/2026-11-22-k8s-viewing-pods-and-nodes]] · Official [[kubernetes]] tutorial "Kubernetes Basics", Module 3. Page last modified 2026-07-22, clipped 2026-11-22. This is the wiki's first Kubernetes source.

## Summary

The page introduces the two basic building blocks of a Kubernetes cluster. A [[kubernetes-pod|Pod]] is a group of one or more [[container|containers]] that share an IP address, port space, and storage volumes, and are always placed together on one machine. Pods are the smallest unit Kubernetes schedules: you never create containers directly, a [[kubernetes-deployment|Deployment]] creates Pods. A [[kubernetes-node|Node]] is a worker machine, virtual or physical, that runs Pods. The [[control-plane]] manages every Node and [[kubernetes-scheduling|schedules]] Pods onto Nodes based on each Node's available resources. Every Node runs a [[kubelet]] (the control plane's agent on the machine) and a container runtime such as [[containerd]]. The second half is a hands-on [[kubectl]] walkthrough: `get`, `describe`, `logs`, `exec`, and `kubectl proxy`.

**Why a control-plane engineer should care:** in a GPU cloud, a Node is usually a bare-metal GPU server and a Pod is one workload on it, for example one inference replica holding some of that server's GPUs. Placing Pods on Nodes, replacing them when a Node dies, and tracking how long each one held its GPUs are the lifecycle, quota, and metering jobs this wiki is about. (general knowledge)

![[raw/assets/k8s-explore-pods.svg]]
Four example Pods, each a circle with its own cluster IP (10.10.10.1–10.10.10.4). Pod 1 holds a single containerized app; Pods 2–4 hold one to three containers plus one or two volumes (shared disks). The point: a Pod can be one container or several tightly coupled ones, but it always has exactly one IP. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])

![[raw/assets/k8s-explore-nodes.svg]]
One Node (the hexagon) running the same four Pods. The strip along the Node's edge labels the "node processes": kubelet and the container runtime, drawn as "Docker". The page text was updated in July 2026 to say containerd; the diagram was not. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])

## Key points

### Pods

- A Pod is a Kubernetes abstraction for one or more application containers plus shared resources: **volumes** (storage), **one unique cluster IP address** (networking), and **run information** per container, such as image version and ports. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- A Pod models an application-specific "logical host". Its containers share an IP and port space, are always co-located and co-scheduled, and run in a shared context on the same Node. Example: a Node.js web server plus a second container that feeds it the data to publish. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- Put containers in one Pod only if they are tightly coupled and need to share resources such as disk. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- **Pods are the atomic unit** on Kubernetes. A Deployment creates Pods, not containers. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- A Pod is tied to the Node it was scheduled on until it terminates (per its restart policy) or is deleted. If the Node fails, identical Pods are scheduled on other available Nodes. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])

### Nodes

- A Pod always runs on a Node. A Node is a worker machine, either virtual or physical depending on the cluster. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- Each Node is managed by the control plane. A Node can run many Pods, and the control plane schedules Pods across Nodes automatically, taking each Node's available resources into account. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- Every Node runs at least: (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
  - **[[kubelet]]**: communicates between the control plane and the Node, and manages the Pods and containers on the machine.
  - **A container runtime** (e.g. [[containerd]]): pulls the container image from a registry, unpacks it, and runs the application.

### Troubleshooting with kubectl

- Four common subcommands: `kubectl get` (list resources), `kubectl describe` (detailed information), `kubectl logs` (a container's logs), `kubectl exec` (run a command in a container). Together they show when apps were deployed, their status, where they run, and their configuration. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- `kubectl describe pods` shows a Pod's containers, IP address, ports, and a list of **events** in the Pod's lifecycle. `describe` works on most objects (Nodes, Pods, Deployments). Its output is for humans and should not be parsed by scripts. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- Pods live on an isolated, private network. `kubectl proxy` opens a local tunnel (port 8001) to the cluster's API, and a Pod can then be reached at `/api/v1/namespaces/default/pods/<pod>:8080/proxy/`. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- With one container in a Pod, `exec` and `logs` need no container name. `kubectl exec -ti <pod> -- bash` opens a shell inside the container. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- Full command reference: [[kubectl]].

## Takeaways

- The Pod, not the container, is the scheduling unit: a small group of containers that must share a machine, an IP, and disk. Kubernetes places, restarts, and replaces Pods, never bare containers. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- When a Node dies, its Pods are not moved; they are lost. The controller that owns them, such as a Deployment, creates identical replacement Pods, and the scheduler places them on healthy Nodes. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]]; "not moved, replaced" framing: general knowledge) For a GPU inference replica, that means a cold start while the replacement reloads model weights. (general knowledge)
- Every Node runs kubelet plus a container runtime. GPU Nodes add the NVIDIA driver, the NVIDIA container toolkit, and the GPU device plugin, so that Pods can request GPUs. (first two: src; GPU add-ons: general knowledge)

## Concepts & entities touched

[[kubernetes]] · [[kubernetes-pod]] · [[kubernetes-node]] · [[kubernetes-deployment]] · [[control-plane]] · [[kubernetes-scheduling]] · [[container]] · [[kubelet]] · [[containerd]] · [[kubectl]] · [[inference-serving]] · [[model-parallelism]]

## Questions raised

- How does a Pod ask for GPUs, and how does the scheduler know which Nodes have free ones? (Device plugins and extended resources; next source to find.)
- How long after a Node fails are its Pods replaced, and what decides that? (Node heartbeats, eviction timeouts; see the Nodes concept doc linked from the page.)
- The URL path `/api/v1/namespaces/default/...` shows Pods live inside a **namespace**. Namespaces are Kubernetes' basic tenancy boundary (general knowledge). A source on namespaces and ResourceQuota would open the tenancy-quotas domain on the Kubernetes side.
- The source says a Node "may be either a virtual or a physical machine". How do GPU clouds bring bare-metal servers into a cluster as Nodes (provisioning, driver install, health gating)?
- Next tutorial module: Services, i.e. how a stable address fronts Pods that come and go. Relevant to load balancing across inference replicas.
