# Kubernetes Ingress

An Ingress is a set of routing rules for external HTTP and HTTPS traffic entering a Kubernetes cluster. It sends requests to internal Services based on the hostname or URL path. The rules are only a specification: an **Ingress Controller** (for example ingress-nginx) must be running in the cluster to read them and do the actual routing.

---

## 1. Why Use Ingress?

Exposing every application with a `NodePort` or `LoadBalancer` Service creates problems:

- **No advanced routing.** Services cannot split traffic by URL path or hostname.
- **Cost and complexity.** A `LoadBalancer` Service normally provisions one cloud load balancer per Service, which adds cost and management overhead.
- **No central TLS handling.** Each Service would have to manage its own certificates.

Ingress solves this with **one entry point** that routes to many Services.

---

## 2. How a Request Flows

```
User ──> Ingress Controller ──> Service ──> Pods
              ▲
              │ reads
        Ingress rules (the Ingress resource)
```

1. The user sends a request to the cluster's entry point.
2. The Ingress Controller (an NGINX pod) receives it.
3. The controller matches the host and path against the Ingress rules.
4. It forwards the request to the matching Service, which sends it to a healthy Pod.

> Ingress resource = the rules. Ingress Controller = the software that enforces them. Without a controller, an Ingress does nothing.

---

## 3. Core Capabilities

- **Path-based routing:** send `/shop` to `shop-service` and `/blog` to `blog-service`.
- **Host-based routing:** send `shop.example.com` and `blog.example.com` to different Services.
- **TLS/SSL termination:** handle HTTPS certificates once, at the edge of the cluster.

---

## 4. Service Types Compared

| Capability | NodePort | LoadBalancer | Ingress |
|---|---|---|---|
| External access | Yes | Yes | Yes |
| One entry point for many Services | No | No | **Yes** |
| Path / host-based routing | No | No | **Yes** |
| TLS termination | No | No | **Yes** |
| Cost for many Services | Low (but exposes node ports) | High (one load balancer each) | **Low (one shared entry point)** |

Ingress usually sits in front of ClusterIP Services, and the controller itself is exposed through a single NodePort or LoadBalancer Service.

---

## 5. Important Rules

- **Same namespace.** An Ingress can only route to Services in its **own namespace**. If your Services are in `state`, the Ingress must also have `namespace: state`. Otherwise you get 503 errors.
- **`ingressClassName` must match your controller** (here `nginx`).
- **`pathType`:** `Prefix` matches the path and everything below it. `Exact` matches only that exact path.
- **`rewrite-target: /`** strips the matched prefix before sending the request to the backend, so `/shop` reaches the app as `/`. This works for simple apps, but apps that load assets from absolute paths (such as `/style.css`) may render broken behind a path prefix.
- **Services need endpoints.** If a Service selects no running Pods, the Ingress returns 503.

---

## 6. Configuration Examples

### Example 1: Path-based routing

Route `/shop` and `/blog` on a shared domain to separate Services.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /shop
            pathType: Prefix
            backend:
              service:
                name: shop-service
                port:
                  number: 80
          - path: /blog
            pathType: Prefix
            backend:
              service:
                name: blog-service
                port:
                  number: 80
```

### Example 2: Host-based routing

Route traffic by requested domain name.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: shop-service
                port:
                  number: 80
    - host: blog.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: blog-service
                port:
                  number: 80
```

### Example 3: TLS / HTTPS termination

The certificate is stored in a TLS Secret, which must be in the same namespace as the Ingress.

```bash
kubectl create secret tls shop-tls --cert=tls.crt --key=tls.key
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - shop.example.com
      secretName: shop-tls
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: shop-service
                port:
                  number: 80
```

---

## 7. Accessing an Ingress on a kind Cluster

A kind cluster runs inside Docker, so the ingress controller is not reachable from the host unless port 80 was mapped when the cluster was created. The simplest way to test is a port-forward to the controller:

```bash
# Terminal 1 (leave running)
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8082:80

# Terminal 2
curl http://localhost:8082/shop
curl http://localhost:8082/blog
```

To open it from another machine's browser, `localhost` will not work because it points at that machine. Either:

- bind to all interfaces: `kubectl port-forward --address 0.0.0.0 ...` and browse to `http://<server-ip>:8082/shop`, or
- use an SSH tunnel from your laptop: `ssh -L 8082:localhost:8082 user@<server-ip>`, then browse to `http://localhost:8082/shop`.

To avoid port-forwarding permanently, create the kind cluster with `extraPortMappings` for ports 80 and 443 and install the kind-specific ingress-nginx manifest.

---

## 8. Operation and Troubleshooting Commands

```bash
# List Ingress resources (add -n <namespace> or -A for all namespaces)
kubectl get ingress

# Show rules, backends and events
kubectl describe ingress app-ingress

# Check the controller is running
kubectl get pods -n ingress-nginx

# Controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller

# Check Services have endpoints (empty = no matching running Pods)
kubectl get endpoints -n <namespace>
```

| Symptom | Likely cause |
|---|---|
| Connection refused | Port-forward not running, wrong port, or the request is not reaching the server |
| 404 from nginx | No rule matches the host or path |
| 503 | Service has no endpoints, or Ingress and Service are in different namespaces |
| Page loads but CSS/JS/images are broken | App uses absolute asset paths behind a path prefix |
