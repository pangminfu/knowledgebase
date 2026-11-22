---
type: concept
tags: [lifecycle, provisioning]
sources: [2026-11-22-k8s-viewing-pods-and-nodes]
created: 2026-11-22
updated: 2026-11-22
---

# Container

A **container** is an application packaged with everything it needs to run (code, libraries, settings) and run as an isolated process on a shared machine. Analogy: a shipping container, which has a standard shape so any ship or crane can handle it no matter what is inside. A container **image** is the packed, read-only template; a container is a running instance of an image; a **registry** is the warehouse images are stored in and pulled from. (general knowledge)

## Details

- Kubernetes does not run bare containers: it runs them inside [[kubernetes-pod|Pods]], which add a shared IP address, port space, and volumes. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- A Pod's spec records how to run each container, such as its image version and which ports to use. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- On each [[kubernetes-node|Node]], a **container runtime** such as [[containerd]] pulls the image from a registry, unpacks it, and runs the application. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- The tutorial's example app is a Node.js server (`server.js`) in a container listening on port 8080. `kubectl exec` runs commands inside it ([[kubectl]]). (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- Containers vs virtual machines: a VM emulates a whole computer with its own operating system; containers on one machine share the host's OS kernel. That makes them lighter and faster to start. (general knowledge)
- **Docker** popularized containers and the image format, and "Docker image" is still a common name for a container image. Kubernetes no longer uses Docker as its runtime; it uses containerd or similar, but still runs Docker-built images. (general knowledge) The source reflects this shift: its text says "containers (such as Docker)" but names containerd as the runtime. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])

## GPU cloud bridge

General knowledge.

- An inference replica's image bundles the [[inference-frameworks|inference server]] (e.g. vLLM), CUDA libraries, and Python. These images are often 10+ GB, so pulling one onto a fresh Node adds to cold-start time.
- Model weights are usually *not* baked into the image. They are loaded at startup from object storage or a volume, which is another cold-start cost ([[gpu-memory-capacity]]).
- Containers do not see GPUs by default. The NVIDIA container toolkit lets the runtime expose specific GPUs to a container.

## Related

- [[kubernetes-pod]]: the wrapper Kubernetes puts around containers.
- [[containerd]]: the runtime that runs them on a Node.
- [[kubectl]]: `logs` and `exec` act on containers.
- [[inference-frameworks]]: what the container runs in an inference replica.
