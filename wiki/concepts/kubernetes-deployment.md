---
type: concept
tags: [lifecycle, provisioning]
sources: [2026-11-22-k8s-viewing-pods-and-nodes, 2026-11-22-k8s-using-a-service-to-expose-your-app, 2026-11-23-k8s-running-multiple-instances-of-your-app]
created: 2026-11-22
updated: 2026-11-24
---

# Deployment (Kubernetes)

A **Deployment** is a [[kubernetes]] object that says "keep N identical copies of this [[kubernetes-pod|Pod]] running". You hand it a Pod template (which [[container]] image, which ports, which resources) and a replica count; Kubernetes creates the Pods and recreates any that disappear. Analogy: a standing order at a staffing agency for "three cashiers at all times"; if one leaves, the agency sends a replacement without you asking. (definition and analogy: general knowledge; Pod creation and replacement: src below)

## Details

- Creating a Deployment makes Kubernetes create Pods to host the application instances. A Deployment creates Pods with containers inside them, not containers directly. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- If a [[kubernetes-node|Node]] fails, identical Pods are scheduled on other available Nodes in the cluster. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]]) This is the Deployment's job: it notices it has fewer Pods than requested and creates new ones. (general knowledge)
- `kubectl describe` works on Deployments as well as Pods and Nodes ([[kubectl]]). (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- A Deployment works through a [[kubernetes-replicaset|ReplicaSet]], which drives the cluster back to the desired number of Pods by creating new ones when Pods are lost. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]]) The Deployment creates a new ReplicaSet per Pod-template version; that is how rolling updates work. (general knowledge)
- **Scaling** means changing the Deployment's replica count, its desired state. Its ReplicaSet then creates or terminates Pods to match, and new Pods go to Nodes with available resources. Use `kubectl scale deployments/<name> --replicas=N`, or set `--replicas` at `kubectl create deployment`. Scaling to zero terminates all Pods. See [[kubernetes-scaling]]. (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]])
- `kubectl get deployments` columns: **READY** (ready/desired replicas), **UP-TO-DATE** (replicas updated to the desired state), **AVAILABLE** (replicas available to users), **AGE**. A scale change shows up in the Deployment's events in `kubectl describe`. (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]])
- A Deployment labels its Pods automatically (the tutorial's key is `app`), and those [[kubernetes-labels-and-selectors|labels]] are what a [[kubernetes-service|Service]] selects on. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])
- A Deployment has no stable network address of its own. (general knowledge) `kubectl expose deployment/<name>` creates a separate Service for it. Deleting that Service cuts outside access but leaves the Pods running; to stop the app, delete the Deployment. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])
- Deployments also do **rolling updates**: when the Pod template changes (e.g. a new image version), old Pods are replaced a few at a time so the service stays up. (general knowledge) The tutorial says that once multiple instances run, "you would be able to do Rolling updates without downtime". (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]]) Strictly, one replica can also update without downtime, because the default surge setting starts the new Pod before the old one stops; several replicas make updates safer, since capacity never drops to a single Pod mid-rollout. (general knowledge)
- Deployments are for long-running services. Run-to-completion work uses a different controller, the **Job**. (general knowledge)

## GPU cloud bridge

General knowledge.

- A dedicated inference endpoint is roughly a Deployment of inference-server Pods behind a [[kubernetes-service|Service]] ([[inference-serving]], [[inference-frameworks]]). Autoscaling changes its replica count ([[kubernetes-scaling]]); a model version upgrade is a rolling update; the Service address stays the same throughout.
- The training-vs-inference split maps onto Deployment vs Job: inference is a service that should never stop, training is a job that should finish ([[training-vs-inference]]).
- A rolling update of a GPU service needs spare GPUs for the new Pods before old ones are removed, or it must remove first and briefly lose capacity. That headroom is a quota and cost question.

## Related

- [[kubernetes-pod]]: what a Deployment creates.
- [[kubernetes-replicaset]]: the per-template-version controller a Deployment manages.
- [[kubernetes-scaling]]: changing the replica count, by hand or by an autoscaler.
- [[kubernetes-scheduling]]: places the Pods a Deployment creates.
- [[kubernetes-service]]: the stable address in front of a Deployment's Pods.
- [[kubernetes-labels-and-selectors]]: how a Deployment and a Service find its Pods.
- [[control-plane]]: runs the controller that keeps the replica count.
- [[training-vs-inference]]: service (Deployment) vs batch job (Job).
- [[deployment-gates]]: who may trigger a rollout or rollback, and rehearsing rollback.

## Open questions

- Covered only through the tutorial so far (Modules 3–5). Module 2 ("Using kubectl to create a Deployment"), Module 6 (rolling updates), and the Deployments concept doc would fill in update strategy (`maxSurge`/`maxUnavailable`) and rollback.
