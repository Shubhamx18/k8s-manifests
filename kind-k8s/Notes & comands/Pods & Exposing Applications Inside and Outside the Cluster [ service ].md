# Kubernetes Services

Pods are ephemeral — they get new IPs every time they restart. Services sit in front of them and provide a stable virtual IP and DNS name that doesn't change, no matter how many times the underlying Pods come and go. Traffic is routed to Pods via label selectors, and kube-proxy handles the actual load balancing.

```
Client → Service (stable ClusterIP / DNS) → Pod(s)
```

---

## Service Types

| Type           | Access          | When to use                              |
| -------------- | --------------- | ---------------------------------------- |
| `ClusterIP`    | Inside cluster  | Internal communication between services  |
| `NodePort`     | Outside cluster | Local dev, quick testing                 |
| `LoadBalancer` | Outside cluster | Production on cloud (EKS, GKE, AKS)      |
| `Headless`     | Inside cluster  | StatefulSets, direct Pod DNS resolution  |

---

## Example 1 — ClusterIP (Default)

Used for internal service-to-service communication. The database Service in the MySQL + Node.js setup is a good example — the Node app reaches MySQL via its ClusterIP DNS name, never directly by Pod IP.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: myspace
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
```

Inside the cluster, any pod can reach this at `backend-service.myspace.svc.cluster.local:80`. The short form `backend-service` works if the calling pod is in the same namespace.

---

## Example 2 — NodePort

Opens a port on every Node (range 30000–32767) and forwards traffic to the Service. Fine for local testing on KIND or bare-metal, but not for production — you're exposing a port on the host and there's no SSL or routing logic.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: myspace
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 3000
      nodePort: 30080
```

Access it at `<NodeIP>:30080`. On KIND:

```bash
kubectl get nodes -o wide  # grab the internal IP
curl http://<node-ip>:30080
```

---

## Example 3 — LoadBalancer

Provisions a cloud load balancer and assigns a public IP automatically. This is how you expose services in production on managed Kubernetes.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: myspace
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 3000
```

```bash
kubectl get svc frontend-service -n myspace
# EXTERNAL-IP column will show the assigned public IP once provisioned
```

On a local cluster (KIND, minikube), `EXTERNAL-IP` stays `<pending>` forever — you'd need MetalLB or just use NodePort for local work.

---

## Example 4 — Headless Service

Setting `clusterIP: None` disables the virtual IP entirely. Instead of load balancing, DNS returns the IPs of individual Pods. Required for StatefulSets so each pod gets a stable, addressable DNS name.

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

Each StatefulSet pod becomes individually reachable:

```
mysql-0.mysql.myspace.svc.cluster.local
mysql-1.mysql.myspace.svc.cluster.local
```

Use this any time you need to connect to a specific pod rather than any pod — replicated databases, Kafka brokers, Zookeeper nodes.

---

## Deployment to go with these Services

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: myspace
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend       # must match the Service selector
    spec:
      containers:
        - name: backend
          image: nginx:alpine
          ports:
            - containerPort: 8080
```

The label `app: backend` on the Pod template must match the `selector` in the Service exactly — if they don't match, the Service has no endpoints and traffic goes nowhere.

---

## Port Forwarding (for debugging)

Lets you hit a service or pod directly from your local machine without creating any external exposure.

```bash
# forward local 8080 to pod port 80
kubectl port-forward pod/mypod 8080:80 -n myspace

# forward local 8080 to a service
kubectl port-forward svc/backend-service 8080:80 -n myspace
```

Useful for checking an internal service's response, connecting to a database, or debugging without touching the Service type.

---

## Commands

```bash
# list services
kubectl get svc -n myspace

# see which pods a service is routing to
kubectl get endpoints backend-service -n myspace

# inspect service config and events
kubectl describe svc backend-service -n myspace

# quickly expose a deployment without writing yaml
kubectl expose deployment backend --type=NodePort --port=80 -n myspace
```

---

## Service vs Ingress

| Feature          | Service              | Ingress                    |
| ---------------- | -------------------- | -------------------------- |
| Works at         | L4 (TCP/UDP)         | L7 (HTTP/HTTPS)            |
| Routing logic    | None (label select)  | Path-based, host-based     |
| TLS termination  | No                   | Yes                        |
| Typical use      | Internal comms       | External web traffic       |

The pattern in production: Services handle internal routing, one Ingress handles all external HTTP traffic and routes to the right Services.

---

## Notes

- If `kubectl get endpoints <service>` shows `<none>`, the selector doesn't match any running Pod labels — that's the most common reason a Service appears to work but gets no traffic
- `port` is what clients call, `targetPort` is what the container actually listens on — they don't have to match
- Services don't do health checking themselves — they rely on the Pod's readiness probe to know whether to include a Pod in rotation
