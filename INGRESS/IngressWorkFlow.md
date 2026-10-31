# Kubernetes Ingress

Think of an Ingress as a smart traffic manager for your Kubernetes cluster. It sits at the edge of your cluster and controls how requests from the outside world are routed to your internal services.

---

## Why Do We Need It?

Normally, services inside a cluster can be exposed using NodePort or LoadBalancer, but those methods have drawbacks:

* They can be limited in advanced routing features.
* They are often cloud-dependent, making them harder to manage uniformly across different environments.

---

## How It Works

An Ingress acts like a routing map for your cluster. It tells Kubernetes:

> "Whenever someone visits this specific URL path or domain name, send them to this exact internal service."

```
User → Ingress Controller → Ingress Rules → Service → Pods
```

---

## Key Features of Ingress

* **Path-Based Routing:** Easily route traffic based on URL paths (e.g., sending `/shop` to your shopping service and `/blog` to your blog service).
* **Host-Based Routing:** Direct traffic depending on the incoming domain name.
* **TLS / SSL Support:** Terminate secure HTTPS connections efficiently at the cluster edge.

---

## Ingress vs NodePort vs LoadBalancer

| Feature | NodePort | LoadBalancer | Ingress |
|---|---|---|---|
| External access | Yes | Yes | Yes |
| Route multiple services | No | No | Yes |
| Path / host routing | No | No | Yes |
| HTTPS / TLS | No | No | Yes |
| Cost at scale | High | Very high | Low |

---

## Example 1 — Path-Based Routing

Route `/shop` and `/blog` to different services on the same domain.

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

## Example 2 — Host-Based Routing

Direct traffic depending on the incoming domain name.

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

## Example 3 — TLS / HTTPS

Terminate secure HTTPS connections efficiently at the cluster edge.

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

## Useful Commands

```bash
# List ingress resources
kubectl get ingress

# Inspect routing rules and events
kubectl describe ingress app-ingress

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller
```