---
type: entity
tags: [lifecycle]
sources: [2026-11-22-k8s-viewing-pods-and-nodes]
created: 2026-11-22
updated: 2026-11-24
---

# containerd

**containerd** is a **container runtime**: the program on each [[kubernetes-node|Node]] that pulls a [[container]] image from a registry, unpacks it, and runs it. In [[kubernetes]], the [[kubelet]] tells containerd which containers to start and stop. It started as a component of Docker and is now a separate open-source project. (src for role: [[2026-11-22-k8s-viewing-pods-and-nodes]]; history: general knowledge)

## Details

- Every Node runs a container runtime "like containerd", responsible for pulling the container image from a registry, unpacking the container, and running the application. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- The tutorial switched its runtime example from Docker to containerd in July 2026 (commit "Update container runtime reference from Docker to containerd"), but its Nodes diagram still says "Docker". (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- Kubernetes removed its built-in Docker support ("dockershim") in v1.24 (2022). Images built with Docker still run under containerd because both use the same standard image format, defined by the OCI (Open Container Initiative). (general knowledge)

## Why it matters for a GPU cloud

General knowledge.

- With the NVIDIA container toolkit installed, containerd can expose specific GPUs and driver libraries inside a container. Without it, a container cannot see the GPUs.
- Image pull time is part of cold start. Large inference images make registry mirrors and image caching on Nodes a lifecycle concern.

## Related

- [[kubelet]]: tells it what to run.
- [[container]]: what it runs.
- [[kubernetes-node]]: where it runs.
