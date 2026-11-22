---
type: entity
tags: [lifecycle]
sources: [2026-11-22-k8s-viewing-pods-and-nodes, 2026-11-22-k8s-using-a-service-to-expose-your-app]
created: 2026-11-22
updated: 2026-11-22
---

# kubectl (cheat sheet)

**kubectl** is the command-line client for [[kubernetes]]. Every command becomes a request to the [[control-plane]]'s API server: list these objects, describe that one, show its logs, run a command inside it. Pronounced "cube-control" or "cube-cuddle". This page is a running cheat sheet; commands are added as sources cover them. (src for role: [[2026-11-22-k8s-viewing-pods-and-nodes]]; API-server framing and pronunciation: general knowledge)

## Details

### The four troubleshooting verbs

Together they answer "when was it deployed, what is its status, where is it running, how is it configured?" (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])

| Command | What it does | Example |
| --- | --- | --- |
| `kubectl get` | List resources | `kubectl get pods` |
| `kubectl describe` | Detailed info on a resource, including lifecycle **events** | `kubectl describe pods` |
| `kubectl logs` | Print a container's logs | `kubectl logs $POD_NAME` |
| `kubectl exec` | Run a command inside a container | `kubectl exec "$POD_NAME" -- env` |

(src: [[2026-11-22-k8s-viewing-pods-and-nodes]]; the `logs` example is general knowledge, the page lists the verb only)

### Commands used in the tutorial

All from the source (src: [[2026-11-22-k8s-viewing-pods-and-nodes]]):

```shell
kubectl get pods                       # list Pods; wait until one shows Running
kubectl describe pods                  # containers, IP, ports, lifecycle events
kubectl proxy                          # in a 2nd terminal: local tunnel to the API on :8001

# save the Pod name in a variable (go-template output)
export POD_NAME="$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')"

# reach the app inside the Pod through the API server proxy
curl http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME:8080/proxy/

kubectl exec "$POD_NAME" -- env        # run one command in the container
kubectl exec -ti $POD_NAME -- bash     # interactive shell in the container; `exit` to leave
```

### Services and labels (Module 4)

All from the source (src: [[2026-11-22-k8s-using-a-service-to-expose-your-app]]):

```shell
kubectl get services                                   # list Services
kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080   # create a Service for a Deployment
kubectl describe services/kubernetes-bootcamp          # cluster IP, ports, assigned NodePort

# save the assigned NodePort in a variable
export NODE_PORT="$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')"
curl http://"$(minikube ip):$NODE_PORT"                # reach the app from outside the cluster
minikube service kubernetes-bootcamp --url             # minikube + Docker Desktop driver: tunnel instead

kubectl describe deployment                            # shows the label the Deployment put on its Pods
kubectl get pods -l app=kubernetes-bootcamp            # filter by label (-l = label selector)
kubectl get services -l app=kubernetes-bootcamp
kubectl label pods "$POD_NAME" version=v1              # add a label to a running Pod
kubectl get pods -l version=v1

kubectl delete service -l app=kubernetes-bootcamp      # delete by label; the Pods keep running
kubectl exec -ti $POD_NAME -- curl http://localhost:8080   # app still answers from inside the Pod
```

- `-l` takes a [[kubernetes-labels-and-selectors|label selector]] and works with `get`, `delete`, and most other verbs. (src for get/delete: [[2026-11-22-k8s-using-a-service-to-expose-your-app]]; "most other verbs": general knowledge)
- `kubectl expose` is a shortcut that builds a [[kubernetes-service|Service]] from an existing Deployment's labels and ports. Production setups usually write the Service as YAML and `kubectl apply` it. (general knowledge)

### Usage notes

- `describe` works on most objects: Nodes, Pods, Deployments. Its output is meant for humans, not for scripts. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]]) For scripts, use `-o json`, `-o yaml`, or `-o go-template` as in the `POD_NAME` example. (`-o go-template`: src; json/yaml: general knowledge)
- With only one container in a Pod, `exec` (and `logs`) need no container name. With several, add `-c <container>`. (first sentence: src; `-c` flag: general knowledge)
- `kubectl proxy` is needed because Pods sit on an isolated, private cluster network. (src: [[2026-11-22-k8s-viewing-pods-and-nodes]])
- In `exec`, `--` separates kubectl's own flags from the command to run in the container; `-ti` means interactive with a terminal. (general knowledge)

### GPU-node triage additions

General knowledge, not from the source; useful on GPU clusters.

| Command | Use |
| --- | --- |
| `kubectl get nodes` | Node list and Ready/NotReady status |
| `kubectl describe node <node>` | Capacity vs allocated, including `nvidia.com/gpu`; taints; recent events |
| `kubectl get pods -o wide` | Which Node each Pod is on |
| `kubectl get pods -n <namespace>` / `-A` | Pods in one namespace / all namespaces |
| `kubectl cordon <node>` / `kubectl drain <node>` | Stop new Pods on a Node / evict its Pods before repair |
| `kubectl exec <pod> -- nvidia-smi` | GPU state as seen from inside a Pod ([[nvidia-smi]]) |

## Related

- [[kubernetes]]: the system it controls.
- [[kubernetes-pod]], [[kubernetes-node]], [[kubernetes-deployment]], [[kubernetes-service]]: the objects these commands act on.
- [[control-plane]]: the API server kubectl talks to.
- [[nvidia-smi]]: the GPU-side counterpart for triage.
