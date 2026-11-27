---
type: concept
tags: [lifecycle, provisioning]
sources: [2026-11-22-k8s-viewing-pods-and-nodes, 2026-11-27-k8s-performing-a-rolling-update]
created: 2026-11-22
updated: 2026-11-27
---

# Container

A **container** is an application packaged with everything it needs to run (code, libraries, settings) and run as an isolated process on a shared machine. Analogy: a shipping container, which has a standard shape so any ship or crane can handle it no matter what is inside. A container **image** is the packed, read-only template; a container is a running instance of an image; a **registry** is the warehouse images are stored in and pulled from. (general knowledge)

## Details

- Kubernetes does not run bare containers: it runs them inside [[kubernetes-pod|Pods]], which add a shared IP address, port space, and volumes. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- A Pod's spec records how to run each container, such as its image version and which ports to use. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- On each [[kubernetes-node|Node]], a **container runtime** such as [[containerd]] pulls the image from a registry, unpacks it, and runs the application. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]]) The [[kubelet]] drives the runtime through the Container Runtime Interface (CRI), a gRPC API. (general knowledge; verified: [Container Runtime Interface](https://kubernetes.io/docs/concepts/containers/cri/))
- The tutorial's example app is a Node.js server (`server.js`) in a container listening on port 8080. `kubectl exec` runs commands inside it ([[kubectl]]). (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- Containers vs virtual machines: a VM emulates a whole computer with its own operating system; containers on one machine share the host's OS kernel. That makes them lighter and faster to start, with weaker isolation than a VM. (general knowledge)
- **Docker** popularized containers and the image format, and "Docker image" is still a common name for a container image. Kubernetes removed its built-in Docker Engine support (dockershim) in v1.24; Nodes now use a CRI runtime such as containerd or CRI-O (Docker Engine still works only through the separate `cri-dockerd` adapter). Images built with Docker run unchanged, because they follow the OCI (Open Container Initiative) image format. (general knowledge; verified: [Dockershim removal FAQ](https://kubernetes.io/blog/2022/02/17/dockershim-faq/)) The source shows this shift half-done: its Pod section still says "containers (such as Docker)", while its Node section names containerd as the runtime. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- **Image references and tags.** An image is named `registry/repository:tag`, e.g. `docker.io/jocatalin/kubernetes-bootcamp:v2`, and the tag picks the version. A [[kubernetes-rolling-update|rolling update]] is started by pointing a Deployment at a new tag. A tag that doesn't exist (`v10`) leaves new Pods in **`ImagePullBackOff`**, and `kubectl describe pods --show-events=true` shows that the image wasn't found. (src: [[2026-11-27-k8s-performing-a-rolling-update]]; naming breakdown: general knowledge) The [[kubelet]] keeps retrying the pull, waiting longer each time up to a 300 s cap. Bad names, bad tags, and missing private-registry credentials all fail this way. (general knowledge; verified: [Images, ImagePullBackOff](https://kubernetes.io/docs/concepts/containers/images/#imagepullbackoff))
- Tags are mutable labels: the same tag can be moved to a new image. Production deployments often pin the immutable content digest (`image@sha256:...`) instead, so that a rollback really returns to the same bytes. (general knowledge)

## GPU cloud bridge

General knowledge.

- An inference replica's image bundles the [[inference-frameworks|inference server]] (e.g. vLLM), CUDA libraries, and Python. Such images are typically many gigabytes, so pulling one onto a fresh Node adds to cold-start time.
- Model weights are usually *not* baked into the image. They are loaded at startup from object storage or a volume, which is another cold-start cost ([[gpu-memory-capacity]]).
- Containers do not see GPUs by default. The NVIDIA Container Toolkit configures the runtime to expose specific GPUs (and the driver libraries) to a container. (verified: [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/overview.html))

## Related

- [[kubernetes-pod]]: the wrapper Kubernetes puts around containers.
- [[containerd]]: the runtime that runs them on a Node.
- [[kubelet]]: tells the runtime which containers to run.
- [[kubectl]]: `logs` and `exec` act on containers.
- [[inference-frameworks]]: what the container runs in an inference replica.
- [[kubernetes-rolling-update]]: changing a Deployment's image tag is what starts a rollout.
