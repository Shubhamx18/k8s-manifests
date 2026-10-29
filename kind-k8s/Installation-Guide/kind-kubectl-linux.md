<h1 align="center">kind & kubectl Installation Guide (Linux)</h1>

<p align="center">
  <img src="https://img.shields.io/badge/OS-Linux-blue?style=for-the-badge&logo=linux&logoColor=white"/>
  <img src="https://img.shields.io/badge/Kubernetes-kind-success?style=for-the-badge&logo=kubernetes&logoColor=white"/>
</p>

This guide helps you install **kubectl** and **kind** (Kubernetes IN Docker) on Linux to run a local Kubernetes cluster. kind runs each Kubernetes node as a Docker container, so it starts fast and supports multi-node clusters.

---

# Prerequisites

Before installing kind, you need:

- A Linux system (Ubuntu/Debian used in this guide)
- Internet connection
- sudo privileges
- Docker (kind requires a container runtime)

---

# Step 1 - Install Docker

Update the system:

```bash
sudo apt update && sudo apt upgrade -y
```

Install Docker:

```bash
sudo apt install -y docker.io
```

Enable and start Docker:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Allow your user to run Docker without sudo (log out and back in after this):

```bash
sudo usermod -aG docker $USER
```

Verify Docker:

```bash
docker --version
docker ps
```

---

# Step 2 - Install kubectl

kubectl is the CLI tool used to talk to a Kubernetes cluster.

Download the latest kubectl binary:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

Install it to the system path:

```bash
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

Remove the downloaded file:

```bash
rm kubectl
```

Verify installation:

```bash
kubectl version --client
```

> For ARM64 machines, replace `amd64` with `arm64` in the download URL.

---

# Step 3 - Install kind

Set the kind version. Check the latest version at https://github.com/kubernetes-sigs/kind/releases and update if needed:

```bash
KIND_VERSION=v0.30.0
```

Download kind:

For AMD64 / x86_64:

```bash
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/${KIND_VERSION}/kind-linux-amd64
```

For ARM64:

```bash
[ $(uname -m) = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/${KIND_VERSION}/kind-linux-arm64
```

Make it executable and move it to the system path:

```bash
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

Verify:

```bash
kind version
```

---

# Step 4 - Create a Cluster

Create a single-node cluster with the default name `kind`:

```bash
kind create cluster
```

Create a cluster with a custom name:

```bash
kind create cluster --name dev-cluster
```

The first run downloads the node image, so it may take a few minutes.

---

# Step 5 - Verify the Cluster

Check cluster info:

```bash
kubectl cluster-info --context kind-kind
```

Check nodes:

```bash
kubectl get nodes
```

You should see one node named `kind-control-plane` in `Ready` state.

Check system pods:

```bash
kubectl get pods -A
```

---

# Multi-Node Cluster

Create a file named `kind-config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

Create the cluster using the config:

```bash
kind create cluster --name multi-node --config kind-config.yaml
```

Verify:

```bash
kubectl get nodes
```

You should see one control-plane node and two worker nodes.

---

# Deploy a Sample Application

### Create Deployment

```bash
kubectl create deployment hello-kind --image=nginx
```

### Expose Deployment

```bash
kubectl expose deployment hello-kind --type=ClusterIP --port=80
```

### Check Resources

```bash
kubectl get deployments
kubectl get pods
kubectl get svc
```

### Access the Application

kind does not expose NodePort services on your host by default, so use port-forward:

```bash
kubectl port-forward service/hello-kind 8080:80
```

Visit http://localhost:8080

### Clean Up the Sample App

```bash
kubectl delete service hello-kind
kubectl delete deployment hello-kind
```

---

# Load a Local Docker Image into kind

kind nodes cannot see images on your host automatically. Load them like this:

```bash
docker build -t my-app:1.0 .
kind load docker-image my-app:1.0 --name kind
```

Use the image in a deployment with `imagePullPolicy: IfNotPresent` (or any tag other than `latest`).

---

# Managing kind Clusters

| Command | Purpose |
|---------|---------|
| `kind get clusters` | List all clusters |
| `kind get nodes --name kind` | List nodes of a cluster |
| `kind create cluster --name <name>` | Create a cluster |
| `kind delete cluster --name <name>` | Delete a specific cluster |
| `kind delete clusters --all` | Delete all clusters |
| `kind export kubeconfig --name <name>` | Re-export kubeconfig |
| `kind load docker-image <image>` | Load a local image into the cluster |

---

# Check Current Context

```bash
kubectl config current-context
```

It should show:

```
kind-kind
```

For a custom cluster name, the context is `kind-<name>`. Switch between clusters with:

```bash
kubectl config use-context kind-dev-cluster
```

---

# Troubleshooting

| Problem | Fix |
|---------|-----|
| `permission denied` on Docker socket | Run `sudo usermod -aG docker $USER`, then log out and back in |
| `kind: command not found` | Confirm `/usr/local/bin` is in `PATH` (`echo $PATH`) |
| Cluster creation hangs or fails | Check Docker is running: `sudo systemctl status docker` |
| `kubectl` shows wrong cluster | Run `kubectl config use-context kind-<name>` |

---

<p align="center">
  Your local Kubernetes cluster with kind is now running on Linux.
</p>
