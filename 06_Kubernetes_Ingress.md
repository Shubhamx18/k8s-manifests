# Kubernetes Ingress

Ingress exposes multiple services externally through a single entry point using HTTP/HTTPS routing. Instead of giving every service its own NodePort or LoadBalancer, you define routing rules — by path or hostname — and the Ingress Controller handles the rest.

---

## How It Works

```
User → Ingress Controller → Ingress Rules → Service → Pods
```

Two things are involved:

- **Ingress resource** — the YAML config that defines routing rules
- **Ingress Controller** — the actual running component that reads those rules and handles traffic

Without a controller deployed in the cluster, the Ingress YAML does nothing. The most common controller is NGINX. Cloud-managed options exist on EKS, GKE, and AKS.

---

## Ingress vs NodePort vs LoadBalancer

| Feature | NodePort | LoadBalancer | Ingress |
|---|---|---|---|
| External access | Yes | Yes | Yes |
| Route multiple services | No | No | Yes |
| Path / host routing | No | No | Yes |
| HTTPS / TLS | No | No | Yes |
| Cost at scale | High | Very high | Low |

> Ingress only handles HTTP/HTTPS (Layer 7). For raw TCP/UDP you need a LoadBalancer service or MetalLB.

---

## Install NGINX Ingress Controller (KIND)

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# wait for it to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

---

## Example 1 — Path-Based Routing

Route `/student` and `/admin` to different services on the same domain.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: myspace
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /student
            pathType: Prefix
            backend:
              service:
                name: student-service
                port:
                  number: 80
          - path: /admin
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 80
```

The `rewrite-target: /` annotation strips the path prefix before forwarding. Without it, your app receives `/student/...` instead of `/...` and likely returns 404s.

---

## Example 2 — Host-Based Routing

Different subdomains routed to different services. More common in production.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-ingress
  namespace: myspace
spec:
  ingressClassName: nginx
  rules:
    - host: student.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: student-service
                port:
                  number: 80
    - host: admin.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 80
```

For local testing, add entries to `/etc/hosts`:

```
127.0.0.1  student.example.com
127.0.0.1  admin.example.com
```

---

## Example 3 — TLS / HTTPS

Generate a self-signed cert, store it as a Secret, reference it in the Ingress.

```bash
# generate self-signed cert
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=student.example.com"

# create the TLS secret
kubectl create secret tls student-tls \
  --cert=tls.crt --key=tls.key \
  -n myspace
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  namespace: myspace
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - student.example.com
      secretName: student-tls
  rules:
    - host: student.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: student-service
                port:
                  number: 80
```

> In production, use cert-manager instead of manually creating TLS secrets. It auto-renews certificates from Let's Encrypt.

---

## Backing Services Required by These Examples

```yaml
apiVersion: v1
kind: Service
metadata:
  name: student-service
  namespace: myspace
spec:
  selector:
    app: student
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: student
  namespace: myspace
spec:
  replicas: 2
  selector:
    matchLabels:
      app: student
  template:
    metadata:
      labels:
        app: student
    spec:
      containers:
        - name: student
          image: nginx:alpine
          ports:
            - containerPort: 8080
```

---

## Commands

```bash
# list ingress resources
kubectl get ingress -n myspace

# inspect routing rules and events
kubectl describe ingress app-ingress -n myspace

# check ingress controller pods
kubectl get pods -n ingress-nginx

# check controller logs if traffic isn't routing correctly
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller
```

---

## Notes

- Always set `ingressClassName: nginx` — without it, newer clusters won't know which controller to use
- The `rewrite-target` annotation is the most commonly forgotten config in path-based routing — causes 404s in your app
- In production, use cert-manager for TLS — it handles certificate issuance and renewals automatically
- Ingress is HTTP/HTTPS only — for TCP or UDP, configure TCP proxying in the NGINX controller or use a LoadBalancer service
