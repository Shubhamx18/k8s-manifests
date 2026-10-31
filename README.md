<!-- <div align="center">

<img src="https://readme-typing-svg.demolab.com?font=JetBrains+Mono&weight=700&size=28&duration=3000&pause=1000&color=326CE5&center=true&vCenter=true&width=600&lines=Kubernetes+from+Zero;Break+It.+Fix+It.+Own+It.;Real+Clusters.+Real+Problems." alt="Typing SVG" />

<br/> -->

```
██╗  ██╗ █████╗ ███████╗    ███╗   ███╗ █████╗ ███╗  ██╗██╗███████╗███████╗███████╗████████╗███████╗
██║ ██╔╝██╔══██╗██╔════╝    ████╗ ████║██╔══██╗████╗ ██║██║██╔════╝██╔════╝██╔════╝╚══██╔══╝██╔════╝
█████╔╝ ╚█████╔╝███████╗    ██╔████╔██║███████║██╔██╗██║██║█████╗  █████╗  ███████╗   ██║   ███████╗
██╔═██╗ ██╔══██╗╚════██║    ██║╚██╔╝██║██╔══██║██║╚████║██║██╔══╝  ██╔══╝  ╚════██║   ██║   ╚════██║
██║  ██╗╚█████╔╝███████║    ██║ ╚═╝ ██║██║  ██║██║ ╚███║██║██║     ███████╗███████║   ██║   ███████║
╚═╝  ╚═╝ ╚════╝ ╚══════╝    ╚═╝     ╚═╝╚═╝  ╚═╝╚═╝  ╚══╝╚═╝╚═╝     ╚══════╝╚══════╝   ╚═╝   ╚══════╝
```

<br/>

<p>
  <a href="https://kubernetes.io"><img src="https://img.shields.io/badge/Kubernetes-1.29+-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white"/></a>
  <a href="https://kind.sigs.k8s.io"><img src="https://img.shields.io/badge/Local_Cluster-KIND-0db7ed?style=for-the-badge&logo=docker&logoColor=white"/></a>
  <a href="#"><img src="https://img.shields.io/badge/Level-Beginner_to_Advanced-22c55e?style=for-the-badge"/></a>
  <a href="#"><img src="https://img.shields.io/badge/PRs-Welcome-f59e0b?style=for-the-badge"/></a>
</p>

<br/>

> **Not a tutorial. Not a cheatsheet.**
> A hands-on repo built by actually running, breaking, and fixing every concept on a real cluster.

</div>

---

<div align="center">

##  What You Will Be Able to Do

</div>

```
Deploy production-shaped apps        →   StatefulSets, Deployments, Services wired together
Manage config and secrets cleanly    →   ConfigMaps, Secrets, never hardcoded credentials
Handle persistent storage            →   PV, PVC, StorageClass, volume lifecycle
Route external traffic properly      →   Ingress, path routing, host routing, TLS
Run scheduled and one-off tasks      →   Jobs, CronJobs, backups, migrations
Debug anything that breaks           →   CrashLoopBackOff, PVC Pending, 502s — all of it
```

---

## Repo Map

```
k8s-manifests/
│
├── 📄 01_KIND_Kubernetes_Cluster.md          ← Local cluster setup
├── 📄 02_Kubernetes_Namespaces.md            ← Isolation, quotas, organization
├── 📄 03_Pods_and_Services.md                ← Core units + all 4 service types
├── 📄 04_ReplicaSet_and_Deployment.md        ← Self-healing, rolling updates, rollback
├── 📄 05_Volumes_PV_PVC.md                   ← Persistent storage, 3-layer model
├── 📄 06_Kubernetes_Ingress.md               ← HTTP routing, TLS, NGINX controller
├── 📄 07_Kubernetes_Jobs.md                  ← One-time tasks, batch, migrations
├── 📄 08_Kubernetes_CronJobs.md              ← Scheduled work, backups, cleanup
├── 📄 09_ConfigMap_Secret_MySQL_NodeJS.md    ← Full app — everything wired together
└── 📄 10_Kubernetes_Debugging.md             ← Every failure case. Diagnosis. Fix.
│
└── manifests/
    ├── namespace.yaml
    ├── mysql-config.yaml
    ├── mysql-secret.yaml
    ├── mysql-service.yaml
    ├── mysql-statefulset.yaml
    ├── node-app-deployment.yaml
    └── node-app-service.yaml
```

---

##  Learning Path

> Follow this order. Each topic builds directly on the previous one.

<br/>

| # | Topic | One Line |
|:---:|---|---|
| `01` | [**KIND — Local Cluster**](./01_KIND_Kubernetes_Cluster.md) | Real Kubernetes on your machine using Docker. No VMs. No cloud bill. |
| `02` | [**Namespaces**](./02_Kubernetes_Namespaces.md) | Logical isolation. Resource quotas. Cross-namespace DNS. |
| `03` | [**Pods & Services**](./03_Pods_and_Services.md) | ClusterIP · NodePort · LoadBalancer · Headless — when and why for each. |
| `04` | [**ReplicaSet & Deployment**](./04_ReplicaSet_and_Deployment.md) | Zero-downtime rolling updates. Rollback in one command. |
| `05` | [**Volumes · PV · PVC**](./05_Volumes_PV_PVC.md) | Data that survives pod death. The 3-layer storage model. |
| `06` | [**Ingress**](./06_Kubernetes_Ingress.md) | One entry point for all HTTP traffic. Path routing. Host routing. TLS. |
| `07` | [**Jobs**](./07_Kubernetes_Jobs.md) | Run once, finish, stop. Migrations, scripts, batch work. |
| `08` | [**CronJobs**](./08_Kubernetes_CronJobs.md) | Scheduled tasks. DB backups. Log cleanup. Automatic. |
| `09` | [**Full Stack — MySQL + Node.js**](./09_ConfigMap_Secret_MySQL_NodeJS.md) | Everything above combined into one real working app. |
| `10` | [**Debugging**](./10_Kubernetes_Debugging.md) | **The most important file.** Every failure. Every fix. |

---

##  The Capstone — MySQL + Node.js on Kubernetes

> This is what the entire repo builds toward. A production-shaped app using every concept covered.

<br/>

```
┌──────────────────────────────────────┐
│          Node.js Deployment          │  ← Stateless, scales horizontally
│   Config  ←  ConfigMap               │
│   Password ← Secret                  │
└─────────────────┬────────────────────┘
                  │ connects via service name "mysql"
                  ▼
┌──────────────────────────────────────┐
│       MySQL Headless Service         │  ← clusterIP: None
│  mysql-0.mysql.myspace.svc.local     │  ← stable DNS per pod
└─────────────────┬────────────────────┘
                  │
                  ▼
┌──────────────────────────────────────┐
│         MySQL StatefulSet            │  ← stable pod name: mysql-0
│              mysql-0                 │  ← survives restarts
└─────────────────┬────────────────────┘
                  │ mounted at /var/lib/mysql
                  ▼
┌──────────────────────────────────────┐
│       PersistentVolume (1Gi)         │  ← data lives here, pod-independent
└──────────────────────────────────────┘
```

<br/>

**Deploy the full stack:**

```bash
kubectl apply -f manifests/namespace.yaml
kubectl apply -f manifests/mysql-config.yaml
kubectl apply -f manifests/mysql-secret.yaml
kubectl apply -f manifests/mysql-service.yaml
kubectl apply -f manifests/mysql-statefulset.yaml
kubectl apply -f manifests/node-app-deployment.yaml
kubectl apply -f manifests/node-app-service.yaml

# verify everything is up
kubectl get all -n myspace
```

---

##  Debugging — Quick Reference

> Full diagnosis guide with commands, causes, and fixes → [`10_Kubernetes_Debugging.md`](./10_Kubernetes_Debugging.md)

<br/>

| What You See | What It Means | First Command to Run |
|---|---|---|
| `CrashLoopBackOff` | Container starts, crashes, repeats | `kubectl logs <pod> --previous -n myspace` |
| `ImagePullBackOff` | Can't pull the container image | `kubectl describe pod <pod> -n myspace` |
| `Pending` forever | Pod can't be scheduled to any node | `kubectl describe pod <pod> -n myspace` → Events |
| `OOMKilled` | Container exceeded its memory limit | Increase `limits.memory` in container spec |
| Service has no traffic | Selector doesn't match any pod labels | `kubectl get endpoints <svc> -n myspace` |
| Ingress returns `404` | Missing `rewrite-target` annotation | `kubectl describe ingress -n myspace` |
| PVC stuck `Pending` | No PV matches access mode or size | `kubectl describe pvc <name> -n myspace` |
| CronJob never fires | Bad schedule expression or `suspend: true` | `kubectl describe cronjob <name> -n myspace` |

<br/>

**The 5 commands that solve 90% of problems:**

```bash
kubectl get pods -n <namespace>                                  # where things stand
kubectl describe pod <pod-name> -n <namespace>                   # what kubernetes did and why
kubectl logs <pod-name> -n <namespace> --previous                # what the app said before it died
kubectl get endpoints <service-name> -n <namespace>              # is the service actually routing?
kubectl get events -n <namespace> --sort-by='.lastTimestamp'     # full timeline of what happened
```

---

##  Concepts at a Glance

<br/>

**Workloads — what runs your app**

| Resource | When to Use |
|---|---|
| `Deployment` | Stateless apps — APIs, web servers, microservices |
| `StatefulSet` | Stateful apps that need stable identity — databases, Kafka |
| `DaemonSet` | One pod on every node — log collectors, monitoring agents |
| `Job` | Run-to-completion tasks — migrations, one-off scripts |
| `CronJob` | Repeating scheduled tasks — backups, report generation |

<br/>

**Networking — how traffic flows**

| Resource | When to Use |
|---|---|
| `ClusterIP` | Service-to-service communication inside the cluster |
| `NodePort` | External access in local or bare-metal clusters |
| `LoadBalancer` | External access in cloud clusters (EKS, GKE, AKS) |
| `Headless Service` | Direct pod-level DNS — StatefulSets, databases |
| `Ingress` | HTTP/HTTPS routing with path rules, host rules, TLS |

<br/>

**Storage — data that outlives pods**

| Resource | What It Is |
|---|---|
| `Volume` | Attached to a pod — dies when the pod dies |
| `PersistentVolume` | Cluster-level storage, independent of any pod |
| `PersistentVolumeClaim` | A pod's request to use a PV |
| `StorageClass` | Dynamic provisioning — cloud disks on demand |

<br/>

**Config & Security**

| Resource | What It Is |
|---|---|
| `ConfigMap` | Non-sensitive config — DB host, port, feature flags |
| `Secret` | Sensitive data — passwords, API keys, tokens |
| `ResourceQuota` | Hard caps on what a namespace can consume |
| `NetworkPolicy` | Firewall rules between pods and namespaces |

---

##  Get Started in 3 Minutes

**Prerequisites:** [Docker](https://docs.docker.com/get-docker/) · [kubectl](https://kubernetes.io/docs/tasks/tools/) · [KIND](https://kind.sigs.k8s.io)

```bash
# 1. Install KIND
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

# 2. Create your cluster
kind create cluster --name k8s-lab

# 3. Verify
kubectl get nodes

# 4. Clone and start from file 01
git clone https://github.com/Shubhamx18/k8s-manifests.git
cd k8s-manifests
```

---

## 💬 Real Talk

Reading these files will make you comfortable with Kubernetes. That's real value.

But mastery is different. Mastery is when something breaks in a way you've never seen, on a cluster you can't restart, and you still know exactly where to look and how to reason through it. That only comes from building things on your own — deciding the architecture yourself, making the wrong call, migrating it. Hitting an error at the worst time with no matching Stack Overflow answer and figuring it out anyway.

**Use this repo to get solid on the fundamentals. Then close it and build something real.**

---

##  Contributing

Found a mistake, a missing failure case, or a better explanation? PRs are open.

The bar is simple — everything must be practical. No filler theory. Every concept needs a working YAML example. Every failure case needs a clear diagnosis path.

---

<div align="center">

<br/>

**Built by [Shubham](https://github.com/Shubhamx18)**

*Every file in this repo came from actually running, breaking, and understanding each concept on a real cluster.*

<br/>

[![GitHub followers](https://img.shields.io/github/followers/Shubhamx18?style=for-the-badge&logo=github&labelColor=161b22&color=326CE5)](https://github.com/Shubhamx18)
[![GitHub stars](https://img.shields.io/github/stars/Shubhamx18/k8s-manifests?style=for-the-badge&logo=github&labelColor=161b22&color=f59e0b)](https://github.com/Shubhamx18/k8s-manifests/stargazers)

<br/>

*If this repo helped you — star it, share it, or open a PR.* 

</div>
