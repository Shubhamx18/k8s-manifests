# ConfigMap, Secret & Full App — MySQL + Node.js on Kubernetes

A complete working Kubernetes setup: a Node.js app backed by a MySQL database, using ConfigMaps and Secrets to manage configuration cleanly. This covers StatefulSets, PVCs, headless Services, and how everything wires together inside a namespace.

---

## What Are ConfigMaps and Secrets?

| Resource | Purpose | Stores |
|---|---|---|
| ConfigMap | Non-sensitive configuration | DB host, port, database name, feature flags |
| Secret | Sensitive configuration | Passwords, API keys, tokens |

Both are injected into Pods as environment variables or mounted as files. The key difference is that Secret values are base64-encoded and can be encrypted at rest — they are not stored as plain text in etcd.

> Never hardcode config or credentials in your container image. Always use ConfigMaps and Secrets.

---

## Architecture

```
Node App (Deployment)
        |
        v
MySQL Service (Headless — clusterIP: None)
        |
        v
MySQL StatefulSet (mysql-0)
        |
        v
PersistentVolume (/var/lib/mysql)
```

---

## Why StatefulSet for MySQL?

Unlike a Deployment, a StatefulSet gives each Pod:
- A stable, predictable name (`mysql-0`, `mysql-1`)
- Its own PVC that doesn't get deleted when the Pod restarts
- A stable DNS entry via the headless Service

This is why databases run as StatefulSets, not Deployments.

---

## Manifests — Apply in This Order

Resources must exist before the things that depend on them.

### 1. Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myspace
```

---

### 2. ConfigMap

Holds non-sensitive database config shared across both MySQL and the Node app.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: myspace
data:
  MYSQL_DATABASE: mydatabase
  DB_HOST: mysql
  DB_PORT: "3306"
```

---

### 3. Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: myspace
type: Opaque
stringData:
  MYSQL_ROOT_PASSWORD: root
```

> `stringData` lets you write plain text — Kubernetes base64-encodes it automatically. In production, use Sealed Secrets or an external vault. Never commit real passwords to git.

---

### 4. MySQL Headless Service

`clusterIP: None` skips load balancing and gives the StatefulSet pod a stable DNS entry:

```
mysql-0.mysql.myspace.svc.cluster.local
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: myspace
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
    - port: 3306
```

---

### 5. MySQL StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: myspace
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
          envFrom:
            - configMapRef:
                name: mysql-config
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_ROOT_PASSWORD
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

`volumeClaimTemplates` automatically creates a PVC named `mysql-data-mysql-0` for the first pod. This PVC persists even if the pod is deleted and restarted.

---

### 6. Node.js Deployment

Each env var is pulled individually from the ConfigMap or Secret using `valueFrom`. This is more explicit than `envFrom` and makes it clear exactly which keys the app depends on.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-app
  namespace: myspace
spec:
  replicas: 1
  selector:
    matchLabels:
      app: node-app
  template:
    metadata:
      labels:
        app: node-app
    spec:
      containers:
        - name: node-app
          image: shubhamm18/diffpods:01
          ports:
            - containerPort: 3000
          env:
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: mysql-config
                  key: DB_HOST
            - name: DB_PORT
              valueFrom:
                configMapKeyRef:
                  name: mysql-config
                  key: DB_PORT
            - name: DB_NAME
              valueFrom:
                configMapKeyRef:
                  name: mysql-config
                  key: MYSQL_DATABASE
            - name: DB_USER
              value: root
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_ROOT_PASSWORD
```

---

### 7. Node.js Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: node-app-service
  namespace: myspace
spec:
  type: ClusterIP
  selector:
    app: node-app
  ports:
    - port: 3000
      targetPort: 3000
```

---

## Deploy

```bash
kubectl apply -f namespace.yaml
kubectl apply -f mysql-config.yaml
kubectl apply -f mysql-secret.yaml
kubectl apply -f mysql-service.yaml
kubectl apply -f mysql-statefulset.yaml
kubectl apply -f node-app-deployment.yaml
kubectl apply -f node-app-service.yaml
```

Or if everything is in one file:

```bash
kubectl apply -f k8s-complete.yaml
```

Verify everything came up:

```bash
kubectl get all -n myspace
```

---

## How Config Gets Into Pods — Two Ways

### envFrom — inject the entire ConfigMap as env vars

```yaml
envFrom:
  - configMapRef:
      name: mysql-config
```

Every key in the ConfigMap becomes an environment variable. Simple but less explicit.

### valueFrom — inject individual keys

```yaml
env:
  - name: DB_HOST
    valueFrom:
      configMapKeyRef:
        name: mysql-config
        key: DB_HOST
```

More verbose but explicit about what the container depends on. Prefer this for Secrets.

---

## How the Node App Connects to MySQL

The Node app connects using the service name `mysql` on port `3306`. Kubernetes DNS resolves `mysql` to the headless Service, which in turn resolves to the Pod IP of `mysql-0`. The connection string inside the app looks like:

```
mysql://root:<password>@mysql:3306/mydatabase
```

The app never needs to know the Pod's IP — the Service name stays stable regardless of restarts.

---

## Notes

- MySQL runs as a StatefulSet so the pod name (`mysql-0`) and PVC stay stable across restarts
- Data lives in the PVC — deleting the Pod does not delete the database
- The Node app image (`shubhamm18/diffpods:01`) is a custom build — swap it for your own
- For production Secrets management, look into Sealed Secrets, HashiCorp Vault, or AWS Secrets Manager
