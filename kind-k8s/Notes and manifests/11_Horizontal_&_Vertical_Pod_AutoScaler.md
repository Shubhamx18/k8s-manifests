# Kubernetes Autoscaling: HPA & VPA

> Fixed Pod counts and guessed resource sizes waste money or fail under load.
> **HPA** changes how many Pods run. **VPA** changes how big each Pod is.

---

## Overview

```
Traffic changes  ->  HPA  ->  more / fewer Pods        (scale out)
Usage changes    ->  VPA  ->  bigger / smaller Pods    (scale up)
Both need        ->  metrics-server  ->  live CPU and memory data
```

| | HPA | VPA |
|---|---|---|
| Scales | Number of Pods | Size of each Pod (CPU / memory requests) |
| Best for | Stateless apps with variable traffic | Single-instance apps, right-sizing |
| Built into Kubernetes | Yes | No, install separately |
| Needs metrics-server | Yes | Yes |
| Disruption | None | Pod restarts (except `Off` / `Initial` modes) |

**Setup order:**

1. Install **metrics-server** (once)
2. Use **HPA** for traffic-based scaling
3. Install **VPA** if you want resource recommendations or tuning

---

## Install metrics-server

**metrics-server** collects CPU and memory usage from the nodes. It is a separate add-on, installed **once per cluster** and shared by HPA, VPA and `kubectl top`. Install it before anything else in this file.

**1. Check if it is already installed:**

```bash
kubectl get deployment metrics-server -n kube-system
kubectl top nodes
```

If `kubectl top nodes` prints CPU and memory numbers, skip to the next section. If you get "Metrics API not available" or the Deployment is not found, install it.

**2. Install:**

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

**3. kind clusters only: patch it.** kind's kubelet certificates are self-signed, so without this flag the Pod never becomes Ready:

```bash
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

**4. Verify** (wait about a minute):

```bash
kubectl get pods -n kube-system | grep metrics-server     # 1/1 Running
kubectl top nodes
kubectl top pods -A
```

| Problem | Fix |
|---|---|
| Pod `0/1 Running` on kind | Apply the `--kubelet-insecure-tls` patch above |
| `Metrics API not available` | metrics-server is not installed or not Ready yet |
| `kubectl top` works but HPA shows `<unknown>` | Containers have no CPU `requests` |
| `kubectl top` works but VPA shows no recommendation | Wait a few minutes, VPA needs usage history |

---

## Part 1: HPA (Horizontal Pod Autoscaler)

> HPA changes the **number of Pods** automatically based on live metrics.

### Why HPA Matters

```
Traffic spike  ->  Pods overloaded  ->  slow responses / crashes
Traffic drops  ->  idle Pods        ->  wasted resources
```

- Good for **stateless apps** (web servers, APIs)
- Not suited to apps that cannot run more than one copy (single-writer databases)

---

### How It Works

```
metrics-server  ->  HPA controller  ->  Deployment  ->  more / fewer Pods
   (reads usage)    (checks every 15s)   (replicas changed)
```

| Component | Role |
|---|---|
| **metrics-server** | Collects CPU and memory usage from the nodes. HPA does not work without it. |
| **HPA controller** | Compares current usage with your target and calculates the replica count. |
| **Deployment / StatefulSet** | The workload whose `replicas` value HPA changes. |

**The formula:**

```
desiredReplicas = ceil( currentReplicas x currentUsage / targetUsage )
```

Example: 2 Pods at 90% CPU, target 50% -> ceil(2 x 90 / 50) = **4 Pods**.

---

### Key Behaviors

| Behavior | Default |
|---|---|
| Check interval | Every 15 seconds |
| Tolerance | No change if within about 10% of the target |
| Scale-down delay | 300 seconds (stabilization window), to avoid flapping |
| Scale-up | Immediate once the target is exceeded |

> CPU utilization is measured as a **percentage of the Pod's CPU request**.
> If the container has no CPU request, HPA cannot calculate anything.

---

### Prerequisites

1. **metrics-server** installed and working (see **Install metrics-server** above)
2. **Resource requests** set on the containers
3. A scalable workload (Deployment, StatefulSet, ReplicaSet)

---

### YAML Examples: Luminary Project

#### 1. Deployment (with resource requests)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: luminary

spec:
  # do not set replicas here once HPA manages it
  selector:
    matchLabels:
      app: web

  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m            # required, HPA % is based on this
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
```

---

#### 2. HPA (autoscaling/v2)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
  namespace: luminary

spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web                      # must match the Deployment name

  minReplicas: 2                   # never fewer than this
  maxReplicas: 10                  # never more than this

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50   # keep average CPU near 50% of requests

  behavior:                        # optional: control the speed
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1                 # remove at most 1 Pod ...
          periodSeconds: 60        # ... per minute
    scaleUp:
      policies:
        - type: Percent
          value: 100               # at most double ...
          periodSeconds: 60        # ... per minute
```

> **Checklist:**
> - `scaleTargetRef.name` matches the Deployment
> - Container has `resources.requests.cpu`
> - `minReplicas` is at least 2 if you need high availability

---

#### 3. Quick Imperative Version

```bash
kubectl autoscale deployment web -n luminary --cpu-percent=50 --min=2 --max=10
```

> Fine for testing. Use YAML for anything you want in Git.

---

### Commands

```bash
# Apply
kubectl apply -f deployment.yaml
kubectl apply -f hpa.yaml

# Watch scaling live
kubectl get hpa -n luminary -w

# Debug: shows conditions, events, current vs target
kubectl describe hpa web-hpa -n luminary

# See current usage
kubectl top pods -n luminary
```

**Reading `kubectl get hpa`:**

```
NAME      REFERENCE        TARGETS   MINPODS   MAXPODS   REPLICAS
web-hpa   Deployment/web   23%/50%   2         10        2
```

| TARGETS value | Meaning |
|---|---|
| `23%/50%` | Current usage / target. Working. |
| `<unknown>/50%` | No metrics. Check metrics-server and CPU requests. |

---

### Test It: Generate Load

```bash
# Terminal 1: watch
kubectl get hpa web-hpa -n luminary -w

# Terminal 2: expose the Deployment and hit it in a loop
kubectl expose deployment web -n luminary --port=80
kubectl run load -n luminary --rm -it --image=busybox:1.36 -- \
  /bin/sh -c "while true; do wget -q -O- http://web; done"
```

Stop the load with Ctrl+C. The replica count falls back after about 5 minutes (the scale-down window).

---

### Common Mistakes

| Mistake | Why It's Wrong |
|---|---|
| No CPU `requests` on the container | HPA shows `<unknown>` and never scales |
| metrics-server not installed | No metrics, so no scaling |
| Hardcoding `replicas` in the Deployment and re-applying it | Resets the count HPA chose |
| `minReplicas` equal to `maxReplicas` | HPA can never scale |
| CPU target too low (e.g. 10%) | Scales out constantly |
| Using HPA and VPA on the same CPU/memory metric | They fight each other |
| Expecting instant scale-down | Default 5 minute stabilization window |
| Cluster has no spare node capacity | New Pods stay `Pending`. HPA adds Pods, it does not add nodes (that is the Cluster Autoscaler). |

---

### HPA at a Glance

| Question | Answer |
|---|---|
| What does it change? | Number of Pods |
| Best for | Stateless apps with variable traffic |
| Needs | metrics-server, resource requests |
| Built into Kubernetes? | Yes |
| Restarts Pods to scale? | No, it only adds or removes Pods |

---

## Part 2: VPA (Vertical Pod Autoscaler)

> VPA watches real usage and adjusts the **size of each Pod**.

### Why VPA Matters

```
Requests too low   ->  CPU throttling / OOMKilled
Requests too high  ->  wasted node capacity, higher cost
```

- Good for apps that **cannot scale out** (single-instance apps, many databases)
- Good for finding the right requests for any app

---

### How It Works

```
Pod usage  ->  Recommender  ->  Updater  ->  Admission Controller
(metrics)      (calculates)     (evicts Pod)  (sets new requests on the new Pod)
```

| Component | Role |
|---|---|
| **Recommender** | Watches usage history and calculates the recommended requests. |
| **Updater** | Evicts Pods whose requests differ too much from the recommendation. |
| **Admission Controller** | Rewrites the requests of Pods as they are created. |

> VPA is **not built into Kubernetes**. You install it separately (CRDs + the three components above). It also needs metrics-server (see Install sections below).

---

### Update Modes

| `updateMode` | What VPA does | Use for |
|---|---|---|
| `Off` | Only calculates recommendations. Changes nothing. | **Start here.** Safe, read-only. |
| `Initial` | Sets requests only when a Pod is created. Never restarts running Pods. | Apps that must not be disrupted |
| `Recreate` | Evicts Pods to apply new requests. | Apps that tolerate restarts |
| `Auto` | Currently behaves like `Recreate`. | Fully automatic tuning |

> Newer VPA versions add an in-place resize option so Pods may not need a restart. Check the VPA release notes for your version.

> **Every mode except `Off` and `Initial` restarts Pods** to change their resources. A single-replica app will see a brief outage.

---

### Install VPA

VPA is **not** metrics-server. It is a separate add-on made of three components, and it needs metrics-server to be running first.

| | What it does | Installed by |
|---|---|---|
| **metrics-server** | Collects CPU and memory usage from the nodes | Its own manifest (see **Install metrics-server** above) |
| **VPA** | Reads those metrics, then recommends or changes Pod requests | `./hack/vpa-up.sh` (below) |

```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh

# Verify
kubectl get pods -n kube-system | grep vpa
kubectl get crd | grep verticalpodautoscaler
```

You should see three Pods: `vpa-recommender`, `vpa-updater`, `vpa-admission-controller`.

**Order:** metrics-server first, then VPA, then your VPA YAML.

---

### YAML Examples: Luminary Project

#### 1. Deployment (target workload)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: luminary

spec:
  replicas: 2
  selector:
    matchLabels:
      app: web

  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx
          resources:
            requests:
              cpu: 100m          # starting guess, VPA will refine it
              memory: 128Mi
```

---

#### 2. VPA in Recommendation-Only Mode

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-vpa
  namespace: luminary

spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web                    # must match the Deployment name

  updatePolicy:
    updateMode: "Off"            # recommend only, no restarts
```

---

#### 3. VPA with Limits and Automatic Updates

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-vpa
  namespace: luminary

spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web

  updatePolicy:
    updateMode: "Auto"

  resourcePolicy:
    containerPolicies:
      - containerName: "*"       # all containers, or use a specific name
        minAllowed:
          cpu: 50m
          memory: 64Mi
        maxAllowed:
          cpu: "1"
          memory: 1Gi
        controlledResources: ["cpu", "memory"]
```

> **Checklist:**
> - `targetRef.name` matches the workload
> - Set `minAllowed` / `maxAllowed` so VPA can't pick extreme values
> - Run with `Off` first and review before switching to `Auto`

---

### Commands

```bash
# Apply
kubectl apply -f vpa.yaml

# See recommendations
kubectl get vpa -n luminary
kubectl describe vpa web-vpa -n luminary

# Compare with actual usage
kubectl top pods -n luminary
```

**Reading the recommendation** (bottom of `describe`):

```
Recommendation:
  Container Recommendations:
    Container Name:  web
    Lower Bound:     Cpu: 25m,  Memory: 64Mi
    Target:          Cpu: 50m,  Memory: 100Mi
    Upper Bound:     Cpu: 200m, Memory: 300Mi
    Uncapped Target: Cpu: 50m,  Memory: 100Mi
```

| Field | Meaning |
|---|---|
| **Target** | The value VPA wants to set. Use this. |
| **Lower Bound** | Minimum reasonable request. Below this the Pod likely struggles. |
| **Upper Bound** | Above this the request is probably wasted. |
| **Uncapped Target** | Target without your `minAllowed` / `maxAllowed` limits applied. |

> Give VPA some time (minutes to hours) after install. Early recommendations are based on little data.

---

### Common Mistakes

| Mistake | Why It's Wrong |
|---|---|
| Using `Auto` with a single replica | The Pod is evicted and the app goes down briefly |
| VPA and HPA both on CPU or memory | They fight: HPA adds Pods while VPA resizes them |
| Starting in `Auto` mode | Unreviewed changes in production. Start with `Off`. |
| No `minAllowed` / `maxAllowed` | VPA could pick a value too small to run or too large to schedule |
| Trusting recommendations right after install | Not enough usage history yet |
| Forgetting metrics-server | VPA has no data to work with |
| Expecting VPA to add nodes | It only changes Pod requests. A bigger Pod may not fit and stays `Pending`. |

---

### VPA at a Glance

| Question | Answer |
|---|---|
| What does it change? | CPU and memory requests (and limits, proportionally) of each Pod |
| Best for | Right-sizing, single-instance apps, databases |
| Needs | VPA components installed, metrics-server |
| Built into Kubernetes? | No, a separate add-on |
| Restarts Pods to apply? | Yes, in `Recreate` / `Auto` modes |

---

## Using HPA and VPA Together

- **Safe together** only if they use different metrics. Do not let both act on CPU or memory for the same workload, because HPA adds Pods while VPA resizes them and they fight.
- **Practical pattern:** run VPA in `Off` mode to learn the right requests, set those requests by hand, then let HPA handle traffic changes.
