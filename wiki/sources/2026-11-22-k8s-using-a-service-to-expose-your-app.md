---
type: source
tags: [provisioning, lifecycle, tenancy-quotas]
sources: []
raw: raw/2026-11-22-k8s-using-a-service-to-expose-your-app.md
url: https://kubernetes.io/docs/tutorials/kubernetes-basics/expose/expose-intro/
author: Kubernetes documentation (kubernetes.io)
created: 2026-11-22
updated: 2026-11-22
---

# Using a Service to Expose Your App (Kubernetes Basics, Module 4)

Raw: [[raw/2026-11-22-k8s-using-a-service-to-expose-your-app]] · Official [[kubernetes]] tutorial "Kubernetes Basics", Module 4. Page last modified 2026-08-26, clipped 2026-11-22. Follows [[2026-11-22-k8s-viewing-pods-and-nodes]] (Module 3).

## Summary

[[kubernetes-pod|Pods]] are mortal: they die with their [[kubernetes-node|Node]] and come back as new Pods with new IP addresses. A [[kubernetes-service|Service]] solves this. It is a stable object that defines a logical set of Pods and a policy for reaching them, and it provides load balancing, service discovery, and (optionally) access from outside the cluster. A Service picks its Pods with a **label selector**, a query over the key/value [[kubernetes-labels-and-selectors|labels]] attached to objects. The `type` field sets how far the Service reaches: ClusterIP (inside the cluster only), NodePort (a port on every Node), LoadBalancer (an external cloud load balancer), or ExternalName (a DNS alias only). The hands-on part exposes the tutorial's [[kubernetes-deployment|Deployment]] as a NodePort Service, queries and adds labels with [[kubectl]], then deletes the Service and shows that the app keeps running.

**Why a control-plane engineer should care:** an inference endpoint needs one stable address in front of replicas that are constantly replaced, rescheduled, and autoscaled. In Kubernetes, that address is a Service. Labels are also the general tagging tool that tenancy, metering, and rollout logic build on. (general knowledge)

![[raw/assets/k8s-expose-labels.svg]]
A Service (`svc`, IP 1.1.1.1) with selector `app=A` sends traffic to every Pod labeled `app: A`. That includes three Pods (10.10.10.1–3) inside a Deployment → ReplicaSet and one standalone Pod (10.10.10.5) outside it. The Pod labeled `app: B` (10.10.10.4) gets nothing. The point: a Service selects by label, not by which Deployment owns a Pod. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]]; reading of the stray-Pod arrow: from the diagram)

## Key points

### Why Services exist

- Pods have a lifecycle. When a worker Node dies, its Pods are lost, and a **ReplicaSet** may create new Pods to bring the cluster back to the desired state. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])
- Example: an image-processing backend with 3 replicas. The replicas are interchangeable, and the front end should not care which one answers or whether one was recreated. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])
- Every Pod has a unique IP, even Pods on the same Node, so something must keep up with Pods as they change. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])
- A Service is an abstraction that defines a logical set of Pods and enables **external traffic exposure, load balancing, and service discovery** for them. It gives dependent Pods loose coupling: a frontend talks to the backend Service, not to backend Pods. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])
- Like every Kubernetes object, a Service is defined in YAML or JSON. Pod IPs are not exposed outside the cluster without a Service. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])

### Service types (`spec.type`)

| Type | Reachable from | How | Relation |
| --- | --- | --- | --- |
| **ClusterIP** (default) | Inside the cluster only | An internal cluster IP | Base |
| **NodePort** | Outside, via `NodeIP:NodePort` | The same port opened on each selected Node, using NAT | Superset of ClusterIP |
| **LoadBalancer** | Outside, via a fixed external IP | The cloud creates an external load balancer (if supported) | Superset of NodePort |
| **ExternalName** | n/a | Returns a DNS `CNAME` to the `externalName` value; no proxying | Separate |

(src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])

- **Services without a selector** do not get a corresponding Endpoints object, so you can map the Service to specific endpoints by hand. ExternalName Services also have no selector. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])

### Labels and selectors

- A Service matches its Pods with labels and selectors, Kubernetes' grouping primitive for logical operations on objects. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])
- Labels are key/value pairs attached to objects. Uses: marking objects as development, test, or production; embedding version tags; classifying objects. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])
- Labels can be attached at creation or later, and changed at any time. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])

### Hands-on

- `kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080` creates a Service. It gets a unique cluster IP, an internal port, and an external IP (the Node's IP). `kubectl describe services/...` shows which Node port was opened. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])
- The app is reachable from outside with `curl http://$(minikube ip):$NODE_PORT`. On minikube with the Docker Desktop driver, the host can't reach the Node directly, so `minikube service <name> --url` opens a tunnel. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])
- The Deployment labeled its Pod automatically (`app=kubernetes-bootcamp`). `kubectl get pods -l app=...` and `kubectl get services -l app=...` filter by label. `kubectl label pods $POD_NAME version=v1` adds a label to a running Pod. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])
- `kubectl delete service -l app=kubernetes-bootcamp` removes the Service, and the app is no longer reachable from outside. `curl localhost:8080` from inside the Pod still works, because the Deployment still manages the app. To shut the app down, delete the Deployment too. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])
- All commands: [[kubectl]].

## Takeaways

- A Service separates *how clients reach an app* from *which Pods currently run it*. Pods come and go; the Service's address and name stay put. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])
- Membership is a live label query, not a list. Any Pod whose labels match the selector receives traffic, whoever created it, so labels effectively control routing. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]]; "live query" framing: general knowledge)
- The four types form a ladder of exposure. LoadBalancer is where Kubernetes hands off to the cloud provider's own networking. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])
- Exposure and workload lifecycle are separate objects. Deleting a Service cuts the route but not the Pods, and deleting Pods does not delete the Service. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])

## Concepts & entities touched

[[kubernetes-service]] · [[kubernetes-labels-and-selectors]] · [[kubernetes-pod]] · [[kubernetes-node]] · [[kubernetes-deployment]] · [[kubernetes]] · [[kubectl]] · [[inference-serving]]

## Questions raised

- The page mentions the Endpoints object. Current Kubernetes tracks Service backends with **EndpointSlices** (general knowledge). How do EndpointSlices work, and how quickly does a dead Pod leave one?
- A Service balances *connections* (layer 4). LLM requests are long, streamed, and benefit from prefix-cache-aware routing. What do inference platforms put in front of or instead of a plain Service (Gateway API Inference Extension, KServe, llm-d)? (general knowledge)
- On bare-metal GPU clouds, who implements `type: LoadBalancer` when there is no hyperscaler load balancer (e.g. MetalLB, provider-built controllers)? (general knowledge)
- How do readiness probes keep a Service from sending traffic to an inference Pod that is still loading weights?
- Next tutorial module: "Running Multiple Instances of Your App" (scaling a Deployment behind a Service). Ingested as [[2026-11-23-k8s-running-multiple-instances-of-your-app]].
