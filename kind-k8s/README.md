# ☸️ Kubernetes Cheatsheet

> Essential commands for container orchestration and cluster management.

---

## 📚 Table of Contents

| # | Section |
|---|---------|
| 01 | [🖥️ Cluster Info & Basics](#-cluster-info--basics) |
| 02 | [📦 Pod Management](#-pod-management) |
| 03 | [🚀 Deployments & ReplicaSets](#-deployments--replicasets) |
| 04 | [🌐 Services & Networking](#-services--networking) |
| 05 | [🔧 ConfigMaps & Secrets](#-configmaps--secrets) |
| 06 | [💾 Storage & Volumes](#-storage--volumes) |
| 07 | [🔍 Troubleshooting & Debugging](#-troubleshooting--debugging) |
| 08 | [📋 Resource Management](#-resource-management) |
| 09 | [⚙️ Advanced Operations](#-advanced-operations) |
| 10 | [📊 Performance & Monitoring](#-performance--monitoring) |
| 11 | [🗂️ Context & Config Management](#-context--config-management) |
| 12 | [✅ Best Practices & Tips](#-best-practices--tips) |
| 13 | [🧠 Real-World Scenario: Exposing an App 3 Ways](#-real-world-scenario-exposing-an-app-3-ways) |

---

## 🖥️ Cluster Info & Basics

> View cluster state, nodes, and namespaces.

```bash
# Cluster
kubectl cluster-info
kubectl config view
kubectl api-resources
kubectl api-versions

# Nodes
kubectl get nodes
kubectl get nodes -o wide
kubectl describe node <node-name>
kubectl top nodes

# Namespaces
kubectl get namespaces
kubectl create namespace my-namespace
kubectl delete namespace my-namespace
kubectl get all -n my-namespace
```

---

## 📦 Pod Management

> Create, inspect, execute, and delete pods.

### ▶️ Create & Run

```bash
kubectl run nginx --image=nginx
kubectl create -f pod.yaml
kubectl run busybox --image=busybox -- echo "Hello World"
kubectl create job hello --image=busybox:1.28 -- echo "Hello World"
```

### 🔎 List & Inspect

```bash
kubectl get pods
kubectl get pods -o wide
kubectl get pods --all-namespaces
kubectl get pods --watch

kubectl describe pod <pod-name>
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name>
kubectl logs -f <pod-name>
```

### ⚡ Execute & Delete

```bash
kubectl exec -it <pod-name> -- /bin/bash
kubectl exec -it <pod-name> -c <container-name> -- sh

kubectl delete pod <pod-name>
kubectl delete pod <pod-name> --grace-period=0 --force
```

---

## 🚀 Deployments & ReplicaSets

> Deploy, scale, update, and rollback applications.

### ➕ Create

```bash
kubectl create deployment nginx --image=nginx
kubectl create deployment webapp --image=nginx --replicas=3
kubectl apply -f deployment.yaml
kubectl expose deployment nginx --port=80 --type=LoadBalancer
```

### 🛠️ Manage

```bash
kubectl get deployments
kubectl describe deployment <name>
kubectl edit deployment <name>
kubectl delete deployment <name>
```

### 📈 Scale

```bash
kubectl scale deployment nginx --replicas=5
kubectl scale rs <replicaset-name> --replicas=3
kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=80
```

### 🔄 Rolling Updates & Rollbacks

```bash
kubectl rollout status deployment/nginx
kubectl rollout history deployment/nginx
kubectl rollout undo deployment/nginx
kubectl rollout undo deployment/nginx --to-revision=2
```

---

## 🌐 Services & Networking

> Expose and connect applications inside and outside the cluster.

### 📌 Service Types

| Type | Access | Use Case |
|------|--------|----------|
| `ClusterIP` | Internal only | Default, pod-to-pod communication |
| `NodePort` | External via node IP | Dev/testing, static port per node |
| `LoadBalancer` | External via cloud LB | Production, cloud providers |
| `ExternalName` | DNS alias | Map to external service by name |

### 🔌 Expose

```bash
kubectl expose deployment nginx --port=80                   # ClusterIP
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl expose deployment nginx --port=80 --type=LoadBalancer
kubectl apply -f service.yaml
```

### 🔎 Inspect

```bash
kubectl get services
kubectl get svc -o wide
kubectl describe service <name>
kubectl get endpoints
```

### 🔁 Port Forwarding

```bash
kubectl port-forward pod/<pod-name> 8080:80
kubectl port-forward svc/<service-name> 8080:80
kubectl port-forward deployment/<name> 8080:80
kubectl port-forward pod/<pod-name> 8080:80 8443:443
```

### 🚪 Ingress

```bash
kubectl get ingress
kubectl describe ingress <name>
kubectl apply -f ingress.yaml
```

---

## 🔧 ConfigMaps & Secrets

> Manage configuration and sensitive data separately from app code.

### 🗃️ ConfigMaps

```bash
# Create
kubectl create configmap app-config --from-literal=db_url=localhost --from-literal=debug=true
kubectl create configmap app-config --from-file=app.properties
kubectl create configmap app-config --from-file=config/

# Manage
kubectl get configmaps
kubectl describe configmap app-config
kubectl get configmap app-config -o yaml
kubectl edit configmap app-config
kubectl delete configmap app-config
```

### 🔐 Secrets

```bash
# Create
kubectl create secret generic db-secret --from-literal=username=admin --from-literal=password=secret123
kubectl create secret generic ssl-certs --from-file=tls.crt --from-file=tls.key
kubectl create secret docker-registry my-registry \
  --docker-server=myregistry.com \
  --docker-username=user \
  --docker-password=pass

# Manage
kubectl get secrets
kubectl describe secret db-secret
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
kubectl delete secret db-secret
```

---

## 💾 Storage & Volumes

> Manage persistent storage for stateful applications.

### 📀 Persistent Volumes

```bash
kubectl get pv
kubectl describe pv <pv-name>
kubectl apply -f persistent-volume.yaml
kubectl delete pv <pv-name>
```

### 📋 Persistent Volume Claims

```bash
kubectl get pvc
kubectl describe pvc <pvc-name>
kubectl apply -f pvc.yaml
kubectl delete pvc <pvc-name>
```

### 🗄️ Storage Classes & Volume Info

```bash
kubectl get storageclass
kubectl describe storageclass <name>

kubectl describe pod <pod-name> | grep -A5 "Mounts:"
kubectl get pod <pod-name> -o yaml | grep -A10 "volumes:"
```

---

## 🔍 Troubleshooting & Debugging

> Diagnose and fix issues across pods, nodes, and services.

### 📜 Logs & Events

```bash
kubectl logs <pod-name>
kubectl logs -f <pod-name>
kubectl logs --previous <pod-name>
kubectl logs <pod-name> -c <container-name>

kubectl get events --sort-by=.metadata.creationTimestamp
```

### 🧾 Describe Resources

```bash
kubectl describe pod <pod-name>
kubectl describe deployment <name>
kubectl describe service <name>
kubectl describe node <node-name>
```

### 📊 Resource Usage

```bash
kubectl top nodes
kubectl top pods
kubectl top pods -n <namespace>
kubectl top pods --sort-by=cpu
```

### 🐛 Interactive Debugging

```bash
kubectl exec -it <pod-name> -- /bin/bash
kubectl debug -it <pod-name> --image=busybox        # K8s 1.23+
kubectl cp <pod-name>:/path/to/file ./local-file
kubectl cp ./local-file <pod-name>:/path/to/destination
```

---

## 📋 Resource Management

> Apply, update, inspect, and validate Kubernetes resources.

### ✅ Apply & Delete

```bash
kubectl apply -f deployment.yaml
kubectl apply -f deployment.yaml -f service.yaml
kubectl apply -f ./k8s-configs/
kubectl apply -f https://example.com/manifest.yaml
kubectl apply -f deployment.yaml --dry-run=client -o yaml

kubectl delete -f deployment.yaml
kubectl delete pod,service -l app=nginx
```

### 🔎 Get & Inspect

```bash
kubectl get all
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase
kubectl get deployment nginx -o yaml
kubectl get pod <name> -o json
```

### ✏️ Edit & Patch

```bash
kubectl edit deployment <name>
kubectl patch deployment nginx -p '{"spec":{"replicas":3}}'
kubectl patch pod <name> --type='json' -p='[{"op":"replace","path":"/metadata/labels/env","value":"prod"}]'
kubectl replace -f updated-deployment.yaml
```

### 🧪 Validate

```bash
kubectl diff -f deployment.yaml
kubectl explain pod.spec.containers
kubectl explain deployment --recursive
kubectl apply -f deployment.yaml --dry-run=client --validate=true
```

---

## ⚙️ Advanced Operations

> Node management, labels, auth, and utility commands.

### 🖧 Node Management

```bash
kubectl cordon <node-name>               # Mark unschedulable
kubectl uncordon <node-name>             # Mark schedulable
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

kubectl taint nodes <node-name> key=value:NoSchedule
kubectl taint nodes <node-name> key:NoSchedule-
```

### 🏷️ Labels & Annotations

```bash
kubectl label pod <pod-name> environment=production
kubectl label pod <pod-name> environment-

kubectl annotate pod <pod-name> description="Frontend web server"

kubectl get pods -l environment=production
kubectl get pods -l 'environment in (production,staging)'
```

### 🔑 Auth & Proxy

```bash
kubectl proxy --port=8080
kubectl auth can-i create pods
kubectl auth can-i '*' '*' --as=system:serviceaccount:default:my-sa
kubectl get pods --as=system:serviceaccount:default:my-sa
kubectl config view --raw -o jsonpath='{.users[*].name}'
```

### 🛠️ Utility

```bash
kubectl wait --for=condition=Ready pod/<pod-name> --timeout=300s
kubectl run tmp-pod --rm -i --tty --image=busybox -- /bin/sh
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml
kubectl get pods --sort-by=.metadata.creationTimestamp
```

---

## 📊 Performance & Monitoring

> Monitor resource usage, health, and cluster performance.

### 📈 Resource Metrics

```bash
kubectl top nodes --sort-by=cpu
kubectl top nodes --sort-by=memory
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory -A
kubectl top pods --containers=true
```

### 💚 Health & Status

```bash
kubectl rollout status deployment/<name>
kubectl get pods --field-selector=status.phase=Running
kubectl get resourcequota
kubectl describe resourcequota
kubectl get componentstatuses
```

### 💡 Optimization & Backup

```bash
kubectl describe node | grep -A5 "Allocated resources:"
kubectl get pdb
kubectl get hpa
kubectl get networkpolicy

# Backup namespace resources
kubectl get all -o yaml -n <namespace> > backup.yaml
kubectl get deployment -o yaml > deployment-backup.yaml
```

---

## 🗂️ Context & Config Management

> Switch clusters, manage kubeconfig, and set defaults.

```bash
# Context
kubectl config current-context
kubectl config get-contexts
kubectl config use-context <name>
kubectl config set-context dev-context --cluster=dev-cluster --user=dev-user --namespace=development

# Kubeconfig
kubectl config view
kubectl config set-cluster <name> --server=https://api-url --certificate-authority=/path/to/ca.crt
kubectl config set-credentials <name> --client-certificate=/path/to/client.crt --client-key=/path/to/key

# Merge configs
KUBECONFIG=~/.kube/config:~/.kube/config2 kubectl config view --merge --flatten > ~/.kube/merged-config

# Defaults
kubectl config set-context --current --namespace=<namespace>
kubectl config view -o jsonpath='{.users[*].name}'
```

---

## ✅ Best Practices & Tips

> Shortcuts, selectors, and safe habits for daily use.

### ⚡ Aliases

```bash
alias k=kubectl
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
```

### 📝 Short Resource Names

| Short | Full Name |
|-------|-----------|
| `po` | pods |
| `svc` | services |
| `deploy` | deployments |
| `ns` | namespaces |
| `no` | nodes |
| `pv` | persistentvolumes |
| `pvc` | persistentvolumeclaims |
| `cm` | configmaps |
| `sa` | serviceaccounts |

```bash
kubectl get po
kubectl get svc
kubectl get deploy
kubectl get ns
kubectl get no
```

### 🏷️ Label Selectors

```bash
kubectl get pods -l app=nginx
kubectl get pods -l 'environment in (prod,staging)'
kubectl get pods -l app=nginx,version!=v1.0
kubectl get pods --field-selector=status.phase=Running
kubectl get pods --field-selector=spec.nodeName=worker-node-1
```

---

## 🧠 Real-World Scenario: Exposing an App 3 Ways

> **Goal:** Deploy an app inside a Kubernetes cluster (KIND / EC2 / EKS) and expose it using all 3 service types — so you understand *when*, *why*, and *how* each one works.

---

### 🗺️ Quick Reference — Which Service Should I Use?

| Service Type | Who Can Access | When to Use |
|---|---|---|
| `ClusterIP` | Only pods inside the cluster | Microservice-to-microservice (frontend → backend → DB) |
| `NodePort` | Anyone who can reach the node's IP | Local testing on KIND, Minikube, or EC2 |
| `LoadBalancer` | The public internet | Real production on AWS / GCP / Azure |

---

### Step 1 — Create a Namespace

> A **namespace** is like a folder inside your cluster. It isolates your app's resources (pods, services, configs) from everything else running in the cluster.
> Best practice: always deploy into a named namespace, never `default`.

```bash
kubectl create namespace myspace
```

---

### Step 2 — Deploy the App

> A **Deployment** tells Kubernetes: *"Keep this container running. If it crashes, restart it. If I ask for 3 replicas, maintain 3."*
> Kubernetes will schedule a **Pod** (the running container) on one of the nodes.

```bash
kubectl create deployment myapp \
  --image=shubhamm18/portfolio:03 \
  -n myspace
```

> Wait for the pod to reach `Running` state before proceeding:

```bash
kubectl get pods -n myspace
```

---

### 🔵 Step 3 — ClusterIP (Internal Only)

> **What it does:** Creates a stable internal IP + DNS name for your service *inside* the cluster. Nothing outside the cluster can reach it directly.
>
> **Why it exists:** Pods are temporary — they get new IPs every time they restart. ClusterIP gives your service a permanent internal address so other pods always know where to find it, even when pods come and go.
>
> **When to use:** Whenever services talk to each other internally. Your frontend calls your backend via ClusterIP. Your backend calls your database via ClusterIP. This is the most common service type in real production systems.

```bash
kubectl expose deployment myapp \
  --type=ClusterIP \
  --port=3000 \
  --target-port=3000 \
  -n myspace

kubectl get svc -n myspace
# You'll see a CLUSTER-IP like 10.96.x.x — only reachable from inside the cluster
```

> **Problem:** ClusterIP is not reachable from your browser. To test it, use `port-forward` — this creates a temporary tunnel from your laptop into the cluster:

```bash
kubectl port-forward svc/myapp 8080:3000 -n myspace
# Now open: http://localhost:8080
```

> `port-forward` is for debugging only. It only works while that terminal is open — it's not a real deployment strategy.

**Key idea:** `ClusterIP` = private service. Pods talk to each other through it. The outside world never sees it directly.

---

### 🟢 Step 4 — NodePort (External via Node IP)

> **What it does:** Opens a port directly on every node in your cluster (in the range 30000–32767). Anyone who can reach that node's IP address can hit your app on that port — no port-forward needed.
>
> **Why it exists:** Sometimes you're on a plain EC2 instance or a local KIND cluster where there's no cloud load balancer available. NodePort lets you expose your app using the machine's own IP.
>
> **When to use:** Local and EC2 testing. When you need real browser access but don't have a cloud provider. **Not for production** — it's not scalable, the ports are ugly, and you're directly exposing node IPs.

```bash
# Delete the previous service first — same name = conflict
kubectl delete svc myapp -n myspace

kubectl expose deployment myapp \
  --type=NodePort \
  --port=3000 \
  --target-port=3000 \
  -n myspace

kubectl get svc -n myspace
# Look under PORT(S) — you'll see something like: 3000:31245/TCP
# That 31245 is your NodePort
```

> Access your app:
> - **KIND / Minikube (local):** `http://localhost:<NodePort>`
> - **EC2:** `http://<EC2-PUBLIC-IP>:<NodePort>` *(open the port in your Security Group first)*

**Key idea:** `NodePort` = opens a port on the machine itself. No port-forward needed. Still not production-ready.

---

### 🟡 Step 5 — LoadBalancer (Real Production)

> **What it does:** Talks to your cloud provider (AWS, GCP, Azure) and automatically provisions a real public load balancer with a public IP. Traffic from the internet hits the LB, which routes it to your pods.
>
> **Why it exists:** You don't want to expose individual node IPs to users. A load balancer is the standard, scalable, fault-tolerant way to accept public traffic. If a node goes down, the LB automatically stops sending traffic to it.
>
> **When to use:** Any time you're on a real cloud provider and need public internet access.

```bash
# Delete the previous service first
kubectl delete svc myapp -n myspace

kubectl expose deployment myapp \
  --type=LoadBalancer \
  --port=80 \
  --target-port=3000 \
  -n myspace

kubectl get svc -n myspace
# Watch the EXTERNAL-IP column — starts as <pending>, fills in once the cloud LB is ready
```

> Access your app:
> - **Cloud (AWS/GCP/Azure):** `http://<EXTERNAL-IP>`
> - **KIND / local:** `EXTERNAL-IP` stays `<pending>` forever — that's normal. LoadBalancer requires a real cloud provider to work.

**Key idea:** `LoadBalancer` = the internet-facing gate into your cluster. Kubernetes asks the cloud to create a real LB automatically.

---

### 🔥 Real Production Traffic Flow

In real companies, the full flow looks like this:

```
Internet
    │
    ▼
LoadBalancer              ← One public IP / entry point (AWS ALB, GCP LB, etc.)
    │
    ▼
Ingress Controller        ← Routes by hostname or URL path (e.g. nginx-ingress)
    │
    ├──▶ /api     →  ClusterIP (backend-svc)   →  Pods
    ├──▶ /        →  ClusterIP (frontend-svc)  →  Pods
    └──▶ /admin   →  ClusterIP (admin-svc)     →  Pods
```

**Why this pattern and not just LoadBalancer per service?**

- A cloud LoadBalancer costs money — you don't want one per service
- **Ingress** does smart routing by path/hostname using one LB
- All internal services use **ClusterIP** — never exposed directly
- **NodePort is skipped entirely** in production

---

### 🧹 Cleanup

```bash
kubectl delete namespace myspace
# Deletes the namespace AND everything inside it (deployment, pods, services, etc.)
```

---

> Refer to the [official Kubernetes documentation](https://kubernetes.io/docs/) for deeper reference.
