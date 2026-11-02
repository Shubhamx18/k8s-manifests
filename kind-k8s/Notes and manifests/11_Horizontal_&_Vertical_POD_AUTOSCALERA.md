# Kubernetes VPA: Vertical Pod Autoscaler

> Guessing CPU and memory requests is hard. Too low and Pods get throttled or killed.
> Too high and you pay for resources nobody uses.
> VPA watches real usage and adjusts the **size of each Pod**.

---

## Why VPA Matters

```
Requests too low   ->  CPU throttling / OOMKilled
Requests too high  ->  wasted node capacity, higher cost
```

- Good for apps that **cannot scale out** (single-instance apps, many databases)
- Good for finding the right requests for any app

---

## How It Works

```
Pod usage  ->  Recommender  ->  Updater  ->  Admission Controller
(metrics)      (calculates)     (evicts Pod)  (sets new requests on the new Pod)
```

| Component | Role |
|---|---|
| **Recommender** | Watches usage history and calculates the recommended requests. |
| **Updater** | Evicts Pods whose requests differ too much from the recommendation. |
| **Admission Controller** | Rewrites the requests of Pods as they are created. |

> VPA is **not built into Kubernetes**. You install it separately (CRDs + the three components above). It also needs metrics-server.

---

## Update Modes

| `updateMode` | What VPA does | Use for |
|---|---|---|
| `Off` | Only calculates recommendations. Changes nothing. | **Start here.** Safe, read-only. |
| `Initial` | Sets requests only when a Pod is created. Never restarts running Pods. | Apps that must not be disrupted |
| `Recreate` | Evicts Pods to apply new requests. | Apps that tolerate restarts |
| `Auto` | Currently behaves like `Recreate`. | Fully automatic tuning |

> Newer VPA versions add an in-place resize option so Pods may not need a restart. Check the VPA release notes for your version.

> **Every mode except `Off` and `Initial` restarts Pods** to change their resources. A single-replica app will see a brief outage.

---

## Install VPA

```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh

# Verify
kubectl get pods -n kube-system | grep vpa
kubectl get crd | grep verticalpodautoscaler
```

You should see three Pods: `vpa-recommender`, `vpa-updater`, `vpa-admission-controller`.

---

## YAML Examples: Luminary Project

### 1. Deployment (target workload)

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

### 2. VPA in Recommendation-Only Mode

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

### 3. VPA with Limits and Automatic Updates

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

## Commands

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

## Common Mistakes

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

## VPA at a Glance

| Question | Answer |
|---|---|
| What does it change? | CPU and memory requests (and limits, proportionally) of each Pod |
| Best for | Right-sizing, single-instance apps, databases |
| Needs | VPA components installed, metrics-server |
| Built into Kubernetes? | No, a separate add-on |
| Restarts Pods to apply? | Yes, in `Recreate` / `Auto` modes |

---

## HPA vs VPA

| | HPA | VPA |
|---|---|---|
| Scales | Number of Pods (out) | Size of each Pod (up) |
| Best for | Stateless apps with variable traffic | Apps that can't scale out, right-sizing |
| Built in | Yes | No, install separately |
| Disruption | None | Pod restarts |
| Safe together? | Only if HPA uses a different metric than VPA (e.g. HPA on custom metrics, VPA on CPU/memory) |

> **Practical pattern:** run VPA in `Off` mode to learn the right requests, set those requests by hand, then let HPA handle traffic changes.
