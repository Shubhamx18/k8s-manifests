# Kubernetes Debugging — Every Real Case

When something breaks in Kubernetes, there is always a trail. This file covers every common failure scenario, what it looks like, where to look, and how to fix it. No theory — just diagnosis and resolution.

---

## The Debugging Mindset

Kubernetes has layers. When something doesn't work, you trace from the top down:

```
Is the Pod running?
    → No  : look at Pod events and status
    → Yes : is the container healthy?
        → No  : look at container logs
        → Yes : is the Service routing correctly?
            → No  : check selector, endpoints, ports
            → Yes : is Ingress routing correctly?
                → No  : check Ingress rules, controller logs
```

Always start with `kubectl get` to see status, then `kubectl describe` for events, then `kubectl logs` for what the container itself says.

---

## The Five Commands You Use 90% of the Time

```bash
kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace>
kubectl get endpoints <service-name> -n <namespace>
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

---

## Pod Failures

### CrashLoopBackOff

**What it means:** The container starts, crashes, Kubernetes restarts it, it crashes again. After repeated failures, Kubernetes backs off with increasing wait times between restarts.

**How to diagnose:**

```bash
kubectl get pods -n myspace
# STATUS shows CrashLoopBackOff

kubectl logs <pod-name> -n myspace
# Shows what the app printed before it crashed

kubectl logs <pod-name> -n myspace --previous
# Shows logs from the PREVIOUS crash — most useful when the container restarts too fast
```

**Common causes and fixes:**

| Cause | What You See in Logs | Fix |
|---|---|---|
| App crashes on startup | Application error / stack trace | Fix the app code or config |
| Missing environment variable | `undefined` or key error | Check ConfigMap / Secret keys match what app expects |
| Wrong DB connection string | `Connection refused` or `ENOTFOUND` | Verify `DB_HOST` matches the Service name exactly |
| Container command wrong | `exec: not found` or `No such file` | Fix the `command` / `args` in the Pod spec |
| Port mismatch | App starts but health check fails | Align `containerPort`, app listen port, and Service `targetPort` |

---

### ImagePullBackOff / ErrImagePull

**What it means:** Kubernetes cannot pull the container image.

```bash
kubectl describe pod <pod-name> -n myspace
# Events section will show exactly why the pull failed
```

**Common causes and fixes:**

| Cause | Fix |
|---|---|
| Image name or tag is wrong | Double-check `image: repo/name:tag` — typos are common |
| Image tag doesn't exist on Docker Hub | Verify the tag exists: `docker pull <image>` locally |
| Private registry, no credentials | Create an `imagePullSecret` and reference it in the Pod spec |
| No internet access from node | Check node network / firewall rules |

---

### Pending — Pod Never Starts

**What it means:** The Pod exists but has not been scheduled to any node.

```bash
kubectl describe pod <pod-name> -n myspace
# Look at Events at the bottom — scheduler will say why it can't place the pod
```

**Common causes and fixes:**

| Cause | Event Message | Fix |
|---|---|---|
| Not enough CPU/memory on any node | `Insufficient cpu` / `Insufficient memory` | Lower resource requests or add nodes |
| ResourceQuota exceeded | `exceeded quota` | Check quota with `kubectl describe quota -n myspace` |
| PVC not bound | `persistentvolumeclaim not found` | Check PVC status — it may be `Pending` waiting for a PV |
| Node selector / affinity mismatch | `didn't match node selector` | Fix `nodeSelector` or remove it |
| All nodes are tainted | `had taints that pod didn't tolerate` | Add tolerations or remove taints |

---

### OOMKilled — Out of Memory

**What it means:** The container exceeded its memory limit and was killed by the kernel.

```bash
kubectl describe pod <pod-name> -n myspace
# Last State: Terminated  Reason: OOMKilled
```

**Fix:** Increase the memory limit in the container spec:

```yaml
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"
```

Or find and fix the memory leak in the application.

---

### Init Container Failing

**What it means:** The Pod has an init container (runs before the main container) that is failing or never completing.

```bash
kubectl describe pod <pod-name> -n myspace
# Shows init container status separately

kubectl logs <pod-name> -c <init-container-name> -n myspace
# -c flag specifies which container to get logs from
```

Common cause: init container waits for a dependency (e.g., database) that isn't ready yet. Check if the dependency service and pod are actually running.

---

## Service & Networking Failures

### Service Has No Endpoints

**What it means:** The Service exists but is not routing to any Pod. Traffic goes nowhere.

```bash
kubectl get endpoints <service-name> -n myspace
# If output shows <none>, the selector matches zero running Pods
```

**Diagnosis:**

```bash
kubectl describe svc <service-name> -n myspace
# Shows the selector the Service is using

kubectl get pods -n myspace --show-labels
# Compare Pod labels against the Service selector
```

**Fix:** The label on the Pod template must exactly match the Service `selector`. One typo (`app: backend` vs `app: Backend`) breaks it entirely.

---

### Can't Reach a Service from Another Pod

**What it means:** A pod inside the cluster cannot connect to a service that appears to be running.

**Step 1 — Verify the service exists and has endpoints:**

```bash
kubectl get svc -n myspace
kubectl get endpoints <service-name> -n myspace
```

**Step 2 — Test DNS resolution from inside the cluster:**

```bash
# Spin up a temporary debug pod
kubectl run debug --image=busybox --rm -it --restart=Never -n myspace -- sh

# Inside the pod:
nslookup <service-name>
nslookup <service-name>.<namespace>.svc.cluster.local
wget -qO- http://<service-name>:<port>
```

**Common causes and fixes:**

| Cause | Fix |
|---|---|
| Wrong service name in app config | Service name must exactly match what's in `DB_HOST` or connection string |
| Wrong port | Check `port` (what clients call) vs `targetPort` (what container listens on) |
| Pod in different namespace | Use full DNS: `service.namespace.svc.cluster.local` |
| NetworkPolicy blocking traffic | Check if any NetworkPolicy is applied to the namespace |

---

### NodePort Not Accessible from Outside

```bash
kubectl get nodes -o wide
# Get the node's INTERNAL-IP

curl http://<node-ip>:<nodePort>
```

**Common causes:**

| Cause | Fix |
|---|---|
| Firewall blocking the port | Open the NodePort range (30000–32767) in firewall rules |
| Wrong node IP | Use the node's external IP if accessing from outside the host |
| Service selector not matching | Check endpoints as above |

---

## Storage Failures

### PVC Stuck in Pending

**What it means:** The PersistentVolumeClaim cannot find a matching PersistentVolume to bind to.

```bash
kubectl get pvc -n myspace
# STATUS shows Pending

kubectl describe pvc <pvc-name> -n myspace
# Events will say why binding failed
```

**Common causes and fixes:**

| Cause | Fix |
|---|---|
| No PV exists that matches | Create a PV with matching `storage`, `accessMode`, and `storageClassName` |
| Access mode mismatch | PVC wants `ReadWriteMany` but PV only offers `ReadWriteOnce` |
| Storage size mismatch | PVC requests 2Gi but only a 1Gi PV exists — PV must be >= PVC request |
| StorageClass doesn't exist | Check `kubectl get storageclass` and use a name that exists |
| No dynamic provisioner | On KIND/bare-metal, PVs must be created manually unless a provisioner is installed |

---

### Pod Can't Mount the Volume

```bash
kubectl describe pod <pod-name> -n myspace
# Events: FailedMount or Unable to attach volume
```

**Common causes:**

| Cause | Fix |
|---|---|
| PVC is still Pending | Fix the PVC first — pod waits until PVC is Bound |
| Volume already mounted on another node | `ReadWriteOnce` can only be mounted on one node at a time |
| Wrong `claimName` in pod spec | The name in `persistentVolumeClaim.claimName` must match exactly |

---

## ConfigMap & Secret Failures

### Environment Variable is Empty or Undefined

**What it means:** The app reads an env var and gets nothing, causing a crash or connection failure.

```bash
# Exec into the running pod and check env vars directly
kubectl exec -it <pod-name> -n myspace -- env | grep DB_
```

**Common causes:**

| Cause | Fix |
|---|---|
| Key name in ConfigMap doesn't match `key:` in Pod spec | They must be identical — case sensitive |
| ConfigMap is in a different namespace | ConfigMap must be in the same namespace as the Pod |
| Secret value has trailing newline | Use `stringData` instead of `data` when writing Secrets manually |
| ConfigMap not applied yet | `kubectl get configmap -n myspace` to verify it exists |

---

### Secret Not Found

```bash
kubectl describe pod <pod-name> -n myspace
# Error: secret "mysql-secret" not found
```

**Fix:** Apply the Secret before the Pod. The Pod will fail to start if a referenced Secret doesn't exist.

```bash
kubectl get secret -n myspace
kubectl describe secret mysql-secret -n myspace
```

---

## Deployment & Rollout Failures

### Rollout Stuck — Not Progressing

```bash
kubectl rollout status deployment/<name> -n myspace
# Waiting for deployment to finish... (hangs)

kubectl describe deployment <name> -n myspace
# Look at Conditions section — will show why it's stuck
```

**Common causes:**

| Cause | Fix |
|---|---|
| New pods are crashing | Fix the underlying pod crash (see CrashLoopBackOff above) |
| Image pull failing | Fix the image name or credentials |
| `maxUnavailable: 0` and `maxSurge: 0` | Can't roll out — at least one must be > 0 |
| Insufficient cluster resources | New pods can't be scheduled — check node capacity |

**Rollback immediately:**

```bash
kubectl rollout undo deployment/<name> -n myspace
```

---

### Pods Not Updating After `kubectl apply`

**What it means:** You changed the image or config but the old pods are still running.

```bash
kubectl describe deployment <name> -n myspace
# Check the image version shown — it may not have updated
```

**Common causes:**

| Cause | Fix |
|---|---|
| Using `image: nginx` with no tag — image didn't change | Pin a specific tag so Kubernetes detects the change |
| ConfigMap changed but deployment wasn't restarted | ConfigMap changes don't auto-restart pods — force a rollout: `kubectl rollout restart deployment/<name> -n myspace` |

---

## Ingress Failures

### 404 — Path Not Found

```bash
kubectl describe ingress <name> -n myspace
# Shows the routing rules as Kubernetes sees them
```

**Common causes:**

| Cause | Fix |
|---|---|
| `rewrite-target` annotation missing on path-based routing | App receives `/student/...` instead of `/` — add the annotation |
| Wrong `pathType` | Use `Prefix` not `Exact` unless you need exact matching |
| Backend service name wrong | Service name in Ingress must match exactly |
| Backend service port wrong | Must match the Service's `port`, not `targetPort` |

---

### 502 Bad Gateway

**What it means:** Ingress controller reached the Service but got no valid response from the Pod.

```bash
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller
# Shows which upstream it tried and what response it got

kubectl get endpoints <service-name> -n myspace
# If <none>, the service has no healthy pods
```

**Fix:** The backing pods are not healthy. Check pod status and logs.

---

### Ingress Has No Address

```bash
kubectl get ingress -n myspace
# ADDRESS column is empty
```

**Cause:** No Ingress Controller is running, or it's not healthy.

```bash
kubectl get pods -n ingress-nginx
kubectl describe pod <controller-pod> -n ingress-nginx
```

---

## StatefulSet Failures

### StatefulSet Pod Stuck in Pending

StatefulSets create pods in order (0, 1, 2). If `mysql-0` is stuck, `mysql-1` will never start.

```bash
kubectl describe pod mysql-0 -n myspace
# Usually a PVC binding issue
```

Fix the PVC for `mysql-0` first — the entire StatefulSet unblocks once it's running.

---

### StatefulSet Pod Lost Its Data After Restart

```bash
kubectl get pvc -n myspace
# Check if mysql-data-mysql-0 still exists and is Bound
```

If the PVC was accidentally deleted, the data is gone. StatefulSet PVCs are not deleted automatically when you delete the StatefulSet — you have to explicitly delete them. This is a protection mechanism.

---

## Job & CronJob Failures

### Job Never Completes

```bash
kubectl describe job <job-name> -n myspace
kubectl logs <job-pod-name> -n myspace
```

**Common causes:**

| Cause | Fix |
|---|---|
| Task exits with non-zero code | Fix the script — check logs for the actual error |
| `backoffLimit` reached | Job is marked Failed — fix the issue and re-apply |
| Dependency not ready (e.g., DB not up) | Add an init container or retry logic in the script |

---

### CronJob Not Firing

```bash
kubectl get cronjob -n myspace
# Check LAST SCHEDULE column — if it says <none>, it has never fired

kubectl describe cronjob <name> -n myspace
```

**Common causes:**

| Cause | Fix |
|---|---|
| Schedule expression is wrong | Test at [crontab.guru](https://crontab.guru) |
| `suspend: true` is set | Set `suspend: false` |
| Previous job still running with `concurrencyPolicy: Forbid` | Wait for it to finish or delete the stuck job |
| Cluster timezone mismatch | CronJob uses the control plane timezone — verify if schedules seem off |

---

## Namespace Issues

### Resource Not Found — But You Know It Exists

Almost always a namespace issue.

```bash
kubectl get pods
# Only shows default namespace

kubectl get pods -n myspace
# Shows what you're looking for
```

Set a default namespace so you stop making this mistake:

```bash
kubectl config set-context --current --namespace=myspace
```

---

### Namespace Stuck in Terminating

```bash
kubectl get ns
# STATUS: Terminating — has been like this for minutes
```

**Cause:** There are finalizers on resources inside the namespace that are blocking deletion.

```bash
kubectl get all -n <namespace>
# See what's still there

# Force remove finalizers on the namespace (last resort)
kubectl get namespace <name> -o json | \
  jq '.spec.finalizers = []' | \
  kubectl replace --raw "/api/v1/namespaces/<name>/finalize" -f -
```

---

## General Debug Toolkit

### Run a Temporary Debug Pod

The most useful thing you can do — get a shell inside the cluster to test DNS, connectivity, and endpoints directly.

```bash
# busybox — for DNS and basic HTTP
kubectl run debug --image=busybox --rm -it --restart=Never -n myspace -- sh

# curl-capable image — for testing HTTP endpoints
kubectl run debug --image=curlimages/curl --rm -it --restart=Never -n myspace -- sh

# mysql client — for testing DB connectivity
kubectl run debug --image=mysql:8.0 --rm -it --restart=Never -n myspace -- bash
```

Inside the debug pod:

```bash
nslookup mysql                          # test DNS for a service
wget -qO- http://backend-service:80     # test HTTP
mysql -h mysql -u root -p              # test DB connection
env                                    # see all environment variables
```

---

### Read Events for the Whole Namespace

```bash
kubectl get events -n myspace --sort-by='.lastTimestamp'
```

This gives you a timeline of everything that happened — scheduling decisions, image pulls, volume mounts, probe failures. When you don't know where to start, start here.

---

### Exec Into a Running Container

```bash
kubectl exec -it <pod-name> -n myspace -- bash
# or if bash isn't available:
kubectl exec -it <pod-name> -n myspace -- sh

# For a pod with multiple containers, specify which one:
kubectl exec -it <pod-name> -c <container-name> -n myspace -- sh
```

Use this to check env vars, test connectivity from inside the container, inspect mounted files.

---

### Check Resource Usage

```bash
# Node-level usage
kubectl top nodes

# Pod-level usage
kubectl top pods -n myspace
```

If `kubectl top` doesn't work, the metrics-server is not installed.

---

## Quick Reference — Status to Action

| Pod Status | First Command to Run |
|---|---|
| `CrashLoopBackOff` | `kubectl logs <pod> --previous -n myspace` |
| `ImagePullBackOff` | `kubectl describe pod <pod> -n myspace` → check Events |
| `Pending` | `kubectl describe pod <pod> -n myspace` → check Events |
| `OOMKilled` | `kubectl describe pod <pod> -n myspace` → increase memory limit |
| `Running` but not working | `kubectl get endpoints <svc> -n myspace` |
| `Terminating` (stuck) | `kubectl delete pod <pod> --force --grace-period=0 -n myspace` |
| `Init:CrashLoopBackOff` | `kubectl logs <pod> -c <init-container-name> -n myspace` |

---

## The Debugging Loop

```
1. kubectl get pods -n myspace
       → find which pod is unhealthy

2. kubectl describe pod <name> -n myspace
       → read Events section at the bottom

3. kubectl logs <name> -n myspace --previous
       → read what the app said before it died

4. kubectl exec -it <name> -n myspace -- sh
       → get inside and test things manually

5. kubectl get events -n myspace --sort-by='.lastTimestamp'
       → when you still have no idea, read the full timeline
```

Repeat until fixed.
