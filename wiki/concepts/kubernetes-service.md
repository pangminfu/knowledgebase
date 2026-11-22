---
type: concept
tags: [provisioning, lifecycle]
sources: [2026-11-22-k8s-using-a-service-to-expose-your-app, 2026-11-23-k8s-running-multiple-instances-of-your-app]
created: 2026-11-22
updated: 2026-11-23
---

# Service (Kubernetes)

A **Service** is a [[kubernetes]] object that gives a changing group of [[kubernetes-pod|Pods]] one stable address. It defines a logical set of Pods, usually chosen by a [[kubernetes-labels-and-selectors|label selector]], and a policy for reaching them: load balancing across them, discovery by name, and optional exposure outside the cluster. Analogy: a restaurant's phone number. The staff who answer change every shift, but customers always dial the same number and someone available picks up. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]]; analogy: general knowledge)

![[raw/assets/k8s-expose-labels.svg]]
Service `svc` (1.1.1.1) with selector `app=A` routes to all four Pods labeled `app: A`: three managed by a Deployment/ReplicaSet and one standalone Pod. The `app: B` Pod is not selected. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])

## Details

### Why it exists

- Pods are mortal. When a Node dies, its Pods are lost and replaced by new Pods with new IPs. Every Pod has a unique IP, so clients cannot hard-code Pod addresses. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])
- A Service gives loose coupling between dependent Pods: a frontend talks to the backend Service and does not care how many backend replicas exist or whether one was recreated. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])
- Pod IPs are not reachable from outside the cluster without a Service. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])

### Types: a ladder of exposure

| `spec.type` | Reachable from | Mechanism | Notes |
| --- | --- | --- | --- |
| ClusterIP (default) | Inside the cluster | Internal virtual IP | (src) |
| NodePort | Outside, `NodeIP:NodePort` | Same port opened on each selected Node via NAT | Superset of ClusterIP (src). Default port range 30000–32767 (general knowledge) |
| LoadBalancer | Outside, fixed external IP | The cloud provisions an external load balancer, if supported | Superset of NodePort (src) |
| ExternalName | n/a | DNS `CNAME` to `externalName`; no proxying | No selector (src) |

(src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]] unless marked)

```mermaid
flowchart LR
  C["External client"] --> LB["Cloud load balancer<br/>(LoadBalancer)"]
  LB --> NP["NodeIP:NodePort<br/>on every Node (NodePort)"]
  NP --> CIP["Cluster IP<br/>(ClusterIP)"]
  IC["In-cluster client"] --> CIP
  CIP --> P1["Pod"]
  CIP --> P2["Pod"]
  CIP --> P3["Pod"]
```

Each type builds on the one below it, so a LoadBalancer Service also has a NodePort and a ClusterIP. (superset relation: src; diagram: this wiki)

### Selectors and endpoints

- The set of Pods is usually defined by a label selector. Any Pod whose labels match receives traffic. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])
- A Service **without a selector** gets no Endpoints object, so you can point it by hand at specific backends. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]]) Newer Kubernetes versions track backends in **EndpointSlices**, the scalable successor to Endpoints. (general knowledge)
- **Endpoints track the replicas.** A Service continuously monitors the running Pods through endpoints, so traffic only goes to available Pods. When a Deployment is [[kubernetes-scaling|scaled]], new Pods are added and terminated ones removed, and the Service itself does not change. In the tutorial, after scaling to 4, repeated `curl`s reach a different Pod each time. (src: [[2026-11-23-k8s-running-multiple-instances-of-your-app]]) Each `curl` opens a new connection, so this fits per-connection balancing. (general knowledge)
- **Discovery:** cluster DNS gives each Service a name such as `my-svc.my-namespace.svc.cluster.local`, so clients look it up by name instead of by IP. (general knowledge; the source only says Services provide "service discovery")
- **How traffic is forwarded:** a component on each Node (kube-proxy, using iptables or IPVS rules) rewrites traffic for the cluster IP to one of the backend Pod IPs. Balancing happens per connection (layer 4), not per HTTP request. (general knowledge)
- **Readiness:** only Pods that pass their readiness probe are listed as backends, so a Pod that is starting up gets no traffic. (general knowledge)

### Lifecycle is separate from the workload

- `kubectl expose deployment/<name> --type=NodePort --port 8080` creates a Service for an existing [[kubernetes-deployment|Deployment]]. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])
- Deleting the Service removes outside access, but the Pods keep running because the Deployment still manages them. (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]])

## GPU cloud bridge

All general knowledge; not from the source.

- **An inference endpoint's address is a Service** (or something built on one) in front of the Deployment of replica Pods ([[inference-serving]]). Autoscaling and Node failures change the Pods; the URL customers hold does not change.
- **Readiness is the cold-start guard.** An inference Pod can take minutes to load weights ([[kubernetes-pod]]). Its readiness probe should fail until the model is loaded, or the Service will send requests to a replica that cannot answer.
- **Layer-4 balancing is a poor fit for LLMs.** Requests vary hugely in length and are streamed, and replicas hold different [[kv-cache|KV-cache]] prefixes. Per-connection round-robin ignores all of this, so inference platforms add request-level, cache-aware routing on top, e.g. the Gateway API Inference Extension ([[inference-optimization]]).
- **LoadBalancer on bare metal.** Hyperscalers fulfil `type: LoadBalancer` with their own load balancers. A bare-metal GPU cloud needs its own implementation (MetalLB, or a provider-built controller), which belongs to its provisioning and networking stack.
- **Tenancy:** Services are namespaced. A multi-tenant platform also needs network policy, so tenant A's Pods cannot reach tenant B's Services just by knowing the DNS name.
- **Scale-down race.** Removing a Pod from endpoints and telling it to stop happen at about the same time, so a few new requests can still reach a Pod that is shutting down. Inference Pods usually add a short pre-stop delay and drain in-flight streams before exiting ([[kubernetes-scaling]]).

## Related

- [[kubernetes-labels-and-selectors]]: how a Service picks its Pods.
- [[kubernetes-pod]]: the backends; why their IPs cannot be relied on.
- [[kubernetes-deployment]]: keeps the backends running; separate object from the Service.
- [[kubernetes-scaling]]: changes how many backends exist; endpoints follow automatically.
- [[kubernetes-node]]: where NodePort ports open.
- [[inference-serving]]: load balancing across replicas, one of the serving responsibilities.
- [[kubectl]]: `expose`, `get services`, `describe services`, `delete service`.

## Open questions

- What sits between a customer's HTTPS request and the Service in real inference platforms (ingress, API gateway, auth, rate limiting)?
- How fast does a failed Pod disappear from a Service's backends, and how many requests fail in between?
