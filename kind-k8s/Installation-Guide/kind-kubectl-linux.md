<h1 align="center">kind & kubectl Installation Guide (Linux)</h1>

<p align="center">
  <img src="https://img.shields.io/badge/OS-Linux-blue?style=for-the-badge&logo=linux&logoColor=white"/>
  <img src="https://img.shields.io/badge/Kubernetes-kind-success?style=for-the-badge&logo=kubernetes&logoColor=white"/>
</p>

This guide helps you install **kubectl** and **kind** (Kubernetes IN Docker) on Linux to run a local Kubernetes cluster. kind runs every Kubernetes node as a Docker container, so clusters start in under a minute and you can create multi-node clusters easily.

---

# Prerequisites

- A Linux system (Ubuntu/Debian used in this guide)
- Internet connection
- sudo privileges

---

# Step 1 - Install Docker

kind needs Docker to run its nodes.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io
```

Enable and start Docker:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Allow your user to run Docker without sudo:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Verify:

```bash
docker --version
docker ps
```

---

# Step 2 - Install kubectl

kubectl is the command line tool used to talk to a Kubernetes cluster.

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl
```

Verify:

```bash
kubectl version --client
```

---

# Step 3 - Install kind

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.30.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

Verify:

```bash
kind version
```

> Newer versions may be available on the [kind releases page](https://github.com/kubernetes-sigs/kind/releases). Just change the version number in the URL.

---

# Step 4 - Create Your First Cluster

```bash
kind create cluster --name dev
```

The first run downloads the node image, so it can take a few minutes.

Check that everything is working:

```bash
kubectl cluster-info --context kind-dev
kubectl get nodes
kubectl get pods -A
```

You should see one node named `dev-control-plane` with status `Ready`.

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

Create the cluster:

```bash
kind create cluster --name multi --config kind-config.yaml
```

Verify:

```bash
kubectl get nodes
```

You should see one control-plane node and two worker nodes.

---

# Deploy a Sample Application

Create a deployment with 2 replicas and expose it:

```bash
kubectl create deployment hello-kind --image=nginx --replicas=2
kubectl expose deployment hello-kind --port=80
```

Check the resources:

```bash
kubectl get deployments
kubectl get pods
kubectl get svc
```

Access the app using port-forward:

```bash
kubectl port-forward service/hello-kind 8080:80
```

Open http://localhost:8080 in your browser. Press `Ctrl + C` to stop the forward.

Clean up:

```bash
kubectl delete service hello-kind
kubectl delete deployment hello-kind
```

---

# Use a Local Docker Image

kind nodes cannot see images built on your machine, so load them into the cluster first:

```bash
docker build -t my-app:1.0 .
kind load docker-image my-app:1.0 --name dev
```

Then use `my-app:1.0` in your deployment. Always use a specific tag (not `latest`), otherwise Kubernetes tries to pull it from the internet.

---

# Useful Commands

| Command | Purpose |
|---------|---------|
| `kind get clusters` | List all clusters |
| `kind create cluster --name <name>` | Create a cluster |
| `kind delete cluster --name <name>` | Delete a cluster |
| `kind delete clusters --all` | Delete all clusters |
| `kind load docker-image <image> --name <name>` | Load a local image |
| `kubectl config get-contexts` | List all contexts |
| `kubectl config use-context kind-<name>` | Switch to another cluster |
| `kubectl config current-context` | Show the active context |

The context name for a kind cluster is always `kind-<cluster-name>`, for example `kind-dev`.

---

# Optional: kubectl Shortcuts

Add an alias and tab completion to make kubectl faster to use:

```bash
echo 'alias k=kubectl' >> ~/.bashrc
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc
```

Now you can type `k get pods` instead of `kubectl get pods`.

---

# Troubleshooting

| Problem | Fix |
|---------|-----|
| `permission denied` on Docker socket | Run `sudo usermod -aG docker $USER`, then log out and back in |
| `kind: command not found` | Check that `/usr/local/bin` is in your PATH with `echo $PATH` |
| Cluster creation hangs or fails | Check Docker is running with `sudo systemctl status docker` |
| kubectl talks to the wrong cluster | Run `kubectl config use-context kind-<name>` |

---

<p align="center">
  Your local Kubernetes cluster with kind is now running on Linux.
</p>
