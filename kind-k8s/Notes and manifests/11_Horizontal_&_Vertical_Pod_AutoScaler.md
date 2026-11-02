# Kubernetes Autoscaling: HPA & VPA

> **HPA** adds or removes Pods when traffic changes.
> **VPA** makes each Pod bigger or smaller based on real usage.
> Both need **metrics-server** to see CPU and memory usage.

---

## The Big Picture

| | HPA | VPA |
|---|---|---|
| What it changes | **Number** of Pods | **Size** of each Pod (CPU / memory requests) |
| Think of it as | Hiring more workers | Giving each worker more strength |
| Best for | Web apps and APIs with changing traffic | Apps that run as a single copy, and finding the right resource sizes |
| Built into Kubernetes | Yes | No, install it yourself |
| Restarts Pods | No | Yes (except in `Off` and `Initial` modes) |

**Follow this order:**

```
Step 1: metrics-server  ->  Step 2: HPA  ->  Step 3: VPA (optional)
```

---

## Step 1: Install metrics-server (once per cluster)

metrics-server reads CPU and memory usage from the nodes. HPA, VPA and `kubectl top` all depend on it.

**Check if it is already installed:**

```bash
kubectl top nodes
```

If this prints CPU and memory numbers, skip to Step 2. If you get an error like "Metrics API not available", install it:

```bash
# Install
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

**Only on kind:** add one flag, because kind's certificates are self-signed. Without it the Pod never becomes Ready.

```bash
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

**Verify** (wait about a minute):

```bash
kubectl get pods -n kube-system | grep metrics-server    # should be 1/1 Running
kubectl top nodes
kubectl top pods -A
```

---

## Step 2: HPA (Horizontal Pod Autoscaler)

### How it works

HPA checks usage every 15 seconds and calculates how many Pods you need:

```
desired Pods = ceil( current Pods x current usage / target usage )
```

Example: 2 Pods running at 90% CPU, target 50% -> ceil(2 x 90 / 50) = **4 Pods**.

Scaling up is fast. Scaling down waits about 5 minutes, so the count doesn't bounce up and down.

> **Important:** CPU usage is measured as a percentage of the container's CPU **request**. If the container has no `requests.cpu`, HPA cannot work.

### 2a. The Deployment (with CPU request)

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: luminary
spec:
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
              cpu: 100m          # required for HPA
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
```

> Do not set `replicas:` in the Deployment once HPA manages it. Re-applying would reset the count.

### 2b. The HPA

```yaml
# hpa.yaml
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
  minReplicas: 2                   # never fewer than 2
  maxReplicas: 10                  # never more than 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50   # keep average CPU near 50% of the request
```

Quick alternative (good for testing):

```bash
kubectl autoscale deployment web -n luminary --cpu-percent=50 --min=2 --max=10
```

### 2c. Apply and watch

```bash
kubectl create namespace luminary
kubectl apply -f deployment.yaml
kubectl apply -f hpa.yaml

kubectl get hpa -n luminary -w
```

Read the output like this:

```
NAME      REFERENCE        TARGETS   MINPODS   MAXPODS   REPLICAS
web-hpa   Deployment/web   23%/50%   2         10        2
```

- `23%/50%` means current usage / target. Working.
- `<unknown>/50%` means no metrics. Check Step 1 and that the Deployment has `requests.cpu`.

### 2d. Test it with load

```bash
# Terminal 1: keep watching
kubectl get hpa web-hpa -n luminary -w

# Terminal 2: give the Deployment a Service, then send traffic in a loop
kubectl expose deployment web -n luminary --port=80
kubectl run load -n luminary --rm -it --image=busybox:1.36 -- \
  /bin/sh -c "while true; do wget -q -O- http://web; done"
```

Replicas should climb above 2. Stop the load with Ctrl+C, and after about 5 minutes the count falls back.

### 2e. HPA problems

| Problem | Cause and fix |
|---|---|
| `<unknown>` in TARGETS | metrics-server not working, or no `requests.cpu` on the container |
| Never scales | `minReplicas` equals `maxReplicas`, or usage never reaches the target |
| New Pods stay `Pending` | The nodes have no free capacity. HPA adds Pods, not nodes |
| Replica count resets after `kubectl apply` | `replicas:` is hard-coded in the Deployment. Remove it |

---

## Step 3: VPA (Vertical Pod Autoscaler)

### How it works

VPA watches how much CPU and memory your Pods really use, then recommends better `requests`. Depending on its mode, it can also apply them by restarting the Pods.

It is **not built into Kubernetes**. It adds three components: a **recommender** (calculates), an **updater** (restarts Pods) and an **admission controller** (sets the new values on new Pods).

### 3a. Update modes

| `updateMode` | What VPA does |
|---|---|
| `Off` | Only shows recommendations. Changes nothing. **Start here.** |
| `Initial` | Applies values only when a Pod is newly created. Never restarts running Pods. |
| `Recreate` | Restarts Pods to apply new values. |
| `Auto` | Currently works like `Recreate`. |

> Newer VPA versions can also resize Pods in place without a restart. Check the release notes for the version you install.

### 3b. Install VPA

metrics-server must already be working (Step 1). VPA is a separate install:

```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh

# Verify: you should see 3 vpa Pods
kubectl get pods -n kube-system | grep vpa
kubectl get crd | grep verticalpodautoscaler
```

The three Pods are `vpa-recommender`, `vpa-updater` and `vpa-admission-controller`.

### 3c. The VPA (recommendation only)

```yaml
# vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-vpa
  namespace: luminary
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web                      # must match the Deployment name
  updatePolicy:
    updateMode: "Off"              # recommend only, no restarts
```

To let VPA apply changes automatically, switch to `"Auto"` and add safe limits:

```yaml
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
      - containerName: "*"
        minAllowed:
          cpu: 50m
          memory: 64Mi
        maxAllowed:
          cpu: "1"
          memory: 1Gi
```

### 3d. Apply and read the result

```bash
kubectl apply -f vpa.yaml
kubectl describe vpa web-vpa -n luminary
```

Wait a few minutes, then look at the bottom of the output:

```
Recommendation:
  Container Recommendations:
    Container Name:  web
    Lower Bound:     Cpu: 25m,  Memory: 64Mi
    Target:          Cpu: 50m,  Memory: 100Mi
    Upper Bound:     Cpu: 200m, Memory: 300Mi
```

| Field | Meaning |
|---|---|
| **Target** | The value VPA recommends. Use this one. |
| **Lower Bound** | Below this, the Pod will probably struggle |
| **Upper Bound** | Above this, the resources are probably wasted |

### 3e. VPA problems

| Problem | Cause and fix |
|---|---|
| No recommendation yet | VPA needs time to collect usage. Wait a few minutes to hours. |
| `Auto` mode causes a brief outage | The Pod is restarted. With only 1 replica the app goes down for a moment. |
| Pod `Pending` after resize | VPA made it bigger than any node can fit. Set `maxAllowed`. |

---

## Cheat Sheet

**Use them together safely:** do not let HPA and VPA both act on CPU or memory for the same app. HPA adds Pods while VPA resizes them, and they fight each other.

**A good practical pattern:**

1. Run VPA in `Off` mode to learn the right requests.
2. Set those requests yourself in the Deployment.
3. Let HPA handle traffic changes.

**All the commands in one place:**

```bash
# metrics-server
kubectl top nodes
kubectl top pods -A

# HPA
kubectl get hpa -n luminary -w
kubectl describe hpa web-hpa -n luminary

# VPA
kubectl get vpa -n luminary
kubectl describe vpa web-vpa -n luminary
```
