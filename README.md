# Kubernetes Services Practical — Minikube

## 2. Objective

In this project I deployed a two-tier application (frontend + backend) on a local Minikube cluster to practice how Kubernetes Services provide stable networking for Pods. I worked with two Service types — **ClusterIP** for internal-only backend access, and **NodePort** for external frontend access — and explored how Services use label selectors to discover Pods, how they stay stable across Pod restarts and scaling, and how to troubleshoot a Service that loses its Endpoints.

## 3. Architecture

```
MINIKUBE CLUSTER
┌─────────────────────────────────────────────────────┐
│                                                       │
│              NodePort Service                        │
│              frontend-service                        │
│                     │                                │
│          ┌──────────┴──────────┐                     │
│          ▼                     ▼                     │
│   Frontend Pod 1         Frontend Pod 2               │
│                                                       │
│                                                       │
│              ClusterIP Service                       │
│              backend-service                         │
│                     │                                │
│      ┌──────────────┼──────────────┐                 │
│      ▼               ▼               ▼               │
│  Backend Pod 1   Backend Pod 2   Backend Pod 3 ...    │
│                                                       │
└─────────────────────────────────────────────────────┘
```

## 4. Resources Created

| Resource   | Name                 | Type      |
|------------|----------------------|-----------|
| Deployment | backend-deployment   | Deployment |
| Service    | backend-service      | ClusterIP |
| Deployment | frontend-deployment  | Deployment |
| Service    | frontend-service     | NodePort  |

## 5. Commands Used

```bash
# Cluster setup
minikube start
kubectl get nodes

# Backend
kubectl apply -f manifests/backend-deployment.yaml
kubectl apply -f manifests/backend-service.yaml
kubectl get deployments
kubectl get pods
kubectl get services
kubectl describe service backend-service

# ClusterIP testing
kubectl run test-client --image=busybox --restart=Never -- sleep 3600
kubectl exec -it test-client -- sh
#   wget -qO- http://backend-service
kubectl delete pod test-client

# Frontend
kubectl apply -f manifests/frontend-deployment.yaml
kubectl apply -f manifests/frontend-service.yaml
kubectl describe service frontend-service
minikube service frontend-service

# Selector experiment
kubectl get pods --show-labels
kubectl apply -f manifests/frontend-service.yaml   # after editing selector to app=wrong
kubectl describe service frontend-service          # confirm empty Endpoints
kubectl apply -f manifests/frontend-service.yaml   # after reverting to app=frontend

# Pod failure experiment
kubectl get pods -l app=backend
kubectl delete pod <pod-name>
kubectl get pods

# Scaling experiment
kubectl scale deployment backend-deployment --replicas=4
kubectl get pods
kubectl describe service backend-service
```

## 6. ClusterIP Testing

`backend-service` is a ClusterIP Service, so it only has a virtual IP reachable from inside the cluster's network, not from my host machine or browser. To test it, I created a temporary `busybox` Pod (`test-client`) inside the cluster and, from a shell inside that Pod, ran `wget -qO- http://backend-service`. This resolved the Service's DNS name to its ClusterIP and returned the default Nginx welcome page, confirming traffic was reaching the backend Pods without me ever needing to know their individual Pod IPs. I deleted `test-client` afterward to clean up.

## 7. NodePort Testing

`frontend-service` is a NodePort Service, which exposes a static port (in the 30000–32767 range) on every node's IP in addition to a ClusterIP. Running `minikube service frontend-service` opened the correct `<node-ip>:<node-port>` URL in my browser automatically, and the Nginx welcome page loaded — confirming external access was working.

## 8. Selector Experiment

I changed `frontend-service`'s selector from `app=frontend` to `app=wrong` and re-applied it. `kubectl describe service frontend-service` command then showed an empty `Endpoints` list which the Service had no Pods to route traffic to, even though the frontend Pods were still running healthy. This demonstrated that a Service's `selector` and a Pod's `labels` are the *only* link between them: Services don't track Pods by name or Deployment, only by matching label key/value pairs. Reverting the selector back to `app=frontend` and re-applying immediately repopulated the Endpoints.

## 9. Pod Failure Experiment

I deleted one backend Pod directly with `kubectl delete pod <pod-name>`. Kubernetes' ReplicaSet controller (managed by `backend-deployment`) noticed the replica count had dropped below the desired 2 and immediately scheduled a replacement Pod with a new name and new IP. The Service itself never disappeared, and its ClusterIP stayed exactly the same throughout — only the Endpoints list updated to swap the old Pod IP for the new one. This is the core value of Services in production: clients keep using one stable address/DNS name while Kubernetes transparently heals and reschedules the Pods behind it.

## 10. Scaling Experiment

I scaled `backend-deployment` from 2 to 4 replicas with `kubectl scale deployment backend-deployment --replicas=4`. `backend-service` required no changes at all — its Endpoints controller watches for any Pod matching `app=backend` and updated automatically, growing from 2 to 4 endpoints. This shows Services decouple client-facing networking from however many Pods are currently backing a workload.

## 11. Questions and Answers

**Part 4 — Backend Service**
1. ClusterIP of `backend-service`: `10.108.80.35`
2. Selector used: `app=backend`
3. Number of endpoints: `2` (one per backend replica)
4. Pods the endpoints point to: the two `backend-deployment` Pods
5. Why two endpoints: the Deployment runs 2 replicas, and the Service's selector matches both so every matching Pod's IP:port becomes an endpoint

**Part 9 — Frontend NodePort**
1. Service port: `80`
2. TargetPort: `80`
3. NodePort: `30704`
4. Why frontend is reachable externally but backend isn't: NodePort opens a port on every node's actual IP address, so it's reachable from outside the cluster; ClusterIP only creates a virtual IP that's routable exclusively within the cluster's internal network, so `backend-service` can't be reached directly from the host.

**Part 10 — Selector Experiment**
1. What happened to the endpoints: they went to empty/none
2. Why the Service stopped finding the Pods: no running Pod had the label `app=wrong`, so nothing matched the selector
3. Relationship between selector and Pod labels: a Service continuously watches for Pods whose labels match its selector and adds their IP:port as an endpoint — it's a live, label-based link, not a static reference

**Part 11 — Pod Failure**
1. What happened to the deleted Pod: it was terminated and removed
2. Why a new Pod was created: the Deployment's ReplicaSet enforces the desired replica count (2) and immediately schedules a replacement
3. Did the backend Service disappear: no
4. Did the Service's ClusterIP change: no, it stayed the same
5. Why this is useful in production: clients and other services keep talking to one stable address/DNS name even as individual Pods crash, get rescheduled, or move nodes — there's no need to track or hardcode Pod IPs anywhere

**Part 12 — Scaling**
1. Backend Pods now running: 4
2. Endpoints on the Service: 4
3. Did I need to modify the Service: no
4. How the Service knows about new Pods: it continuously matches its selector (`app=backend`) against all Pods in the namespace and updates its Endpoints object automatically as Pods are added or removed

## 12. Screenshots

- `screenshots/kubectl-get-nodes.png` — `kubectl get nodes` showing `Ready`
- `screenshots/backend-deployed.png` — `kubectl apply -f manifests/backend-deployment.yaml `
- `screenshots/backend-service.png` — `kubectl apply -f manifests/backend-service.yaml`, `kubectl get services`, `kubectl describe service backend-service`
- `screenshots/frontend-deployments-and-pod.png` — `kubectl apply -f manifests/frontend-deployment.yaml`, `kubectl get deployments`, `kubectl get pods`
- `screenshots/frontend-service.png` — Nginx welcome page via `frontend-service`
- `screenshots/test-client-pod.png` — `wget` output from inside `test-client`
- `screenshots/selector-app=wrong.png`
- `screenshots/selector-app=frontend.png`
- `screenshots/scaled-backend.png` — 4 endpoints after scaling

## 13. Lessons Learned

1. A Service's connection to Pods is entirely driven by label selectors, not by names or the Deployment that created them — break the label match and the Service goes blind to otherwise-healthy Pods.
2. ClusterIP and NodePort aren't separate mechanisms — NodePort is a superset of ClusterIP (every NodePort Service still has a ClusterIP), with an extra externally-reachable port layered on top.
3. Services give a stable network identity that Pods themselves never have — Pod IPs are ephemeral and change on every restart or reschedule, but the Service's ClusterIP and DNS name don't.
4. Scaling a Deployment doesn't require touching its Service at all — the Endpoints object updates reactively based on the selector, which is what makes horizontal scaling transparent to clients.
5. DNS-based service discovery (`http://backend-service`) removes the need for any application to know or hardcode Pod IPs, which is what makes microservice communication resilient to Pod churn.
