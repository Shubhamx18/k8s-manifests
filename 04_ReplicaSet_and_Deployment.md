# ReplicaSet & Deployment

Pods are temporary. If a Pod crashes, it stays dead. Controllers like ReplicaSet and Deployment watch your Pods and keep them alive automatically.

---

## ReplicaSet

Keeps a fixed number of identical Pods running at all times.

```
Pod crashes → ReplicaSet detects it → Creates a new one automatically
```

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-replicaset
  namespace: myspace
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app         # must match selector above
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
```

**Limitation:** ReplicaSet cannot do rolling updates. If you change the container image, existing Pods are not updated — only new Pods will use the new image. This is why you should always use a Deployment instead.

---

## Deployment

Wraps a ReplicaSet and adds rolling updates and rollback.

```
You manage → Deployment → creates/manages → ReplicaSet → creates/manages → Pods
```

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
      maxUnavailable: 1       # max Pods that can be down during update
      maxSurge: 1             # max extra Pods allowed during update
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
          image: nginx:1.21   # always pin a specific version
          ports:
            - containerPort: 80
```

---

## How a Rolling Update Works

```
Before:  [v1] [v1] [v1]
Step 1:  [v1] [v1] [v2]   ← 1 new Pod brought up
Step 2:  [v1] [v2] [v2]   ← 1 old Pod removed
Step 3:  [v2] [v2] [v2]   ← done, zero downtime
```

---

## Update Strategies

| Type | Behavior | Use When |
|---|---|---|
| `RollingUpdate` | Replaces Pods gradually | Production (default) |
| `Recreate` | Kills all Pods first, then starts new ones | Dev/staging only |

---

## ReplicaSet vs Deployment

| Feature | ReplicaSet | Deployment |
|---|:---:|:---:|
| Keeps Pods running | Yes | Yes |
| Self-healing | Yes | Yes |
| Rolling updates | No | Yes |
| Rollback | No | Yes |
| Use in production | No | Yes |

---

## Commands

```bash
# apply
kubectl apply -f deployment.yaml

# check rollout status (run after every update)
kubectl rollout status deployment/my-deployment -n myspace

# watch pods live
kubectl get pods -n myspace -w

# update image — triggers rolling update
kubectl set image deployment/my-deployment nginx=nginx:1.22 -n myspace

# view revision history
kubectl rollout history deployment/my-deployment -n myspace

# rollback to previous version
kubectl rollout undo deployment/my-deployment -n myspace

# rollback to a specific revision
kubectl rollout undo deployment/my-deployment --to-revision=2 -n myspace

# scale pods
kubectl scale deployment/my-deployment --replicas=5 -n myspace

# describe — shows events and conditions useful for debugging
kubectl describe deployment my-deployment -n myspace

# delete
kubectl delete deployment my-deployment -n myspace
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Using ReplicaSet directly in production | Always use Deployment |
| `image: nginx` with no version tag | Pin a version: `image: nginx:1.21` |
| Not checking rollout status after update | Always run `kubectl rollout status` |
| Manually deleting Pods to "restart" them | Scale down via Deployment instead |

---

## What to Learn Next

| Resource | What It Adds |
|---|---|
| `StatefulSet` | Like Deployment, but for stateful apps (databases) |
| `DaemonSet` | Runs exactly one Pod on every node |
| `HPA` | Auto-scales Pods based on CPU/memory |
| Probes | Tells Kubernetes when a Pod is ready and healthy |
