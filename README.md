# ☸️ Kubernetes Cheatsheet

> Essential commands for container orchestration and cluster management.

---

## 📚 Table of Contents

| # | Section |
|---|---------|
| 01 | [🖥️ Cluster Info & Basics](#️-cluster-info--basics) |
| 02 | [📦 Pod Management](#-pod-management) |
| 03 | [🚀 Deployments & ReplicaSets](#-deployments--replicasets) |
| 04 | [🌐 Services & Networking](#-services--networking) |
| 05 | [🔧 ConfigMaps & Secrets](#-configmaps--secrets) |
| 06 | [💾 Storage & Volumes](#-storage--volumes) |
| 07 | [🔍 Troubleshooting & Debugging](#-troubleshooting--debugging) |
| 08 | [📋 Resource Management](#-resource-management) |
| 09 | [⚙️ Advanced Operations](#️-advanced-operations) |
| 10 | [📊 Performance & Monitoring](#-performance--monitoring) |
| 11 | [🗂️ Context & Config Management](#️-context--config-management) |
| 12 | [✅ Best Practices & Tips](#-best-practices--tips) |

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

# Inspect volume mounts in a pod
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

> refer to the [official Kubernetes documentation](https://kubernetes.io/docs/).
