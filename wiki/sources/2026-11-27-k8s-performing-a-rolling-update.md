---
type: source
tags: [lifecycle, tenancy-quotas, software-delivery]
sources: []
raw: raw/2026-11-27-k8s-performing-a-rolling-update.md
url: https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/
author: Kubernetes documentation (kubernetes.io)
created: 2026-11-27
updated: 2026-11-27
---

# Performing a Rolling Update (Kubernetes Basics, Module 6)

Raw: [[raw/2026-11-27-k8s-performing-a-rolling-update]] · Module 6 of the official [[kubernetes]] tutorial "Kubernetes Basics". Page last modified 2026-11-15, clipped 2026-11-27. Follows [[2026-11-23-k8s-running-multiple-instances-of-your-app]] (Module 5).

## Summary

A **[[kubernetes-rolling-update|rolling update]]** ships a new version of an app with no downtime. Kubernetes replaces the [[kubernetes-pod|Pods]] of a [[kubernetes-deployment|Deployment]] a few at a time. New Pods are scheduled onto [[kubernetes-node|Nodes]] with free resources, and Kubernetes waits for them to start before it removes old ones. Two limits set the pace: how many Pods may be unavailable, and how many extra Pods may be created. The tutorial says both default to one. That is not quite right; see [[kubernetes-rolling-update#Contradictions]]. Throughout the update, the [[kubernetes-service|Service]] sends traffic only to Pods that can handle requests. Updates are versioned, so any of them can be reverted. In the hands-on part, `kubectl set image` moves the app to `v2`. A second update to a `v10` tag that doesn't exist stalls with `ImagePullBackOff`, and `kubectl rollout undo` brings the app back to `v2`.

**Why this matters for a GPU cloud:** upgrading a model or an inference server on a live endpoint is a rolling update. The extra Pods it creates need free GPUs, and each new Pod must load its model before it can serve, so GPU capacity and cold start decide how fast and how safely the upgrade can go. (general knowledge)

### The diagrams

![[raw/assets/k8s-rollingupdates-1.svg]]
Before the update: Deployment A runs four Pods on three Nodes (10.0.0.5; 10.0.2.4 and 10.0.2.5 together on one Node; 10.0.1.8), all on the old version. Service A (10.3.250.236) covers all four. (src: [[2026-11-27-k8s-performing-a-rolling-update]]; IPs read from the diagram)

![[raw/assets/k8s-rollingupdates-2.svg]]
Step 1: a new Pod with the updated app and a new IP (10.0.0.6) appears next to 10.0.0.5, which is fading out. The Service boundary already includes the new Pod. (src: [[2026-11-27-k8s-performing-a-rolling-update]])

![[raw/assets/k8s-rollingupdates-3.svg]]
Step 2: 10.0.0.5 is gone. On the bottom Node, 10.0.1.8 has been replaced by an updated Pod with a new IP (10.0.1.6). Two of the four Pods now run the new version. (src: [[2026-11-27-k8s-performing-a-rolling-update]])

![[raw/assets/k8s-rollingupdates-4.svg]]
Done: the last two Pods were replaced by 10.0.2.6 and 10.0.2.7, so all four run the new version, each with a new IP. **The Service IP never changed**, so clients didn't notice the swap. (src: [[2026-11-27-k8s-performing-a-rolling-update]])

## Key points

### How a rolling update works

- Users expect apps to be available all the time, and developers ship new versions several times a day. Kubernetes handles this with rolling updates. (src: [[2026-11-27-k8s-performing-a-rolling-update]])
- A rolling update replaces the current Pods with new ones step by step. New Pods are scheduled on Nodes with available resources, and Kubernetes waits for them to start before it removes old ones. (src: [[2026-11-27-k8s-performing-a-rolling-update]])
- The tutorial calls running multiple instances (Module 5) a *requirement* for updating without affecting availability. (src: [[2026-11-27-k8s-performing-a-rolling-update]]) This wiki treats it as strongly recommended rather than strictly required; see [[kubernetes-deployment]].
- The tutorial says the default for both the maximum number of unavailable Pods and the maximum number of extra new Pods is **one**. Both can be set as a number or as a percentage of Pods. (src: [[2026-11-27-k8s-performing-a-rolling-update]]) The real defaults are 25% each. They happen to come out to 1 and 1 at exactly 4 replicas ([[kubernetes-rolling-update#Contradictions]]).
- Updates are versioned. Any Deployment update can be reverted to an earlier stable version. (src: [[2026-11-27-k8s-performing-a-rolling-update]])
- If the Deployment is exposed, the Service sends traffic only to Pods that can handle requests, so users keep reaching the app during the update. (src: [[2026-11-27-k8s-performing-a-rolling-update]])
- Rolling updates support three things: promoting an app from one environment to another by updating its container image, rolling back to earlier versions, and CI/CD (continuous integration and delivery) with zero downtime. (src: [[2026-11-27-k8s-performing-a-rolling-update]])

### Hands-on: update, verify, roll back

| Step | Command | What you see |
| --- | --- | --- |
| Check the current image | `kubectl describe pods` | the `Image` field |
| Start the update | `kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=docker.io/jocatalin/kubernetes-bootcamp:v2` | the Deployment is told to use a new image, which starts a rolling update |
| Watch it | `kubectl get pods` | new Pods come up, old ones terminate |
| Verify | `curl http://"$(minikube ip):$NODE_PORT"` (repeat) | a different Pod answers each time, all on `v2` |
| Confirm | `kubectl rollout status deployments/kubernetes-bootcamp` | the rollout's status |
| Bad update | `kubectl set image ... kubernetes-bootcamp=gcr.io/google-samples/kubernetes-bootcamp:v10` | `get deployments` shows fewer available Pods than desired; some Pods are in `ImagePullBackOff` |
| Diagnose | `kubectl describe pods --show-events=true` | the `Events` section says the `v10` image doesn't exist in the repository |
| Roll back | `kubectl rollout undo deployments/kubernetes-bootcamp` | the Deployment goes back to the previous known state (`v2`) |
| Clean up | `kubectl delete deployments/kubernetes-bootcamp services/kubernetes-bootcamp` | the Deployment and Service are deleted |

(src: [[2026-11-27-k8s-performing-a-rolling-update]]; all commands collected in [[kubectl]])

- In the `set image` arguments, the part before `=` names the *container* inside the Pod template, and the part after it is the new image reference, `registry/repository:tag`. (general knowledge)
- The failure is an image-pull problem, not an app crash: the [[kubelet]] cannot download the `v10` image, so the container never starts. (src: [[2026-11-27-k8s-performing-a-rolling-update]]; kubelet's role: general knowledge) The `v10` update also switches registry, from `docker.io/jocatalin` to `gcr.io/google-samples`, but the events blame the missing tag. (src: [[2026-11-27-k8s-performing-a-rolling-update]])
- If the Service was deleted in Module 4, recreate it with `kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080`. (src: [[2026-11-27-k8s-performing-a-rolling-update]])

## Takeaways

- A version change is a desired-state change, like scaling. You edit the Deployment's Pod template, the image here, and controllers move the Pods to match, a few at a time ([[kubernetes-replicaset]]). (src for `set image` starting the rollout: [[2026-11-27-k8s-performing-a-rolling-update]]; reconciliation framing: general knowledge)
- Zero downtime comes from three parts working together: limits on how many Pods are replaced at once, readiness gating in the Service, and a stable Service address. If any one is missing, users see errors during the update. (src: [[2026-11-27-k8s-performing-a-rolling-update]]; "readiness gating": general knowledge, see [[kubernetes-service]])
- A bad rollout **stalls; it does not roll itself back.** The unavailable limit keeps most old Pods serving, but someone (a person, or a pipeline watching the rollout) has to notice and run `rollout undo`. (stall and manual undo: src; "does not roll back by itself": general knowledge; verified: [Deployments, failed Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#failed-deployment))
- On GPUs, every new Pod needs free GPUs and a model load before it counts as started. That makes rollouts and rollbacks slower and capacity-hungry. Details in [[kubernetes-rolling-update#GPU cloud bridge]]. (general knowledge)

## Concepts & entities touched

[[kubernetes-rolling-update]] · [[kubernetes-deployment]] · [[kubernetes-replicaset]] · [[kubernetes-service]] · [[kubernetes-pod]] · [[kubernetes-node]] · [[container]] · [[kubelet]] · [[kubectl]] · [[kubernetes-scaling]] · [[deployment-gates]] · [[kubernetes]]

## Questions raised

- What does the Deployment report when a rollout stalls, and how long until it is declared failed? Answered in [[kubernetes-rolling-update]]: `ProgressDeadlineExceeded` after 600 s by default, and no automatic action.
- How do inference platforms upgrade a model when the surge Pod can't get a GPU: lower `maxSurge`, use spare capacity, or switch traffic between two Deployments (blue/green)?
- While a model upgrade is in progress, the old and new versions answer side by side. How do platforms keep clients from seeing inconsistent results?
- How does a pipeline turn `rollout status` or the Deployment's conditions into an automatic rollback ([[deployment-gates]])?
- Next: the Kubernetes Basics tutorial ends here. The Deployments concept doc and the autoscaling docs are the natural follow-ups.
