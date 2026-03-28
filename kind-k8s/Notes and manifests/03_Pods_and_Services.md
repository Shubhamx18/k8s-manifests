# Pods & Services

## Pods

A Pod is the smallest deployable unit in Kubernetes. It wraps one or more containers that share the same network namespace and storage. Pods are ephemeral — they get a new IP every time they restart, which is why Services exist.

---

## Services

A Service sits in front of Pods and provides a stable virtual IP and DNS name that doesn't change, regardless of how many times the underlying Pods come and go. Traffic is routed to Pods via label selectors. kube-proxy handles the actual load balancing.

```
Client → Service (stable ClusterIP / DNS) → Pod(s)
```

---

## Service Types

| Type | Access | When to Use |
|---|---|---|
| `ClusterIP` | Inside cluster only | Internal service-to-service communication |
| `NodePort` | Outside cluster | Local dev, quick testing |
| `LoadBalancer` | Outside cluster | Production on cloud (EKS, GKE, AKS) |
| `Headless` | Inside cluster | StatefulSets, direct Pod DNS resolution |

---

## ClusterIP (Default)

Used for internal communication. The Node.js app in the MySQL setup reaches MySQL via its ClusterIP DNS name — never directly by Pod IP.

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

Inside the cluster, any pod reaches this at `backend-service.myspace.svc.cluster.local:80`. Short form `backend-service` works within the same namespace.

---

## NodePort

Opens a port on every Node (range 30000–32767) and forwards traffic to the Service. Fine for local KIND testing, not for production.

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
kubectl get nodes -o wide   # grab the internal IP
curl http://<node-ip>:30080
```

---

## LoadBalancer

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
# EXTERNAL-IP column shows the assigned public IP once provisioned
```

> On a local cluster (KIND, Minikube), `EXTERNAL-IP` stays `<pending>` forever. Use NodePort for local work, or MetalLB if you need LoadBalancer locally.

---

## Headless Service

Setting `clusterIP: None` disables the virtual IP entirely. DNS returns the IPs of individual Pods directly. Required for StatefulSets so each pod gets a stable, individually addressable DNS name.

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

Use this when you need to connect to a specific pod — replicated databases, Kafka brokers, Zookeeper nodes.

---

## Deployment to Go with a Service

The label on the Pod template **must match** the selector in the Service exactly. If they don't match, the Service has no endpoints and traffic goes nowhere.

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
        app: backend        # must match the Service selector
    spec:
      containers:
        - name: backend
          image: nginx:alpine
          ports:
            - containerPort: 8080
```

---

## Port Forwarding (Debugging)

Lets you hit a service or pod directly from your local machine without any external exposure.

```bash
# forward local 8080 to pod port 80
kubectl port-forward pod/mypod 8080:80 -n myspace

# forward local 8080 to a service
kubectl port-forward svc/backend-service 8080:80 -n myspace
```

---

## Commands

```bash
# list services
kubectl get svc -n myspace

# see which pods a service is routing to
kubectl get endpoints backend-service -n myspace

# inspect service config and events
kubectl describe svc backend-service -n myspace

# quickly expose a deployment without YAML
kubectl expose deployment backend --type=NodePort --port=80 -n myspace
```

---

## Service vs Ingress

| Feature | Service | Ingress |
|---|---|---|
| Works at | L4 (TCP/UDP) | L7 (HTTP/HTTPS) |
| Routing logic | Label selection only | Path-based, host-based |
| TLS termination | No | Yes |
| Typical use | Internal communication | External web traffic |

The production pattern: Services handle all internal routing. One Ingress handles all external HTTP traffic and routes to the right Services.

---

## Notes

- `port` is what clients call, `targetPort` is what the container actually listens on — they don't have to match
- If `kubectl get endpoints <service>` shows `<none>`, the selector doesn't match any running Pod labels — the most common reason a Service gets no traffic
- Services don't do health checking — they rely on the Pod's readiness probe to decide whether to include a Pod in rotation
