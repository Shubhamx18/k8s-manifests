# Kubernetes Ingress

An Ingress acts as an intelligent traffic manager for a Kubernetes cluster. Positioned at the cluster boundary, it manages and routes external HTTP and HTTPS requests to internal cluster services.

---

## Overview & Architecture

### Why Use Ingress?

Standard Kubernetes service publication methods like `NodePort` or `LoadBalancer` introduce distinct operational challenges:

- **Limited Advanced Routing capabilities** — Standard services lack built-in support for path-based or host-based traffic split policies.

- **Cloud Infrastructure Dependency** — Provisioning dedicated cloud load balancers per service increases operational complexity and cloud spending across environments.

### Request Flow Architecture

An Ingress resource functions as an internal routing specification, dictating service destinations based on request paths or hostnames.

> *"Direct requests matching defined domain names or paths to designated internal cluster services."*

```
[ User Request ] ──> [ Ingress Controller ] ──> [ Ingress Rules ] ──> [ Service ] ──> [ Pods ]
```

---

## Core Capabilities

▸ **Path-Based Routing**  
Direct traffic using request paths (e.g., routing `/shop` requests to `shop-service` and `/blog` requests to `blog-service`).

▸ **Host-Based Routing**  
Route inbound requests based on the requested domain name or subdomain.

▸ **TLS / SSL Termination**  
Manage SSL certificates and terminate HTTPS traffic efficiently at the edge of the cluster.

---

## Service Type Comparison

| Capability | NodePort | LoadBalancer | Ingress |
|---|---|---|---|
| External Access | Yes | Yes | Yes |
| Single Entry Multi-Service Routing | No | No | **Yes** |
| Path / Host-Based Routing | No | No | **Yes** |
| SSL / TLS Termination | No | No | **Yes** |
| Resource Cost Efficiency | Moderate | High | **Optimal** |

---

## Configuration Examples

### Example 1 — Path-Based Routing

Route distinct request paths (`/shop` and `/blog`) on a shared domain to dedicated backend services.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
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

---

### Example 2 — Host-Based Routing

Direct traffic dynamically based on the requested host domain name.

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

---

### Example 3 — TLS / HTTPS Termination

Manage certificate termination securely at the Ingress controller boundary.

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

## Operation & Troubleshooting Commands

```bash
# List all configured Ingress resources
kubectl get ingress

# Inspect detailed routing configurations and cluster events
kubectl describe ingress app-ingress

# Fetch operational logs from the NGINX Ingress Controller
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller
```