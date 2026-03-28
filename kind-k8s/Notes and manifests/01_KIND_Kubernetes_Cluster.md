# KIND — Kubernetes IN Docker

KIND runs each Kubernetes node as a Docker container. No VMs, no cloud cost. It's what the Kubernetes project itself uses for testing, and every YAML you write here works identically on EKS, GKE, or any kubeadm cluster.

---

## KIND vs Minikube

| Feature | Minikube | KIND |
|---|---|---|
| Runs on | VM / Docker | Docker only |
| Startup speed | Slower | Faster |
| Multi-node cluster | Limited | Easy |
| CI/CD friendly | No | Yes |
| Production-like behavior | Medium | High |

---

## Prerequisites

- Docker (nodes run as Docker containers — non-negotiable)
- kubectl
- KIND binary

---

## Install KIND (Linux)

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind version
```

---

## Single-Node Cluster (Default)

```bash
kind create cluster --name my-cluster
```

This pulls the node image, starts a Docker container, bootstraps the control plane, and updates your kubeconfig automatically.

---

## Multi-Node Cluster

For a production-like setup with one control plane and multiple workers, use a config file.

**`kind-config.yaml`**

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    image: kindest/node:v1.32.2
  - role: worker
    image: kindest/node:v1.32.2
  - role: worker
    image: kindest/node:v1.32.2
  - role: worker
    image: kindest/node:v1.32.2
```

```bash
kind create cluster --name my-cluster --config kind-config.yaml
```

This gives you 1 control plane + 3 worker nodes — close to what a real EKS or GKE cluster looks like. Useful for testing pod scheduling across nodes, DaemonSets, node affinity, and failure scenarios where a node goes down.

---

## Verify

```bash
kubectl cluster-info
kubectl get nodes
```

Expected output for multi-node:

```
NAME                       STATUS   ROLES           AGE   VERSION
my-cluster-control-plane   Ready    control-plane   ...   v1.32.2
my-cluster-worker          Ready    <none>          ...   v1.32.2
my-cluster-worker2         Ready    <none>          ...   v1.32.2
my-cluster-worker3         Ready    <none>          ...   v1.32.2
```

---

## How It Works Internally

```
Docker Container (per node)
└── Kubernetes Node
     ├── kubelet
     ├── kube-proxy
     ├── containerd
     └── control-plane components  ← only on control-plane node
```

When you run `kubectl apply -f`, Kubernetes processes it exactly the same way as in production. The containerized node is transparent to cluster internals.

---

## Delete a Cluster

```bash
kind delete cluster --name my-cluster
```

Wipes everything — no leftover resources, no stale state.

---

## Notes

- Always pin the `image` version in your config file — `kindest/node:v1.32.2` ensures consistent behavior across machines
- Multi-node clusters let you test real scheduling behavior — without them, every pod lands on the same node
- The same `kubectl` commands and YAML manifests work here as on EKS, GKE, or AKS — no surprises
