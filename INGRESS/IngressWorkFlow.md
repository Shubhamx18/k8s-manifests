# Kubernetes Ingress: Understand It and Build It from Scratch

## 1. What Ingress Is

An **Ingress** is a set of routing rules that sends external HTTP/HTTPS traffic to Services inside the cluster, based on URL path or hostname. It gives you **one entry point for many Services**, instead of one NodePort or LoadBalancer per app.

An Ingress only holds rules. An **Ingress Controller** (here ingress-nginx) is the program that reads those rules and does the routing. Without a controller, an Ingress does nothing.

```
User ──> Ingress Controller (NGINX pod) ──> Service ──> Pods
                 ▲
                 │ reads the rules from
          Ingress resource (ingress.yml)
```

| | NodePort | LoadBalancer | Ingress |
|---|---|---|---|
| One entry point for many Services | No | No | **Yes** |
| Path / host routing | No | No | **Yes** |
| TLS termination | No | No | **Yes** |

---

## 2. Rules to Remember

- The Ingress must be in the **same namespace** as its Services, or you get 503 errors.
- `ingressClassName: nginx` must match the installed controller.
- `pathType: Prefix` matches the path and everything under it.
- `rewrite-target: /` removes the matched prefix, so `/nginx` reaches the app as `/`.
- A Service with no running Pods has no endpoints, and the Ingress returns 503.

---

## 3. Build It from Scratch

### Step 1: Create a kind cluster

Optional: map ports 80 and 443 so you can use `http://localhost` without port-forward. If your cluster already exists without this, skip it and use port-forward in Step 6.

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
```

```bash
kind create cluster --config kind-config.yaml
```

### Step 2: Install the Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait -n ingress-nginx --for=condition=ready pod \
  -l app.kubernetes.io/component=controller --timeout=120s
```

### Step 3: Create the namespace and the apps

```bash
kubectl create namespace state
```

```yaml
# app.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  namespace: state
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx-container
          image: nginx:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: state
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

```yaml
# my.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: port
  namespace: state
spec:
  selector:
    matchLabels:
      app: port
  template:
    metadata:
      labels:
        app: port
    spec:
      containers:
        - name: port-app
          image: shubhamm18/portfolio:v1
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: port-service
  namespace: state
spec:
  selector:
    app: port
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f app.yml -f my.yml
kubectl get pods,svc,endpoints -n state     # endpoints must show pod IPs
```

### Step 4: Create the Ingress

```yaml
# ingress.yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: state
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /nginx
            pathType: Prefix
            backend:
              service:
                name: nginx-service
                port:
                  number: 80
          - path: /port
            pathType: Prefix
            backend:
              service:
                name: port-service
                port:
                  number: 80
```

```bash
kubectl apply -f ingress.yml
kubectl get ingress -n state
```

### Step 5: Look at what the controller created

The Ingress does not create a Service. The controller Service is created when ingress-nginx is installed, in the `ingress-nginx` namespace.

```bash
# The controller Service (the one your port-forward targets)
kubectl get svc -n ingress-nginx

# Full details: type, ports, NodePort, selector, endpoints
kubectl describe svc ingress-nginx-controller -n ingress-nginx

# The controller pod behind it
kubectl get pods -n ingress-nginx -o wide

# Confirm the Service points at that pod
kubectl get endpoints ingress-nginx-controller -n ingress-nginx
```

### Step 6: Access the apps

**If you mapped ports in Step 1:**

```bash
curl http://localhost/nginx
curl http://localhost/port
```

**Otherwise, use port-forward** (leave it running in one terminal):

```bash
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8082:80
```

```bash
# second terminal, on the same machine
curl http://localhost:8082/nginx
curl http://localhost:8082/port
```

To use a browser on another machine, `localhost` points at that machine, not the server. Use an SSH tunnel from your laptop, then browse to `http://localhost:8082/nginx`:

```bash
ssh -L 8082:localhost:8082 user@<server-ip>
```

Or run the forward with `--address 0.0.0.0` and browse to `http://<server-ip>:8082/nginx`.

---

## 4. Troubleshooting

```bash
kubectl describe ingress app-ingress -n state      # backends, pod IPs, events
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller
kubectl get endpoints -n state
```

| Symptom | Likely cause |
|---|---|
| Connection refused | Port-forward not running, wrong port, or using the wrong machine's `localhost` |
| 404 from nginx | No rule matches the path |
| 503 | No endpoints, or Ingress and Service in different namespaces |
| Page loads but CSS/JS broken | App uses absolute asset paths (`/style.css`) behind a path prefix |

---

## 5. Other Routing Types (Reference)

**Host-based:** add `host: shop.example.com` under a rule instead of routing by path.
**TLS:** add a `tls:` block with `hosts` and `secretName`, where the secret is created with `kubectl create secret tls <name> --cert=tls.crt --key=tls.key`.
