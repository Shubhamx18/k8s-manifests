# ☸️ Kubernetes Workloads — ReplicaSet & Deployment

---

## Why Controllers Exist

Pods are temporary. If a Pod crashes, it stays dead.
Controllers like **ReplicaSet** and **Deployment** watch your Pods and keep them alive automatically.

---

## 1. ReplicaSet

Keeps a fixed number of identical Pods running at all times.

```
Pod crashes → ReplicaSet detects it → Creates a new one automatically
```

### YAML

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-replicaset
  namespace: myspace
spec:
  replicas: 3                   # always keep 3 Pods running
  selector:
    matchLabels:
      app: demo-app             # manage Pods with this label
  template:
    metadata:
      labels:
        app: demo-app           # label on each Pod (must match selector)
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
```

### Commands

```bash
kubectl apply -f replicaset.yaml
kubectl get pods -n myspace
kubectl edit replicaset my-replicaset -n myspace
kubectl delete replicaset my-replicaset -n myspace
```

### ⚠️ Limitation
ReplicaSet **cannot do updates**. If you change the image, existing Pods are not updated — only new Pods will use the new image.

---

## 2. Deployment

Wraps a ReplicaSet and adds **rolling updates + rollback**.

```
You manage → Deployment → creates/manages → ReplicaSet → creates/manages → Pods
```

### YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  namespace: myspace
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1         # max Pods that can be down during update
      maxSurge: 1               # max extra Pods allowed during update
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
        - name: nginx
          image: nginx:1.21     # always pin a specific version
          ports:
            - containerPort: 80
```

### How Rolling Update Works

```
Before:  [v1] [v1] [v1]
Step 1:  [v1] [v1] [v2]   ← 1 new Pod added
Step 2:  [v1] [v2] [v2]   ← 1 old Pod removed
Step 3:  [v2] [v2] [v2]   ← done, zero downtime
```

### Update Strategy

| Type | Behavior | Use When |
|---|---|---|
| `RollingUpdate` | Replaces Pods gradually | Production (default) |
| `Recreate` | Kills all Pods, then starts new ones | Dev/staging only |

---

## 3. ReplicaSet vs Deployment

| Feature | ReplicaSet | Deployment |
|---|:---:|:---:|
| Keeps Pods running | ✅ | ✅ |
| Self-healing | ✅ | ✅ |
| Rolling updates | ❌ | ✅ |
| Rollback | ❌ | ✅ |
| Use in production | ❌ | ✅ |

---

## 4. Commands — Deployment

```bash
# Create / update
kubectl apply -f deployment.yaml

# Check status (run after every update)
kubectl rollout status deployment/my-deployment -n myspace

# Watch Pods live
kubectl get pods -n myspace -w

# Update image (triggers rolling update)
kubectl set image deployment/my-deployment nginx=nginx:1.21 -n myspace

# View revision history
kubectl rollout history deployment/my-deployment -n myspace

# Rollback to previous version
kubectl rollout undo deployment/my-deployment -n myspace

# Rollback to a specific revision
kubectl rollout undo deployment/my-deployment --to-revision=2 -n myspace

# Scale Pods
kubectl scale deployment/my-deployment --replicas=5 -n myspace

# Describe (debug events and conditions)
kubectl describe deployment my-deployment -n myspace

# Delete
kubectl delete deployment my-deployment -n myspace
```

---

## 5. Common Mistakes

| ❌ Mistake | ✅ Fix |
|---|---|
| Using ReplicaSet directly in production | Always use Deployment |
| `image: nginx` (no version tag) | Use `image: nginx:1.21` |
| Not checking rollout after update | Always run `kubectl rollout status` |
| Manually deleting Pods | Scale down via Deployment instead |

---

## What to Learn Next

| Topic | One Line |
|---|---|
| `StatefulSet` | Like Deployment, but for stateful apps (DBs) |
| `DaemonSet` | Runs one Pod on every node |
| `HPA` | Auto-scales Pods based on CPU/memory |
| `Probes` | Tells K8s when a Pod is healthy or not |
