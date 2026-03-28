# Kubernetes Namespaces

Namespaces are logical partitions inside a cluster. They provide separate name scopes, separate access controls, and separate resource quotas. Once you run multiple apps, teams, or environments on the same cluster, namespaces are what keeps everything organized.

---

## Default Namespaces (Shipped with Every Cluster)

| Namespace | Purpose |
|---|---|
| `default` | Where resources land if you don't specify one |
| `kube-system` | Core Kubernetes components (DNS, scheduler, etc.) |
| `kube-public` | Public cluster-wide info, readable by anyone |
| `kube-node-lease` | Node heartbeat and lease tracking |

> Never deploy your apps into `default` or `kube-system`. Always create your own namespace.

---

## What Is and Isn't Namespaced

**Namespaced resources** (scoped to a namespace): Pods, Deployments, Services, StatefulSets, ConfigMaps, Secrets, PVCs.

**Cluster-scoped resources** (exist outside any namespace): Nodes, PersistentVolumes, ClusterRoles, Namespaces themselves.

```bash
# see everything that is namespaced
kubectl api-resources --namespaced=true

# see everything that isn't
kubectl api-resources --namespaced=false
```

---

## Common Namespace Patterns

| Namespace | Use Case |
|---|---|
| `dev` | Development environment |
| `staging` | Pre-prod / QA |
| `prod` | Production |
| `monitoring` | Prometheus, Grafana |
| `logging` | ELK / EFK stack |

> Namespaces don't isolate networking by default. A pod in `dev` can still reach a service in `prod` using its full DNS name. For actual network isolation, use NetworkPolicies.

---

## Create a Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myspace
```

```bash
kubectl apply -f namespace.yaml

# or imperatively
kubectl create namespace myspace
```

Any resource created after this needs `namespace: myspace` in its metadata, or it lands in `default`.

---

## Deployment Scoped to a Namespace

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: myspace
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
```

You can have another `nginx` Deployment in `prod` with zero conflicts — the namespace is part of the resource identity.

---

## ResourceQuota — Limit What a Namespace Can Use

Caps how much CPU and memory the entire namespace can consume. Useful when multiple teams share a cluster.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: myspace-quota
  namespace: myspace
spec:
  hard:
    pods: "10"
    requests.cpu: "2"
    requests.memory: 2Gi
    limits.cpu: "4"
    limits.memory: 4Gi
```

Once the quota is hit, new pods won't schedule until something is freed up.

---

## Commands

```bash
# list all namespaces
kubectl get ns

# get all resources inside a namespace
kubectl get all -n myspace

# set a sticky default so you stop typing -n myspace every command
kubectl config set-context --current --namespace=myspace

# delete a namespace and EVERYTHING inside it
kubectl delete namespace myspace
```

---

## Namespace vs Separate Cluster

| Feature | Namespace | Separate Cluster |
|---|---|---|
| Isolation | Logical | Physical |
| Network isolation | Only with NetworkPolicies | Full by default |
| Cost | Free | Extra infra |
| Resource sharing | Yes | No |
| Typical use | Environments / teams | Org-level split |

---

## Cross-Namespace Service Discovery

Services are reachable across namespaces using the full DNS name:

```
service-name.namespace.svc.cluster.local
```

Short form (`service-name`) only works within the same namespace.

---

## Notes

- Deleting a namespace deletes everything inside it — Deployments, Services, Secrets, PVCs, all of it. Double-check before running `kubectl delete namespace`
- For access control per namespace, pair namespaces with RBAC (RoleBindings scoped to a namespace)
