# Kubernetes Jobs

Jobs are for workloads that need to run once, finish successfully, and stop — database migrations, one-off scripts, backups, batch processing. Unlike a Deployment that keeps restarting pods indefinitely, a Job tracks completion and retries failed pods until the task succeeds.

---

## How It Works

```
Job → Pod → Task Executes → Pod exits 0 → Job marked Complete
```

If the Pod fails, the Job creates a new one and retries (up to `backoffLimit`). Once the required number of successful completions is reached, the Job stops. The record stays in Kubernetes so you can inspect logs afterward.

---

## Key Fields

| Field | What It Does |
|---|---|
| `completions` | How many successful Pod runs are needed before the Job is done |
| `parallelism` | How many Pods run simultaneously |
| `backoffLimit` | How many retries before the Job is marked Failed |
| `restartPolicy` | `Never` = new Pod on each retry, `OnFailure` = same Pod restarts |

---

## Example 1 — Quick Test

Minimal Job to confirm the setup works. Runs once and exits.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
  namespace: myspace
spec:
  completions: 1
  parallelism: 1
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: hello
          image: busybox
          command:
            - sh
            - -c
            - echo "Job ran at $(date)"
```

---

## Example 2 — Database Migration

Runs a migration script against MySQL before deploying a new app version. `backoffLimit: 3` means Kubernetes retries up to 3 times before giving up.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  namespace: myspace
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migrate
          image: shubhamm18/diffpods:01
          command:
            - sh
            - -c
            - |
              echo "Running DB migration..."
              node migrate.js
              echo "Migration complete"
          env:
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: mysql-config
                  key: DB_HOST
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_ROOT_PASSWORD
```

---

## Example 3 — Parallel Batch Processing

Processes 5 tasks in total, running 2 Pods at a time. Kubernetes runs 2 Pods simultaneously until 5 successful completions are recorded.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-processor
  namespace: myspace
spec:
  completions: 5
  parallelism: 2
  backoffLimit: 2
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: processor
          image: busybox
          command:
            - sh
            - -c
            - |
              echo "Processing on pod: $HOSTNAME"
              sleep 5
              echo "Done"
```

---

## Commands

```bash
# apply
kubectl apply -f job.yaml

# check job status
kubectl get jobs -n myspace

# check pods created by the job
kubectl get pods -n myspace

# view logs
kubectl logs <pod-name> -n myspace

# inspect events and conditions
kubectl describe job db-migration -n myspace

# clean up completed jobs manually
kubectl delete job db-migration -n myspace
```

---

## Job vs CronJob vs Deployment

| Use Case | Resource |
|---|---|
| One-time task | Job |
| Scheduled / repeated task | CronJob |
| Long-running application | Deployment |

---

## Notes

- `restartPolicy: Never` vs `OnFailure` — `Never` creates a fresh Pod on each retry (clean state). `OnFailure` restarts the same Pod. For migrations, use `OnFailure` to avoid running the script twice in parallel.
- Completed Jobs and their Pods are not deleted automatically. Set `ttlSecondsAfterFinished` on the Job spec for auto-cleanup.
- Jobs guarantee at-least-once execution, not exactly-once. Make your task idempotent if retries are possible.
