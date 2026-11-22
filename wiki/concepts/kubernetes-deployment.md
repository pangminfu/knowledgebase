---
type: concept
tags: [lifecycle, provisioning]
sources: [2026-11-22-k8s-viewing-pods-and-nodes]
created: 2026-11-22
updated: 2026-11-22
---

# Deployment (Kubernetes)

A **Deployment** is a [[kubernetes]] object that says "keep N identical copies of this [[kubernetes-pod|Pod]] running". You hand it a Pod template (which [[container]] image, which ports, which resources) and a replica count; Kubernetes creates the Pods and recreates any that disappear. Analogy: a standing order at a staffing agency for "three cashiers at all times"; if one leaves, the agency sends a replacement without you asking. (definition and analogy: general knowledge; Pod creation and replacement: src below)

## Details

- Creating a Deployment makes Kubernetes create Pods to host the application instances. A Deployment creates Pods with containers inside them, not containers directly. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- If a [[kubernetes-node|Node]] fails, identical Pods are scheduled on other available Nodes in the cluster. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]]) This is the Deployment's job: it notices it has fewer Pods than requested and creates new ones. (general knowledge)
- `kubectl describe` works on Deployments as well as Pods and Nodes ([[kubectl]]). (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- Deployments also do **rolling updates**: when the Pod template changes (e.g. a new image version), old Pods are replaced a few at a time so the service stays up. (general knowledge)
- Deployments are for long-running services. Run-to-completion work uses a different controller, the **Job**. (general knowledge)

## GPU cloud bridge

General knowledge.

- A dedicated inference endpoint is roughly a Deployment of inference-server Pods ([[inference-serving]], [[inference-frameworks]]). Autoscaling changes its replica count; a model version upgrade is a rolling update.
- The training-vs-inference split maps onto Deployment vs Job: inference is a service that should never stop, training is a job that should finish ([[training-vs-inference]]).
- A rolling update of a GPU service needs spare GPUs for the new Pods before old ones are removed, or it must remove first and briefly lose capacity. That headroom is a quota and cost question.

## Related

- [[kubernetes-pod]]: what a Deployment creates.
- [[kubernetes-scheduling]]: places the Pods a Deployment creates.
- [[control-plane]]: runs the controller that keeps the replica count.
- [[training-vs-inference]]: service (Deployment) vs batch job (Job).

## Open questions

- Covered only in passing by the current source. The Kubernetes Basics Module 2 ("Using kubectl to create a Deployment") and the Deployments concept doc would fill this page in.
