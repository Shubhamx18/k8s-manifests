# Kubernetes Ingress Hands-On

## Use case

Two applications run in the same Kubernetes cluster (kind):

- nginx, which shows the default nginx welcome page
- portfolio, a personal portfolio website (image shubhamm18/portfolio:v1)

Instead of giving each application its own port, one Ingress is used as a single entry point. The URL path decides which application answers:

- /nginx is served by nginx-service
- /port is served by port-service

---

## Overall flow diagram

```text
Browser
http://localhost:18080/nginx   or   http://localhost:18080/port
                    |
                    v
kubectl port-forward  (local port 18080 to controller port 80)
                    |
                    v
+-------------------- kind cluster --------------------------------+
|                                                                  |
|     +-------------------------------------+                      |
|     |      NGINX Ingress Controller       |                      |
|     |      namespace: ingress-nginx       |                      |
|     |                                     |                      |
|     |  rules from ingress.yml:            |                      |
|     |    /nginx  to  nginx-service        |                      |
|     |    /port   to  port-service         |                      |
|     |    rewrite-target: /                |                      |
|     +-------------------------------------+                      |
|            |                        |                            |
|     path /nginx, sent as /   path /port, sent as /               |
|            |                        |                            |
|            v                        v                            |
|     +---------------+        +---------------+                   |
|     | nginx-service |        | port-service  |                   |
|     | ClusterIP :80 |        | ClusterIP :80 |                   |
|     +---------------+        +---------------+                   |
|            |                        |                            |
|            v                        v                            |
|     +---------------+        +---------------+                   |
|     |   nginx pod   |        | portfolio pod |                   |
|     | app: nginx    |        | app: port     |                   |
|     +---------------+        +---------------+                   |
|                                                                  |
+------------------------------------------------------------------+
```

Read the diagram from top to bottom. The browser reaches the controller through the port-forward. The controller follows the rules from ingress.yml, rewrites the path to /, and passes the request to the matching Service, which selects the pod by its label.

---

## app.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
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
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

Explanation:

- The Deployment starts a pod running nginx and gives it the label app: nginx.
- The Service finds that pod through its selector (app: nginx). The label and the selector must match exactly.
- type: ClusterIP means the Service is reachable only inside the cluster. Users cannot open it directly, so every request must pass through the Ingress.
- port: 80 is the port of the Service. targetPort: 80 is the port the container listens on.

## my.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: portfolio
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
        - name: portfolio
          image: shubhamm18/portfolio:v1
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: port-service
spec:
  type: ClusterIP
  selector:
    app: port
  ports:
    - port: 80
      targetPort: 80
```

The portfolio application works the same way. Its pod has the label app: port, and port-service selects that label.

---

## ingress.yml

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

### Line by line

apiVersion: networking.k8s.io/v1
The Kubernetes API that defines the Ingress object.

kind: Ingress
The type of object being created. An Ingress is a set of routing rules for web traffic coming into the cluster.

metadata.name: app-ingress
The name of this Ingress, used in commands such as kubectl describe ingress app-ingress.

annotations
Extra settings for the controller. This is explained in detail in the next section.

spec.ingressClassName: nginx
Selects which Ingress Controller must follow these rules. Here it is the NGINX Ingress Controller.

spec.rules
The list of routing rules. No host name is given, so the rules apply to any address used to reach the controller.

http.paths
The list of URL paths and where each one goes.

path: /nginx and path: /port
The URL path to match.

pathType: Prefix
Match the path and everything below it. For /port this matches /port and /port/anything. The value is case sensitive, so Prefix must start with a capital P.

backend.service.name
The Service that receives the traffic. It must match the Service name exactly: nginx-service or port-service.

backend.service.port.number: 80
The port of that Service. This is the Service port, not a NodePort.

---

## The annotation

```yaml
nginx.ingress.kubernetes.io/rewrite-target: /
```

The annotation has three parts:

- nginx.ingress.kubernetes.io is the prefix. It shows that the setting belongs to the NGINX Ingress Controller. A controller only reads annotations with its own prefix.
- rewrite-target is the name of the setting. It is a feature built into the NGINX Ingress Controller.
- The value / is the path that is sent to the application.

Why it is needed:

The applications serve their home page at /, not at /nginx or /port. The path /nginx is only used to choose the right application. With the rewrite:

- A request for /nginx reaches the nginx pod as /
- A request for /port reaches the portfolio pod as /

Without the rewrite, the pod would receive /nginx and return 404 because that page does not exist inside the application.

Kubernetes does not check annotation names. If the name is misspelled, nothing fails and the setting is silently ignored.

---

## Steps performed

### 1. Create the applications

```bash
kubectl apply -f app.yml
kubectl apply -f my.yml
kubectl get pods
kubectl get endpoints nginx-service port-service
```

Each Service must list a pod IP under endpoints. This confirms that the Service selector matches the pod label.

### 2. Create the Ingress

```bash
kubectl apply -f ingress.yml
kubectl get ingress
kubectl describe ingress app-ingress
```

The describe output shows both rules connected to pod IPs:

```
/nginx   nginx-service:80 (10.244.3.3:80)
/port    port-service:80 (10.244.2.4:80)
```

### 3. Install the Ingress Controller

An Ingress only contains rules. A controller must be running in the cluster to apply them. Checking showed that no controller was installed:

```bash
kubectl get pods -n ingress-nginx
```

Result: No resources found. This is also why the ADDRESS column was empty and Events showed none.

The NGINX Ingress Controller for kind was installed:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
kubectl get pods -n ingress-nginx
```

The controller pod must show Running with 1/1 ready.

### 4. Open the applications

The controller runs inside the cluster, so a port-forward is used to reach it from the machine:

```bash
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 18080:80
```

The format is LOCAL_PORT:SERVICE_PORT. Port 18080 was used because 8080 and 8081 were already in use on the machine. This terminal must stay open.

Test from a second terminal:

```bash
curl http://localhost:18080/nginx
curl http://localhost:18080/port
```

Open in the browser:

```
http://localhost:18080/nginx
http://localhost:18080/port
```

When the cluster runs on a remote server, localhost in the browser means the local computer. An SSH tunnel connects the two:

```bash
ssh -L 18080:localhost:18080 root@<server-ip>
```

---

## How a request flows

1. The browser requests /port.
2. The request reaches the NGINX Ingress Controller.
3. The controller finds the rule for /port, which points to port-service.
4. The rewrite changes the path to /.
5. The request is delivered to a portfolio pod.
6. The response returns the same way to the browser.

---

## Result

- One entry point serves both applications.
- /nginx opens the nginx welcome page.
- /port opens the portfolio application.
- Both Services are ClusterIP, so the Ingress is the only way in.

Note: because rewrite-target sends every request to /, an application that loads CSS or JavaScript from absolute paths such as /static/app.js may look unstyled when served under /port. The nginx welcome page has no such files, so it is not affected.
