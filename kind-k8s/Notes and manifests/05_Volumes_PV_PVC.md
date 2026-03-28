# Volumes, Persistent Volumes (PV) & Persistent Volume Claims (PVC)

Pods are ephemeral. When a Pod is deleted or restarted, any data stored inside the container is lost. For applications like databases, log stores, or file uploads, data must survive Pod restarts. Kubernetes solves this with a three-layer storage architecture.

---

## The Problem with Regular Container Storage

```
Pod restarts → container filesystem is wiped → data is gone
```

This is fine for stateless apps. It is not fine for databases, file uploads, or anything that needs to outlive a Pod.

---

## Layer 1 — Volume (Pod-level, Temporary)

A Volume is storage attached directly to a Pod. It survives container restarts within the same Pod, but is deleted when the Pod itself is deleted.

```
Pod deleted → Volume deleted → data gone
```

Use cases: cache, temp files, sharing data between containers in the same Pod.

> Volumes are not suitable for databases or anything requiring long-term persistence.

---

## Layer 2 — Persistent Volume (PV)

A PersistentVolume represents actual physical storage made available to the cluster. It is a cluster-level resource — independent of any Pod or namespace. It exists even after the Pod that used it is deleted.

Backing storage types:
- `hostPath` — local disk on the node (dev/testing only)
- Cloud disks — AWS EBS, GCE PD
- Network storage — NFS

Think of a PV as a real hard disk that has been made available to Kubernetes.

---

## Layer 3 — Persistent Volume Claim (PVC)

A PVC is a request for storage made by an application. Pods never mount a PV directly — they always go through a PVC.

```
Pod → PVC → PV → Physical Storage
```

This separation keeps apps portable. The PVC says "I need 1Gi with ReadWriteOnce access." Kubernetes finds a matching PV and binds them. If no PV matches, the PVC stays unbound and the Pod won't start.

---

## Volume vs Persistent Volume

| Feature | Volume | Persistent Volume |
|---|---|---|
| Scope | Pod-level | Cluster-level |
| Lifetime | Pod lifetime | Independent of Pods |
| Data persists after Pod deletion | No | Yes |
| Suitable for databases | No | Yes |

---

## Access Modes

| Mode | Short | Meaning |
|---|---|---|
| ReadWriteOnce | RWO | Mounted read-write by one node |
| ReadOnlyMany | ROX | Mounted read-only by many nodes |
| ReadWriteMany | RWX | Mounted read-write by many nodes |

Most cloud disk types (EBS, GCE PD) only support `ReadWriteOnce`. For `ReadWriteMany` you need NFS or a distributed storage system.

---

## YAML Examples

### 1. Persistent Volume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/data    # only for local dev — never use hostPath in production
```

### 2. Persistent Volume Claim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: myspace
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Kubernetes binds this PVC to any available PV that has at least 1Gi and supports `ReadWriteOnce`.

### 3. Pod Using the PVC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: myspace
spec:
  containers:
    - name: app-container
      image: nginx
      volumeMounts:
        - mountPath: /data
          name: storage-volume
  volumes:
    - name: storage-volume
      persistentVolumeClaim:
        claimName: my-pvc
```

---

## Commands

```bash
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml

kubectl get pv
kubectl get pvc -n myspace

kubectl describe pvc my-pvc -n myspace
```

Check the `STATUS` column:
- `Bound` — PVC found a matching PV, Pod can start
- `Pending` — no matching PV available, Pod won't start

---

## StatefulSet — Automatic PVC per Pod

StatefulSets use `volumeClaimTemplates` to automatically create one PVC per Pod replica. This is the correct way to run databases in Kubernetes.

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

This creates `data-mysql-0`, `data-mysql-1`, etc. — each pod gets its own dedicated persistent storage.

---

## Production Pattern — StorageClass (Dynamic Provisioning)

In production, you don't create PVs manually. You define a StorageClass, and Kubernetes provisions PVs on demand when a PVC is created.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard   # maps to a cloud disk type
  resources:
    requests:
      storage: 5Gi
```

Cloud providers (EKS, GKE, AKS) come with StorageClasses pre-configured.

---

## Common Mistakes

| Mistake | Why It's Wrong |
|---|---|
| Storing database data in the container filesystem | Lost on every Pod restart |
| Expecting a plain Volume to persist data across Pod deletions | Volumes are Pod-scoped |
| Using `hostPath` in production | Ties your data to a specific node |
| Mounting a PV directly into a Pod without a PVC | Not how Kubernetes storage works |
