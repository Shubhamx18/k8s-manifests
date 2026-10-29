<h1 align="center">kind & kubectl Installation Guide (Windows)</h1>

<p align="center">
  <img src="https://img.shields.io/badge/OS-Windows-blue?style=for-the-badge&logo=windows&logoColor=white"/>
  <img src="https://img.shields.io/badge/Kubernetes-kind-success?style=for-the-badge&logo=kubernetes&logoColor=white"/>
</p>

This guide walks you through installing **kubectl** and **kind** (Kubernetes IN Docker) on Windows to run a local Kubernetes cluster. kind runs each Kubernetes node as a Docker container, so it starts fast and supports multi-node clusters.

---

# Prerequisites

Before installing kind, you must install:

- **Docker Desktop for Windows** (with the WSL 2 backend enabled)
  Download: https://www.docker.com/products/docker-desktop

kind needs a container runtime, and Docker Desktop provides it. Make sure Docker Desktop is **running** before creating a cluster.

Verify Docker in PowerShell:

```powershell
docker --version
docker ps
```

---

# Part 1 - Install kubectl

kubectl is the CLI tool used to talk to a Kubernetes cluster.

### Step 1 - Install Chocolatey (Package Manager)

Open **PowerShell as Administrator** and run:

```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force; `
[System.Net.ServicePointManager]::SecurityProtocol = `
[System.Net.ServicePointManager]::SecurityProtocol -bor 3072; `
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
```

Verify:

```powershell
choco --version
```

---

### Step 2 - Install kubectl

```powershell
choco install kubernetes-cli -y
```

---

### Step 3 - Verify kubectl Installation

Close and reopen PowerShell, then run:

```powershell
kubectl version --client
```

You should see the installed client version.

---

# Part 2 - Install kind

### Option A - Using Chocolatey (Recommended)

```powershell
choco install kind -y
```

### Option B - Manual Binary Download

Check the latest version at https://github.com/kubernetes-sigs/kind/releases and update the version if needed:

```powershell
curl.exe -Lo kind-windows-amd64.exe https://kind.sigs.k8s.io/dl/v0.30.0/kind-windows-amd64
Move-Item .\kind-windows-amd64.exe C:\kind\kind.exe
```

Create the `C:\kind` folder first if it does not exist, and add it to your system `PATH`:

```powershell
New-Item -ItemType Directory -Force -Path C:\kind
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\kind", "User")
```

---

### Verify kind Installation

Close and reopen PowerShell, then run:

```powershell
kind version
```

---

# Part 3 - Create a Cluster

Create a single-node cluster with the default name `kind`:

```powershell
kind create cluster
```

Create a cluster with a custom name:

```powershell
kind create cluster --name dev-cluster
```

The first run downloads the node image, so it may take a few minutes.

---

# Part 4 - Verify the Cluster

Check cluster info:

```powershell
kubectl cluster-info --context kind-kind
```

Check nodes:

```powershell
kubectl get nodes
```

You should see one node named `kind-control-plane` in `Ready` state.

Check system pods:

```powershell
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

```powershell
kind create cluster --name multi-node --config kind-config.yaml
```

Verify:

```powershell
kubectl get nodes
```

You should see one control-plane node and two worker nodes.

---

# Deploy Your First Application

### Create Deployment

```powershell
kubectl create deployment hello-kind --image=nginx
```

### Expose Deployment

```powershell
kubectl expose deployment hello-kind --type=ClusterIP --port=80
```

### Check Resources

```powershell
kubectl get deployments
kubectl get pods
kubectl get svc
```

### Access the Application

kind does not expose NodePort services on your host by default, so use port-forward:

```powershell
kubectl port-forward service/hello-kind 8080:80
```

Visit http://localhost:8080

### Clean Up the Sample App

```powershell
kubectl delete service hello-kind
kubectl delete deployment hello-kind
```

---

# Load a Local Docker Image into kind

kind nodes cannot see images on your host automatically. Load them like this:

```powershell
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

```powershell
kubectl config current-context
```

It should show:

```
kind-kind
```

For a custom cluster name, the context is `kind-<name>`. Switch between clusters with:

```powershell
kubectl config use-context kind-dev-cluster
```

---

# Troubleshooting

| Problem | Fix |
|---------|-----|
| `Cannot connect to the Docker daemon` | Start Docker Desktop and wait until it shows Running |
| `kind` is not recognized | Reopen PowerShell, or check kind is in your `PATH` |
| Cluster creation hangs or fails | Confirm WSL 2 is enabled in Docker Desktop settings |
| `kubectl` shows wrong cluster | Run `kubectl config use-context kind-<name>` |

---

<p align="center">
  Your local Kubernetes cluster with kind is ready.
</p>
