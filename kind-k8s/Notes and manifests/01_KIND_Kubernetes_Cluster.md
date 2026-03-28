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

## Create a Cluster

```bash
kind create cluster --name my-cluster
```

This pulls the node image, starts a Docker container, bootstraps the control plane, and updates your kubeconfig automatically.

---

## Verify

```bash
kubectl cluster-info
kubectl get nodes
```

Expected output:

```
NAME                       STATUS   ROLES           AGE   VERSION
my-cluster-control-plane   Ready    control-plane   ...   v1.xx.x
```

---

## How It Works Internally

```
Docker Container
└── Kubernetes Node
     ├── kubelet
     ├── kube-proxy
     ├── containerd
     └── control-plane components
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

- Multi-node setup is just a config file away — see KIND docs for `extraMounts` and worker node configuration
- Great for testing Deployments, StatefulSets, Services, Ingress, PVCs, and failure scenarios without any cloud cost
