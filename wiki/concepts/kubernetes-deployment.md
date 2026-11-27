---
type: concept
tags: [lifecycle, provisioning]
sources: [2026-11-22-k8s-viewing-pods-and-nodes, 2026-11-22-k8s-using-a-service-to-expose-your-app, 2026-11-23-k8s-running-multiple-instances-of-your-app, 2026-11-27-k8s-performing-a-rolling-update]
created: 2026-11-22
updated: 2026-11-27
---

# Deployment (Kubernetes)

A **Deployment** is a [[kubernetes]] object that says "keep N identical copies of this [[kubernetes-pod|Pod]] running". You hand it a Pod template (which [[container]] image, which ports, which resources) and a replica count; Kubernetes creates the Pods and recreates any that disappear. Analogy: a standing order at a staffing agency for "three cashiers at all times"; if one leaves, the agency sends a replacement without you asking. (definition and analogy: general knowledge; Pod creation and replacement: src below)

## Details

- Creating a Deployment makes Kubernetes create Pods to host the application instances. A Deployment creates Pods with containers inside them, not containers directly. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- If a [[kubernetes-node|Node]] fails, identical Pods are scheduled on other available Nodes in the cluster. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]]) This is the Deployment's job: it notices it has fewer Pods than requested and creates new ones. (general knowledge)
- `kubectl describe` works on Deployments as well as Pods and Nodes ([[kubectl]]). (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- A Deployment works through a [[kubernetes-replicaset|ReplicaSet]], which drives the cluster back to the desired number of Pods by creating new ones when Pods are lost. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]]) The Deployment creates a new ReplicaSet per Pod-template version; that is how [[kubernetes-rolling-update|rolling updates]] work. (general knowledge)
- **Scaling** means changing the Deployment's replica count, its desired state. Its ReplicaSet then creates or terminates Pods to match, and new Pods go to Nodes with available resources. Use `kubectl scale deployments/<name> --replicas=N`, or set `--replicas` at `kubectl create deployment`. Scaling to zero terminates all Pods. See [[kubernetes-scaling]]. (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]])
- `kubectl get deployments` columns: **READY** (ready/desired replicas), **UP-TO-DATE** (replicas updated to the desired state), **AVAILABLE** (replicas available to users), **AGE**. A scale change shows up in the Deployment's events in `kubectl describe`. (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]])
- A Deployment labels its Pods automatically (the tutorial's key is `app`), and those [[kubernetes-labels-and-selectors|labels]] are what a [[kubernetes-service|Service]] selects on. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])
- A Deployment has no stable network address of its own. (general knowledge) `kubectl expose deployment/<name>` creates a separate Service for it. Deleting that Service cuts outside access but leaves the Pods running; to stop the app, delete the Deployment. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])
- Deployments also do **[[kubernetes-rolling-update|rolling updates]]**: when the Pod template changes (e.g. `kubectl set image` to a new image version), Pods are replaced incrementally so the app stays up, and the [[kubernetes-service|Service]] routes only to Pods that can serve. Updates are versioned; `kubectl rollout undo` reverts to the previous one. (src: [[2026-11-27-k8s-performing-a-rolling-update]])
- Both tutorial modules present multiple replicas as what makes zero-downtime updates possible ("a requirement" in Module 6). (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]], [[2026-11-27-k8s-performing-a-rolling-update]]) Strictly, one replica can also update without downtime, because the default surge setting starts the new Pod before the old one stops; several replicas make updates safer, since capacity never drops to a single Pod mid-rollout. (general knowledge; see [[kubernetes-rolling-update#Contradictions]])
- Rolling-update defaults: `strategy.type: RollingUpdate` with `maxSurge: 25%` (rounded up) and `maxUnavailable: 25%` (rounded down); the tutorial's "one and one" holds only at 4 replicas. `revisionHistoryLimit` defaults to 10 old ReplicaSets kept for rollback; `progressDeadlineSeconds` (default 600) marks a stalled rollout as failed but triggers no rollback. Worked numbers and failure behaviour: [[kubernetes-rolling-update]]. (general knowledge; verified: [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment))
- Deployments are for long-running services. Run-to-completion work uses a different controller, the **Job**. (general knowledge)

## GPU cloud bridge

General knowledge.

- A dedicated inference endpoint is roughly a Deployment of inference-server Pods behind a [[kubernetes-service|Service]] ([[inference-serving]], [[inference-frameworks]]). Autoscaling changes its replica count ([[kubernetes-scaling]]); a model version upgrade is a rolling update; the Service address stays the same throughout.
- The training-vs-inference split maps onto Deployment vs Job: inference is a service that should never stop, training is a job that should finish ([[training-vs-inference]]).
- A rolling update of a GPU service needs spare GPUs for the new Pods before old ones are removed, or it must remove first and briefly lose capacity. That headroom is a quota and cost question ([[kubernetes-rolling-update#GPU cloud bridge]]).

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
- [[kubernetes-rolling-update]]: how a template change is rolled out, paced, watched, and undone.

## Open questions

- Covered through the tutorial Modules 3–6. Module 2 ("Using kubectl to create a Deployment") and the Deployments concept doc as a source in its own right (pausing/resuming rollouts, proportional scaling during a rollout) remain unread.
