---
type: source
tags: [lifecycle, tenancy-quotas, metering-billing]
sources: []
raw: raw/2026-11-23-k8s-running-multiple-instances-of-your-app.md
url: https://kubernetes.io/docs/tutorials/kubernetes-basics/scale/scale-intro/
author: Kubernetes documentation (kubernetes.io)
created: 2026-11-23
updated: 2026-11-24
---

# Running Multiple Instances of Your App (Kubernetes Basics, Module 5)

Raw: [[raw/2026-11-23-k8s-running-multiple-instances-of-your-app]] · Official [[kubernetes]] tutorial "Kubernetes Basics", Module 5. Page last modified 2026-07-30, clipped 2026-11-23. Follows [[2026-11-22-k8s-using-a-service-to-expose-your-app]] (Module 4).

## Summary

**Scaling** an app in Kubernetes means changing one number: the replica count of its [[kubernetes-deployment|Deployment]]. That number is the *desired state*. The Deployment's [[kubernetes-replicaset|ReplicaSet]] compares it with how many [[kubernetes-pod|Pods]] actually run, then creates or terminates Pods until the two match. New Pods are [[kubernetes-scheduling|scheduled]] onto [[kubernetes-node|Nodes]] that have free resources. Scaling to zero is allowed and terminates every Pod. The [[kubernetes-service|Service]] in front keeps a live list of running Pods (its *endpoints*) and spreads traffic across all of them, so added replicas start receiving requests without any change to the Service. The hands-on part uses `kubectl scale` to go from 1 to 4 replicas and back down to 2. It shows that repeated `curl`s reach different Pods. Autoscaling is mentioned but out of scope. See [[kubernetes-scaling]].

**Why a control-plane engineer should care:** every inference endpoint's capacity, cost, and cold-start behaviour comes down to this replica count. Autoscalers, quotas, and GPU-hour meters all read or write it. (general knowledge)

![[raw/assets/k8s-scale-1.svg]]
Before scaling: Deployment A manages one Pod (10.0.0.5) on one of three Nodes, and Service A (10.3.250.236) covers just that Pod. (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]]; IPs read from the diagram)

![[raw/assets/k8s-scale-2.svg]]
After scaling to 4: the Pods are spread over all three Nodes (10.0.0.5; 10.0.2.4 and 10.0.2.5 on one Node; 10.0.1.8). The Service's boundary now encloses all four, **with the same Service IP**. Clients see one address while the set of Pods behind it grows. (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]]; IPs read from the diagram)

## Key points

### Scaling model

- Scaling is accomplished by changing the number of replicas in a Deployment. A Deployment can also start with several replicas: `kubectl create deployment ... --replicas=N`. (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]])
- Scaling out makes Kubernetes create new Pods and schedule them to Nodes with available resources, until the Pod count reaches the new desired state. (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]])
- Scaling to zero is possible. It terminates all Pods of the Deployment. (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]])
- Kubernetes also supports **autoscaling** of Pods, which this tutorial does not cover. (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]])
- Running multiple instances is what makes **rolling updates without downtime** possible (next module). (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]])

### Load balancing across replicas

- Multiple instances need a way to distribute traffic. Services have an integrated load balancer that distributes network traffic to all Pods of an exposed Deployment. (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]])
- Services continuously monitor running Pods using **endpoints**, so traffic goes only to available Pods. (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]])
- Demo: repeated `curl http://$(minikube ip):$NODE_PORT` calls get answers from different Pods (`Running on: kubernetes-bootcamp-644c5687f4-wp67j`, `...-hs9dj`, `...-4hjvf`). (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]])
- The tutorial assumes a `type: LoadBalancer` Service, but still reaches it through the NodePort. That works because a LoadBalancer Service is a superset of NodePort ([[kubernetes-service]]). (src for the commands: [[2026-11-23-k8s-running-multiple-instances-of-your-app]]; superset: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])

### Reading the status

| Command | Column | Meaning |
| --- | --- | --- |
| `kubectl get deployments` | READY | Ready / desired replicas, e.g. `1/1` |
| | UP-TO-DATE | Replicas updated to match the desired state |
| | AVAILABLE | Replicas available to users |
| | AGE | Time the app has been running |
| `kubectl get rs` | DESIRED | Replica count you asked for (the desired state) |
| | CURRENT | Replicas currently running |

(src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]])

- A ReplicaSet's name is `[DEPLOYMENT-NAME]-[RANDOM-STRING]`, where the string is generated using the **pod-template-hash** as a seed. (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]]) Correction: the suffix is the pod-template-hash itself, a deterministic hash of the Pod template, not a random string; see [[kubernetes-replicaset#Contradictions]]. (general knowledge; verified: [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#creating-a-deployment))
- After scaling, `kubectl get pods -o wide` shows 4 Pods with different IPs. `kubectl describe deployments/<name>` records the change in the Deployment's events. (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]])
- Scaling down to 2 with the same `scale` command terminates 2 Pods. (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]])
- All commands: [[kubectl]].

## Takeaways

- You don't tell Kubernetes to "start 3 more Pods". You set the desired count, and a controller closes the gap. The same loop that replaces a Pod after a Node failure also adds Pods when you scale up. (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]]; "same loop" framing: general knowledge)
- Capacity (the replica count) and access (the Service) are separate objects. Scaling changes only the first, and the Service follows automatically through endpoints. (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]])
- "Scheduled to Nodes with available resources" hides the hard case. When no Node has room, the new Pods wait as Pending, which is common when the resource is a GPU ([[kubernetes-scaling]], [[kubernetes-scheduling]]). (general knowledge)
- Scale-to-zero turns an idle Deployment into zero running Pods. On GPUs, that trades GPU-hours saved for a cold start on the next request ([[kubernetes-scaling]]). (general knowledge)

## Concepts & entities touched

[[kubernetes-scaling]] · [[kubernetes-replicaset]] · [[kubernetes-deployment]] · [[kubernetes-service]] · [[kubernetes-pod]] · [[kubernetes-scheduling]] · [[kubernetes]] · [[kubectl]] · [[inference-serving]] · [[control-plane]]

## Questions raised

- How does the Horizontal Pod Autoscaler work, which metrics can it scale on, and can it scale to zero? Partly answered in [[kubernetes-scaling]] (scale-to-zero is beta and on by default since v1.37).
- When scaling down, which Pods does the ReplicaSet pick to terminate, and how do in-flight requests (e.g. long streamed LLM responses) finish cleanly?
- What happens to a scale-up blocked by a tenant's ResourceQuota, compared with one blocked by a lack of free GPUs?
- How does a GPU cloud add Nodes when replicas are Pending (cluster autoscaler, bare-metal provisioning), and how long does that take?
- Next tutorial module: "Performing a Rolling Update".
