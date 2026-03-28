# Kubernetes Monitoring with Prometheus & Grafana

---

## Table of Contents

1. [Introduction](#introduction)
2. [Understanding the Basics](#understanding-the-basics)
3. [Architecture Overview](#architecture-overview)
4. [Complete Setup Process](#complete-setup-process)
5. [Using Prometheus](#using-prometheus)
6. [Using Grafana](#using-grafana)
7. [Key Concepts Explained](#key-concepts-explained)
8. [Troubleshooting](#troubleshooting)
9. [Quick Reference](#quick-reference)

---

## Introduction

### What is Monitoring?

Monitoring means continuously checking your system's health and performance instead of waiting for things to break or users to complain.

**Without Monitoring:**
- ❌ Wait for crashes
- ❌ Users complain first
- ❌ Debug blindly
- ❌ React to problems

**With Monitoring:**
- ✅ Track CPU, memory, pods
- ✅ Get instant alerts
- ✅ See problems before users
- ✅ Proactive problem-solving

### Why Do We Need This?

Imagine your backend pod crashes at 2 AM:

**Without monitoring:** Users complain in the morning  
**With monitoring:** Alert sent immediately, you fix it before impact

---

## Understanding the Basics

### What is Each Component?

| Component | Simple Explanation | Real-World Analogy |
|-----------|-------------------|-------------------|
| **Prometheus** | Collects and stores metrics (numbers) | Data collector + database |
| **Grafana** | Shows metrics as graphs | TV screen showing system health |
| **Node Exporter** | Checks server health (CPU, RAM, disk) | Health sensor for machines |
| **kube-state-metrics** | Checks Kubernetes objects status | Health checker for pods/deployments |
| **Alertmanager** | Sends alerts when problems occur | Alarm system |
| **Helm** | Easy installer for Kubernetes apps | App Store for Kubernetes |
| **Kind** | Local Kubernetes cluster in Docker | Practice environment |

### How They Work Together

```
Your Kubernetes Cluster
        ↓
Node Exporter (checks machines)
kube-state-metrics (checks pods)
        ↓
Prometheus (collects all data)
        ↓
Grafana (shows beautiful graphs)
        ↓
Alertmanager (sends alerts)
```

---

## Architecture Overview

### The Complete Picture

```
┌─────────────────────────────────────────────────┐
│           Kubernetes Cluster (Kind)              │
│                                                  │
│  ┌──────────┐        ┌──────────┐               │
│  │  Nodes   │        │   Pods   │               │
│  └────┬─────┘        └────┬─────┘               │
│       │                   │                      │
│       │ metrics           │ metrics              │
│       ▼                   ▼                      │
│  ┌─────────────────────────────────┐            │
│  │      Node Exporter              │            │
│  │   kube-state-metrics            │            │
│  │   cAdvisor (in kubelet)         │            │
│  └──────────────┬──────────────────┘            │
│                 │                                │
│                 │ scrapes every 15s              │
│                 ▼                                │
│  ┌─────────────────────────────────┐            │
│  │        PROMETHEUS                │            │
│  │  (Collects & Stores Metrics)    │            │
│  └──────────┬──────────────────────┘            │
│             │                                    │
│   ┌─────────┴──────────┐                       │
│   │                    │                        │
│   ▼                    ▼                        │
│ ┌──────┐         ┌──────────────┐              │
│ │Grafana│       │ Alertmanager  │              │
│ │(View) │       │  (Alerts)     │              │
│ └──────┘         └──────────────┘              │
└─────────────────────────────────────────────────┘
```

### Monitoring Layers

| Layer | What We Monitor | Tool Used |
|-------|----------------|-----------|
| **Infrastructure** | Node CPU, RAM, Disk | Node Exporter |
| **Kubernetes** | Pods, Deployments, Services | kube-state-metrics |
| **Containers** | Container resources | cAdvisor |
| **Applications** | API requests, errors | Custom metrics |

---

## Complete Setup Process

### Prerequisites

Before starting, ensure you have:
- Docker installed
- Terminal/Command Prompt access
- Internet connection

### Step 1: Create Kubernetes Cluster

**Command:**
```bash
kind create cluster
```

**Why?**  
We need a Kubernetes cluster to monitor. Without a cluster, there's nothing to monitor (like trying to check a car's health without having a car).

**Verify:**
```bash
kubectl get nodes
```

**Expected Output:**
```
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   1m    v1.27.0
```

---

### Step 2: Install Helm

**Command:**
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

**Why?**  
Installing monitoring manually requires 20+ YAML files. Helm is like an app store - it installs everything in one command.

**Verify:**
```bash
helm version
```

**Expected Output:**
```
version.BuildInfo{Version:"v3.x.x", ...}
```

---

### Step 3: Add Prometheus Repository

**Commands:**
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

**Why?**  
Helm needs to know where the monitoring tools are stored. This adds the official Prometheus charts repository.

---

### Step 4: Install Monitoring Stack

**Command:**
```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

**Why?**  
This ONE command installs everything you need:
- Prometheus (data collector)
- Grafana (visualization)
- Alertmanager (alerts)
- Node Exporter (node metrics)
- kube-state-metrics (Kubernetes metrics)
- Prometheus Operator (management)

**What Happens:**
1. Creates `monitoring` namespace
2. Installs all components
3. Configures everything automatically
4. Connects Grafana to Prometheus
5. Imports default dashboards

---

### Step 5: Wait for Pods to Start

**Command:**
```bash
kubectl get pods -n monitoring
```

**Why?**  
We need to confirm all components started successfully. Wait until all show `Running`.

**Expected Output:**
```
NAME                                                     READY   STATUS    RESTARTS   AGE
alertmanager-monitoring-kube-prometheus-alertmanager-0   2/2     Running   0          2m
monitoring-grafana-xxxxxxxxxx-xxxxx                      3/3     Running   0          2m
monitoring-kube-prometheus-operator-xxxxxxxxxx-xxxxx     1/1     Running   0          2m
monitoring-kube-state-metrics-xxxxxxxxxx-xxxxx           1/1     Running   0          2m
monitoring-prometheus-node-exporter-xxxxx                1/1     Running   0          2m
prometheus-monitoring-kube-prometheus-prometheus-0       2/2     Running   0          2m
```

**Note:** Initial states like `PodInitializing` or `ContainerCreating` are normal. Wait 2-3 minutes.

---

### Step 6: Check Services

**Command:**
```bash
kubectl get svc -n monitoring
```

**Why?**  
Services provide network access to pods. All will be `ClusterIP` (internal only).

**Expected Output:**
```
NAME                                      TYPE        CLUSTER-IP      PORT(S)
monitoring-grafana                        ClusterIP   10.96.51.190    80/TCP
monitoring-kube-prometheus-prometheus     ClusterIP   10.96.212.161   9090/TCP
...
```

---

### Step 7: Access Prometheus

**Command:**
```bash
kubectl port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090 -n monitoring
```

**Why?**  
Since Kind runs in Docker, services aren't directly accessible. Port-forwarding creates a tunnel from your laptop to the Kubernetes service.

**Open Browser:**
```
http://localhost:9090
```

**Keep this terminal running!** Close it and access stops.

---

### Step 8: Access Grafana

**Command (in a new terminal):**
```bash
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
```

**Open Browser:**
```
http://localhost:3000
```

---

### Step 9: Get Grafana Password

**Command:**
```bash
kubectl get secret monitoring-grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 -d
```

**Why Decode?**  
Kubernetes stores secrets in Base64 encoding (not encryption, just encoding). We decode to see the actual password.

**Login Credentials:**
- **Username:** `admin`
- **Password:** Output from command above

---

## Using Prometheus

### Access Prometheus UI

Open `http://localhost:9090` in your browser.

### 1. Verify Targets

**Navigate to:** `Status → Targets`

**What to Check:**
All targets should show **UP** status. This confirms Prometheus is successfully scraping metrics.

**Targets You'll See:**
- `monitoring-kube-prometheus-prometheus` - Prometheus itself
- `monitoring-kube-state-metrics` - Kubernetes object metrics
- `monitoring-prometheus-node-exporter` - Node metrics
- `kubernetes-apiservers` - API server metrics
- `kubernetes-nodes` - Node metrics via kubelet

### 2. Run Basic Queries

Prometheus uses **PromQL** (Prometheus Query Language) to query metrics.

#### Check if Targets are Alive

**Query:**
```promql
up
```

**Result:** `1` means target is healthy, `0` means down

#### Check CPU Usage

**Query:**
```promql
rate(node_cpu_seconds_total{mode="idle"}[5m])
```

**What it shows:** CPU idle rate over last 5 minutes

#### Check Memory Available

**Query:**
```promql
node_memory_MemAvailable_bytes
```

**What it shows:** Available memory in bytes

#### Check Pod Restarts

**Query:**
```promql
kube_pod_container_status_restarts_total
```

**What it shows:** Number of times containers have restarted

### 3. Useful Queries for Beginners

```promql
# Total number of pods
count(kube_pod_info)

# Pods not running
kube_pod_status_phase{phase!="Running"}

# Node memory usage percentage
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# CPU usage by pod
sum(rate(container_cpu_usage_seconds_total{pod!=""}[5m])) by (pod)

# Network received bytes
rate(node_network_receive_bytes_total[5m])
```

### Understanding Query Results

- **Instant vector:** Current value
- **Range vector:** Values over time period `[5m]`
- **rate():** Calculate per-second rate
- **sum():** Add values together
- **by (label):** Group results by label

---

## Using Grafana

### Initial Login

1. Open `http://localhost:3000`
2. Username: `admin`
3. Password: From the secret (see Step 9)

### 1. Verify Data Source

**Navigate to:** Settings (⚙️) → Data Sources

**What You'll See:**
- **Prometheus (default)** - Already configured by Helm
- Status: "Data source is working"

**Why It's Auto-Configured:**  
The `kube-prometheus-stack` Helm chart automatically connects Grafana to Prometheus. You don't need to add it manually.

### 2. Explore Pre-Built Dashboards

**Navigate to:** Dashboards → Browse

**Dashboards Installed Automatically:**

| Dashboard Name | What It Shows |
|---------------|---------------|
| **Kubernetes / Compute Resources / Cluster** | Overall cluster CPU, memory, network |
| **Kubernetes / Compute Resources / Namespace** | Resources by namespace |
| **Kubernetes / Compute Resources / Pod** | Individual pod metrics |
| **Kubernetes / Networking / Cluster** | Network traffic and bandwidth |
| **Node Exporter / Nodes** | Detailed node metrics |

**How to Use:**
1. Click on any dashboard
2. Use time range selector (top-right) to change viewing period
3. Use dropdowns at top to filter by namespace, pod, or node
4. Click on any panel to see details

### 3. Understanding Dashboard Panels

**Panel Types:**

- **Time Series:** Line graphs showing changes over time
- **Gauge:** Current value with colored thresholds
- **Stat:** Single number with trend
- **Table:** Data in rows and columns
- **Heatmap:** Color-coded density charts

**Color Meanings:**
- 🟢 **Green:** Healthy/Normal (0-60%)
- 🟡 **Yellow:** Warning (60-80%)
- 🔴 **Red:** Critical (80-100%)

### 4. Create Your Own Dashboard

#### Step-by-Step Guide

**1. Create New Dashboard**
```
Click "+" → Dashboard → Add visualization
```

**2. Select Data Source**
```
Choose: Prometheus
```

**3. Write Query**

Example - CPU Usage:
```promql
rate(node_cpu_seconds_total{mode!="idle"}[5m])
```

**4. Configure Panel**

- **Panel Title:** "Node CPU Usage"
- **Visualization:** Time series
- **Unit:** Percent (0-100)
- **Min/Max:** 0 to 100

**5. Add Thresholds**

```
Green:  0-60
Yellow: 60-80
Red:    80-100
```

**6. Save Dashboard**
```
Click Save (💾) → Enter name → Save
```

### 5. Import Community Dashboards

**Popular Dashboard IDs:**

| ID | Name | Description |
|----|------|-------------|
| 315 | Kubernetes Cluster Monitoring | Complete cluster overview |
| 1860 | Node Exporter Full | Detailed node metrics |
| 6417 | Kubernetes Cluster | Multi-cluster dashboard |
| 747 | Kubernetes Deployment | Deployment-level metrics |

**How to Import:**

1. Go to https://grafana.com/grafana/dashboards
2. Find dashboard and copy ID
3. In Grafana: Click "+" → Import
4. Enter Dashboard ID → Click "Load"
5. Select Prometheus data source
6. Click "Import"

### 6. Customize Dashboard Colors

**Edit Panel Colors:**

1. Click panel title → Edit
2. Go to "Panel options"
3. Under "Standard options":
   - Change **Color scheme**
   - Set **Thresholds** (Green/Yellow/Red)
   - Adjust **Unit** (%, bytes, seconds)

**Example Color Setup for Memory:**
```
Green:  0% - 70% (healthy)
Yellow: 70% - 85% (warning)
Red:    85% - 100% (critical)
```

**Why Color Matters:**  
In production, you need to spot problems instantly. Colors give visual alerts faster than reading numbers.

---

## Key Concepts Explained

### What is Base64 Encoding?

**Question:** Why do we decode secrets?

**Answer:** Kubernetes stores all secrets in Base64 format.

**Base64 is NOT encryption** - it's just encoding text into ASCII-safe characters.

**Example:**
```
Original:  admin123
Base64:    YWRtaW4xMjM=
```

**To Decode:**
```bash
echo "YWRtaW4xMjM=" | base64 -d
# Output: admin123
```

**Why Kubernetes Uses It:**  
YAML files only support safe text. Binary data must be encoded. But remember: anyone can decode Base64 - it's not secure encryption.

---

### NodePort vs ClusterIP vs Port-Forward

**ClusterIP (Default):**
- Only accessible inside cluster
- Internal communication
- Example: `10.96.51.190:80`

**NodePort:**
- Exposed on each node
- Port range: 30000-32767
- Accessible via `<node-ip>:<nodeport>`

**Port-Forward (What We Use):**
- Creates temporary tunnel
- Good for development/testing
- Command: `kubectl port-forward ...`

**Why We Use Port-Forward with Kind:**

Kind runs Kubernetes in Docker containers. NodePort opens ports inside the Docker container, not on your laptop. Port-forward creates a direct tunnel from your laptop to the Kubernetes service.

---

### How Prometheus Discovers Targets

**Service Discovery:**

Prometheus uses Kubernetes API to automatically find targets:

1. **ServiceMonitor:** Custom resource that tells Prometheus what to scrape
2. **Kubernetes API:** Prometheus queries API for pods/services
3. **Labels:** Matches resources based on labels
4. **Endpoints:** Scrapes metrics from discovered endpoints

**Example Flow:**
```
Your Pod with /metrics endpoint
        ↓
ServiceMonitor defines scrape config
        ↓
Prometheus Operator creates scrape job
        ↓
Prometheus scrapes every 15 seconds
```

**You don't manually configure targets** - it's automatic!

---

### Prometheus Operator Explained

**What is it?**  
Prometheus Operator manages Prometheus instances using Kubernetes custom resources.

**Custom Resources (CRDs):**
- **Prometheus:** Defines Prometheus deployment
- **ServiceMonitor:** Defines what to scrape
- **PrometheusRule:** Defines alert rules
- **Alertmanager:** Defines alerting config

**Why Use It?**  
Instead of editing Prometheus config files manually, you create Kubernetes resources. The Operator automatically updates Prometheus.

**Example:**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: metrics
```

This tells Prometheus to scrape any service with label `app: my-app`.

---

### Metrics vs Logs vs Traces

**Metrics (What We Use):**
- Numbers over time
- Example: CPU = 75%
- Good for: Monitoring trends, alerts

**Logs:**
- Text messages
- Example: "Error: Connection refused"
- Good for: Debugging specific issues

**Traces:**
- Request path through services
- Example: API → Backend → Database
- Good for: Understanding latency

**Our Stack = Metrics Only**  
For logs, you'd use ELK/Loki. For traces, you'd use Jaeger/Tempo.

---

## Troubleshooting

### Pods Not Starting

**Check Status:**
```bash
kubectl get pods -n monitoring
kubectl describe pod <pod-name> -n monitoring
```

**Common Issues:**

1. **Pending:** Not enough resources
   ```bash
   kubectl top nodes  # Check if nodes have capacity
   ```

2. **ImagePullBackOff:** Can't download image
   ```bash
   kubectl describe pod <pod-name> -n monitoring
   # Check Events section
   ```

3. **CrashLoopBackOff:** Pod keeps crashing
   ```bash
   kubectl logs <pod-name> -n monitoring
   ```

### Can't Access Prometheus/Grafana

**Check Port-Forward:**
```bash
# List running port-forwards
jobs

# If stopped, restart
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring &
```

**Check Services:**
```bash
kubectl get svc -n monitoring
# Ensure services exist and have correct ports
```

### Grafana Shows "No Data"

**Verify Prometheus Connection:**
1. Settings → Data Sources
2. Click "Prometheus"
3. Scroll down → Click "Test"
4. Should show "Data source is working"

**Check Prometheus Targets:**
```bash
# Access Prometheus UI
http://localhost:9090/targets

# All should be "UP"
```

**Verify Metrics Exist:**
```bash
# In Prometheus UI, run:
up

# Should return 1 for each target
```

### Wrong Grafana Password

**Reset Password:**
```bash
# Delete existing secret
kubectl delete secret monitoring-grafana -n monitoring

# Uninstall and reinstall
helm uninstall monitoring -n monitoring
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring
```

---


### Common Commands

```bash
# Check all monitoring pods
kubectl get pods -n monitoring

# Check services
kubectl get svc -n monitoring

# Access Prometheus
kubectl port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090 -n monitoring

# Access Grafana
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring

# Get Grafana password
kubectl get secret monitoring-grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 -d

# View pod logs
kubectl logs <pod-name> -n monitoring

# Delete monitoring stack
helm uninstall monitoring -n monitoring

# Delete cluster
kind delete cluster
```



```promql
# Basic health check
up

# CPU usage percentage
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage percentage
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Pod restart count
sum(kube_pod_container_status_restarts_total) by (pod, namespace)

# Disk usage
(node_filesystem_size_bytes - node_filesystem_avail_bytes) / node_filesystem_size_bytes * 100

# Network received (MB/s)
rate(node_network_receive_bytes_total[5m]) / 1024 / 1024

# Count running pods
count(kube_pod_status_phase{phase="Running"})
```

### Key URLs

- **Prometheus:** http://localhost:9090
- **Grafana:** http://localhost:3000
- **Grafana Dashboards:** https://grafana.com/grafana/dashboards
- **Prometheus Docs:** https://prometheus.io/docs/
- **Helm Charts:** https://github.com/prometheus-community/helm-charts

---

