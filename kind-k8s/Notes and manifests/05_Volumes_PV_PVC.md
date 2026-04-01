# Kubernetes Storage — Volumes, PV & PVC

> Pods are ephemeral. When a Pod dies, its data dies too.  
> Kubernetes solves this with a **three-layer storage architecture**.

---

## Why Storage Matters

```
Pod restarts  ->  container filesystem wiped  ->  data gone
```

- Fine for **stateless apps**
- Not fine for **databases, file uploads, logs**

---

## The Three Layers

```
Pod  ->  PVC  ->  PV  ->  Physical Storage
```

| Layer | Name | Scope | Survives Pod Deletion? |
|---|---|---|---|
| 1 | Volume | Pod-level | No |
| 2 | Persistent Volume (PV) | Cluster-level | Yes |
| 3 | Persistent Volume Claim (PVC) | Namespace-level | Yes |

---

## Layer 1 — Volume (Temporary)

Attached directly to a Pod. Lives and dies with the Pod.

**Use for:** cache, temp files, sharing data between containers in the same Pod.  
**Not for:** databases or anything needing long-term persistence.

---

## Layer 2 — Persistent Volume (PV)

A cluster-level resource representing **real physical storage** made available to Kubernetes.

**Backing storage types:**

| Type | Description |
|---|---|
| `hostPath` | Local disk on the node — dev/testing only |
| AWS EBS / GCE PD | Cloud block storage |
| NFS | Network file storage |

> Think of a PV as a hard disk that Kubernetes knows about.

---

## Layer 3 — Persistent Volume Claim (PVC)

A **request for storage** made by an application. Pods never mount a PV directly — always through a PVC.

Kubernetes binds a PVC to a PV when all three match:

| Field | Must match |
|---|---|
| `storageClassName` | Exact match |
| `accessModes` | Must overlap |
| `storage` | PVC request <= PV capacity |

If no PV matches -> PVC stays `Pending` -> Pod won't start.

---

## Access Modes

| Mode | Short | Meaning |
|---|---|---|
| ReadWriteOnce | RWO | Read-write by **one node** |
| ReadOnlyMany | ROX | Read-only by **many nodes** |
| ReadWriteMany | RWX | Read-write by **many nodes** |

> AWS EBS and GCE PD only support `ReadWriteOnce`.  
> For `ReadWriteMany` you need NFS or a distributed storage system.

---

## YAML Examples — Luminary Project

### 1. Persistent Volume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: luminary-pv

spec:
  capacity:
    storage: 1Gi

  accessModes:
    - ReadWriteOnce

  storageClassName: manual          # label used for manual binding

  persistentVolumeReclaimPolicy: Retain   # keep data after PVC is deleted

  hostPath:
    path: /mnt/data                 # local dev only — never use in production
```

---

### 2. Persistent Volume Claim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: luminary-pvc
  namespace: luminary

spec:
  accessModes:
    - ReadWriteOnce               # matches PV

  resources:
    requests:
      storage: 1Gi               # matches PV capacity

  storageClassName: manual       # matches PV exactly — required for binding
```

> **Binding checklist:**
> - `storageClassName: manual` — matches PV
> - `accessModes: ReadWriteOnce` — matches PV
> - `storage: 1Gi` — within PV capacity

---

### 3. Pod Using the PVC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: luminary

spec:
  containers:
    - name: app-container
      image: nginx
      volumeMounts:
        - mountPath: /data          # path inside the container
          name: storage-volume

  volumes:
    - name: storage-volume
      persistentVolumeClaim:
        claimName: luminary-pvc     # references the PVC above
```

---

## Commands

```bash
# Apply storage resources
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml

# Check status
kubectl get pv
kubectl get pvc -n luminary

# Debug binding issues
kubectl describe pvc luminary-pvc -n luminary
```

**PVC status meanings:**

| Status | Meaning |
|---|---|
| `Bound` | Matched a PV — Pod can start |
| `Pending` | No matching PV found — Pod won't start |

---

## StatefulSet — One PVC per Pod (Databases)

StatefulSets use `volumeClaimTemplates` to auto-create a dedicated PVC per replica.  
This is the **correct way to run databases** in Kubernetes.

```yaml
volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

Creates: `data-mysql-0`, `data-mysql-1`, etc. — each Pod gets its own storage.

---

## Production Pattern — Dynamic Provisioning with StorageClass

In production, **never create PVs manually**.  
Define a `StorageClass` — Kubernetes provisions PVs automatically when a PVC is created.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard        # maps to a cloud disk type (EBS, GCE PD, etc.)
  resources:
    requests:
      storage: 5Gi
```

> Cloud providers (EKS, GKE, AKS) come with StorageClasses pre-configured.

---

## Common Mistakes

| Mistake | Why It's Wrong |
|---|---|
| Storing DB data in container filesystem | Lost on every Pod restart |
| Expecting a plain Volume to persist across Pod deletions | Volumes are Pod-scoped |
| Using `hostPath` in production | Ties data to a specific node — breaks if Pod moves |
| Mounting a PV directly into a Pod | Not how Kubernetes storage works — always use a PVC |
| Mismatched `storageClassName` between PV and PVC | PVC stays `Pending`, Pod never starts |
| PVC requests more storage than PV capacity | No binding — PVC stays `Pending` |
