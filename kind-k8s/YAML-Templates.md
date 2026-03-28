# Kubernetes YAML Template Sheet


---

## Core Workload Resources

### 1. Pod

The smallest deployable unit in Kubernetes. It can hold one or more containers.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: my-container
      image: nginx:latest
      ports:
        - containerPort: 80
```

**Use Case:** Minimal example of a running pod with a single NGINX container on port 80.

---

### 2. ReplicaSet

Ensures a specified number of identical Pods are running at all times.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-replicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: nginx
          image: nginx
```

**Use Case:** Maintains 3 identical pod replicas for high availability.

---

### 3. Deployment

Provides declarative updates for Pods and ReplicaSets.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-container
          image: nginx:latest
          ports:
            - containerPort: 80
```

**Features:**
- Creates and manages 3 replicas of your pod
- Automatically replaces unhealthy pods for high availability
- Supports rolling updates and rollbacks

---

### 4. StatefulSet

Used for stateful applications that require stable, unique network identities and persistent storage.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-statefulset
spec:
  selector:
    matchLabels:
      app: stateful
  serviceName: "stateful-service"
  replicas: 2
  template:
    metadata:
      labels:
        app: stateful
    spec:
      containers:
        - name: stateful-container
          image: mysql
```

**Use Case:** Databases, message queues, and applications requiring stable network identities.

---

### 5. DaemonSet

Ensures that a pod runs on every node in the cluster.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: my-daemonset
spec:
  selector:
    matchLabels:
      app: daemon
  template:
    metadata:
      labels:
        app: daemon
    spec:
      containers:
        - name: daemon-container
          image: busybox
          command: ["sleep", "3600"]
```

**Use Case:** Log collectors, monitoring agents, node-level services.

---

## Service & Networking

### 6. Service (ClusterIP)

Exposes your app on an internal IP address in the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

**Use Case:** Internal service that allows stable access to pods within the cluster.

---

### 7. Service (NodePort)

Exposes your app on a static port on each Node's IP.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30009
```

**Use Case:** External access to services without a load balancer.

---

### 8. Service (LoadBalancer)

Exposes your app to the internet via a cloud provider's load balancer.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-loadbalancer
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```

**Use Case:** Production-grade external access with cloud provider integration.

---

### 9. Ingress

Manages external access to services, typically HTTP/HTTPS.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80
```

**Use Case:** Routes external HTTP traffic to internal services using domain names. Often used with Ingress controllers (nginx, traefik).

---

### 10. NetworkPolicy

Controls how groups of Pods communicate with each other and external endpoints.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-http
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
    - Ingress
  ingress:
    - from:
        - ipBlock:
            cidr: 192.168.1.0/24
      ports:
        - protocol: TCP
          port: 80
```

**Use Case:** Restricts network access to specific IP ranges. Crucial for securing internal communication.

---

### 11. Endpoints (Manual)

Defines manually specified network endpoints for a Service.

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: my-endpoints
subsets:
  - addresses:
      - ip: 10.10.1.1
    ports:
      - port: 80
```

**Use Case:** Custom service endpoints for external databases or legacy systems.

---

## Configuration & Storage

### 12. ConfigMap

Stores non-sensitive configuration data to be used by applications.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  key1: "value1"
  key2: "value2"
```

**Use Case:** Environment variables, configuration files mounted into pods.

---

### 13. Secret

Stores sensitive information such as passwords, tokens, or keys.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  username: dXNlcg==       # base64 encoded "user"
  password: cGFzc3dvcmQ=   # base64 encoded "password"
```

**Use Case:** Secure storage of credentials, API keys, certificates.

---

### 14. PersistentVolume (PV)

A piece of storage in the cluster provisioned by an admin.

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
    path: "/mnt/data"
```

**Use Case:** Defines persistent storage for data persistence across pod restarts.

---

### 15. PersistentVolumeClaim (PVC)

Requests storage from an available PersistentVolume.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

**Use Case:** Pods use PVCs to mount persistent storage.

---

### 16. Volume (EmptyDir)

Temporary volume that exists as long as the Pod exists.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod
spec:
  containers:
    - name: busybox
      image: busybox
      command: ["sleep", "3600"]
      volumeMounts:
        - name: temp-storage
          mountPath: /data
  volumes:
    - name: temp-storage
      emptyDir: {}
```

**Use Case:** Temporary storage for cache, scratch space, or sharing data between containers in a pod.

---

## Scheduling & Scaling

### 17. Job

Runs a one-time task or batch job that executes and exits.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: my-job
spec:
  template:
    spec:
      containers:
        - name: job-container
          image: busybox
          command: ["echo", "Hello Kubernetes"]
      restartPolicy: Never
```

**Use Case:** Batch processing, database migrations, cleanup tasks.

---

### 18. CronJob

Runs a scheduled task on a recurring schedule.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: my-cronjob
spec:
  schedule: "*/5 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: cron-container
              image: busybox
              command: ["echo", "Scheduled Task"]
          restartPolicy: OnFailure
```

**Use Case:** Scheduled backups, reports, cleanup tasks. Similar to Unix cron.

---

### 19. HorizontalPodAutoscaler (HPA)

Automatically scales the number of pods based on CPU/memory usage.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-deployment
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

**Use Case:** Automatically scales pods between 2-5 replicas based on 50% CPU utilization.

---

### 20. PodDisruptionBudget (PDB)

Ensures a minimum number of pods stay available during voluntary disruptions.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: my-app
```

**Use Case:** Maintains availability during node upgrades, cluster maintenance.

---

## Security & Access Control

### 21. Namespace

Provides a mechanism for isolating groups of resources within a cluster.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-namespace
```

**Use Case:** Multi-tenancy, environment separation (dev, staging, prod).

---

### 22. ServiceAccount

Provides an identity for processes running in a Pod to interact with the Kubernetes API.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
```

**Use Case:** Pod authentication to Kubernetes API or external services.

---

### 23. Role

Defines permissions within a namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: my-namespace
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
```

**Use Case:** Namespace-scoped permissions for reading pods.

---

### 24. RoleBinding

Grants a Role to a user or service account within a namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: my-namespace
subjects:
  - kind: ServiceAccount
    name: my-service-account
    namespace: my-namespace
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Use Case:** Binds the pod-reader role to a service account.

---

### 25. ClusterRole

Defines permissions across the entire cluster.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-admin-role
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
```

**Use Case:** Cluster-wide permissions for admin operations.

---

### 26. ClusterRoleBinding

Binds a ClusterRole to a user or group at the cluster level.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-binding
subjects:
  - kind: User
    name: admin
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin-role
  apiGroup: rbac.authorization.k8s.io
```

**Use Case:** Grants cluster-wide admin access to a user.

---

## Resource Management

### 27. ResourceQuota

Manages resource usage per namespace.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: my-quota
  namespace: my-namespace
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "10"
    limits.memory: 16Gi
```

**Use Case:** Prevents resource exhaustion by limiting namespace resource consumption.

---

### 28. LimitRange

Sets default resource requests and limits for pods and containers in a namespace.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: my-namespace
spec:
  limits:
    - default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 200m
        memory: 256Mi
      type: Container
```

**Use Case:** Enforces default resource constraints for all containers in a namespace.

---

## Advanced Pod Configuration

### 29. Environment Variables

Multiple ways to inject environment variables into containers.

#### Direct Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-pod
spec:
  containers:
    - name: my-container
      image: nginx
      env:
        - name: ENVIRONMENT
          value: "production"
        - name: LOG_LEVEL
          value: "info"
        - name: MAX_CONNECTIONS
          value: "100"
```

#### Environment Variables from ConfigMap

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-env-pod
spec:
  containers:
    - name: my-container
      image: nginx
      env:
        # Single value from ConfigMap
        - name: CONFIG_KEY
          valueFrom:
            configMapKeyRef:
              name: my-config
              key: key1
      # Load all ConfigMap keys as environment variables
      envFrom:
        - configMapRef:
            name: my-config
```

#### Environment Variables from Secret

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
    - name: my-container
      image: nginx
      env:
        # Single value from Secret
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: my-secret
              key: password
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: my-secret
              key: username
      # Load all Secret keys as environment variables
      envFrom:
        - secretRef:
            name: my-secret
```

#### Environment Variables from Field References

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: field-ref-pod
spec:
  containers:
    - name: my-container
      image: nginx
      env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
```

---

### 30. Volume Mounts (ConfigMap & Secret)

#### Mount ConfigMap as Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-volume-pod
spec:
  containers:
    - name: my-container
      image: nginx
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
          readOnly: true
  volumes:
    - name: config-volume
      configMap:
        name: my-config
        items:
          - key: key1
            path: config-key1.txt
          - key: key2
            path: config-key2.txt
```

#### Mount Secret as Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-pod
spec:
  containers:
    - name: my-container
      image: nginx
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: secret-volume
      secret:
        secretName: my-secret
        items:
          - key: username
            path: db-username
          - key: password
            path: db-password
```

---

### 31. Resource Requests and Limits

Define CPU and memory requirements for containers.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
    - name: my-container
      image: nginx
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
```

**Explanation:**
- **Requests:** Minimum resources guaranteed to the container
- **Limits:** Maximum resources the container can use

---

### 32. Health Checks (Probes)

#### Liveness Probe

Checks if the container is running. Restarts the container if it fails.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-pod
spec:
  containers:
    - name: my-container
      image: nginx
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 3
        periodSeconds: 10
        timeoutSeconds: 5
        failureThreshold: 3
```

#### Readiness Probe

Checks if the container is ready to serve traffic.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-pod
spec:
  containers:
    - name: my-container
      image: nginx
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 5
        timeoutSeconds: 3
        successThreshold: 1
        failureThreshold: 3
```

#### Startup Probe

Checks if the application has started. Useful for slow-starting containers.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-pod
spec:
  containers:
    - name: my-container
      image: nginx
      startupProbe:
        httpGet:
          path: /startup
          port: 8080
        initialDelaySeconds: 0
        periodSeconds: 10
        timeoutSeconds: 3
        failureThreshold: 30
```

#### Probe Types

**HTTP GET Probe:**
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
    httpHeaders:
      - name: Custom-Header
        value: "CustomValue"
```

**TCP Socket Probe:**
```yaml
livenessProbe:
  tcpSocket:
    port: 3306
  initialDelaySeconds: 15
  periodSeconds: 20
```

**Exec Command Probe:**
```yaml
livenessProbe:
  exec:
    command:
      - cat
      - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

---

### 33. Init Containers

Runs before the main container starts. Useful for setup tasks.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-container-pod
spec:
  initContainers:
    - name: init-service
      image: busybox
      command: ['sh', '-c', 'until nslookup my-service; do echo waiting for service; sleep 2; done']
    - name: init-db
      image: busybox
      command: ['sh', '-c', 'until nc -z db-service 3306; do echo waiting for db; sleep 2; done']
  containers:
    - name: main-app
      image: nginx
      ports:
        - containerPort: 80
```

**Use Case:** Wait for dependencies, database migrations, config initialization.

---

### 34. Multi-Container Pod Patterns

#### Sidecar Pattern

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-pod
spec:
  containers:
    # Main application container
    - name: app
      image: nginx
      ports:
        - containerPort: 80
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/nginx
    
    # Sidecar container for log shipping
    - name: log-shipper
      image: fluent/fluentd
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/nginx
  
  volumes:
    - name: shared-logs
      emptyDir: {}
```

#### Ambassador Pattern

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ambassador-pod
spec:
  containers:
    # Main application
    - name: app
      image: myapp
      ports:
        - containerPort: 8080
    
    # Ambassador proxy
    - name: ambassador
      image: envoyproxy/envoy
      ports:
        - containerPort: 9901
```

#### Adapter Pattern

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: adapter-pod
spec:
  containers:
    # Main application with non-standard metrics
    - name: app
      image: myapp
      ports:
        - containerPort: 8080
    
    # Adapter to convert metrics to Prometheus format
    - name: metrics-adapter
      image: prometheus-adapter
      ports:
        - containerPort: 9090
```

---

### 35. Node Affinity and Pod Affinity

#### Node Affinity

Schedule pods on specific nodes based on labels.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-affinity-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 1
          preference:
            matchExpressions:
              - key: zone
                operator: In
                values:
                  - us-west-1a
  containers:
    - name: nginx
      image: nginx
```

#### Pod Affinity

Schedule pods together or apart.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-affinity-pod
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - cache
          topologyKey: kubernetes.io/hostname
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchExpressions:
                - key: app
                  operator: In
                  values:
                    - database
            topologyKey: kubernetes.io/hostname
  containers:
    - name: nginx
      image: nginx
```

---

### 36. Taints and Tolerations

#### Taint a Node (kubectl command)

```bash
kubectl taint nodes node1 key=value:NoSchedule
kubectl taint nodes node1 key=value:NoExecute
kubectl taint nodes node1 key=value:PreferNoSchedule
```

#### Pod with Tolerations

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: toleration-pod
spec:
  tolerations:
    - key: "key"
      operator: "Equal"
      value: "value"
      effect: "NoSchedule"
    - key: "node-role.kubernetes.io/master"
      operator: "Exists"
      effect: "NoSchedule"
  containers:
    - name: nginx
      image: nginx
```

---

### 37. Security Context

Define security settings for pods and containers.

#### Pod-level Security Context

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    fsGroupChangePolicy: "OnRootMismatch"
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: nginx
      image: nginx
      securityContext:
        allowPrivilegeEscalation: false
        runAsNonRoot: true
        capabilities:
          drop:
            - ALL
          add:
            - NET_BIND_SERVICE
        readOnlyRootFilesystem: true
```

---

### 38. Horizontal Pod Autoscaler with Custom Metrics

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: custom-metrics-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
```

---

### 39. Complete Deployment with All Features

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: full-featured-deployment
  namespace: production
  labels:
    app: myapp
    version: v1
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      serviceAccountName: my-service-account
      securityContext:
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
      
      initContainers:
        - name: init-db
          image: busybox
          command: ['sh', '-c', 'until nc -z postgres 5432; do sleep 2; done']
      
      containers:
        - name: app
          image: myapp:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          
          env:
            - name: ENVIRONMENT
              value: "production"
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: db_host
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: password
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
          
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secret
          
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 3
            successThreshold: 1
            failureThreshold: 3
          
          startupProbe:
            httpGet:
              path: /startup
              port: 8080
            initialDelaySeconds: 0
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 30
          
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
              readOnly: true
            - name: secret-volume
              mountPath: /etc/secrets
              readOnly: true
            - name: data-volume
              mountPath: /data
            - name: cache-volume
              mountPath: /cache
          
          securityContext:
            allowPrivilegeEscalation: false
            runAsNonRoot: true
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
      
      volumes:
        - name: config-volume
          configMap:
            name: app-config
        - name: secret-volume
          secret:
            secretName: app-secret
        - name: data-volume
          persistentVolumeClaim:
            claimName: app-pvc
        - name: cache-volume
          emptyDir: {}
      
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - myapp
                topologyKey: kubernetes.io/hostname
      
      tolerations:
        - key: "node.kubernetes.io/not-ready"
          operator: "Exists"
          effect: "NoExecute"
          tolerationSeconds: 300
```

---

### 40. Lifecycle Hooks

PostStart and PreStop hooks for container lifecycle management.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-pod
spec:
  containers:
    - name: lifecycle-container
      image: nginx
      lifecycle:
        postStart:
          exec:
            command: ["/bin/sh", "-c", "echo 'Container started' > /tmp/started"]
        preStop:
          exec:
            command: ["/bin/sh", "-c", "nginx -s quit; while killall -0 nginx; do sleep 1; done"]
      ports:
        - containerPort: 80
```

**Use Cases:**
- **postStart:** Initialize application, register with service discovery
- **preStop:** Graceful shutdown, deregister from load balancer, flush logs

---

### 41. Command and Arguments

Override container's default command and arguments.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: command-pod
spec:
  containers:
    - name: my-container
      image: busybox
      # Override ENTRYPOINT
      command: ["/bin/sh"]
      # Override CMD
      args: ["-c", "while true; do echo Hello Kubernetes; sleep 10; done"]
```

**Comparison with Dockerfile:**
- `command` → overrides `ENTRYPOINT`
- `args` → overrides `CMD`

---

### 42. ImagePullSecrets

Pull images from private registries.

#### Create Docker Registry Secret

```bash
kubectl create secret docker-registry my-registry-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=myusername \
  --docker-password=mypassword \
  --docker-email=myemail@example.com
```

#### Use in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-image-pod
spec:
  containers:
    - name: my-container
      image: myregistry.com/myapp:1.0.0
  imagePullSecrets:
    - name: my-registry-secret
```

---

### 43. hostPath Volume

Mount a file or directory from the host node's filesystem.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
    - name: test-container
      image: nginx
      volumeMounts:
        - name: host-volume
          mountPath: /data
  volumes:
    - name: host-volume
      hostPath:
        path: /var/data
        type: DirectoryOrCreate
```

**hostPath Types:**
- `DirectoryOrCreate` - Create directory if doesn't exist
- `Directory` - Must be an existing directory
- `FileOrCreate` - Create file if doesn't exist
- `File` - Must be an existing file
- `Socket` - Must be an existing UNIX socket
- `CharDevice` - Must be a character device
- `BlockDevice` - Must be a block device

---

### 44. Downward API

Expose pod and container fields to running containers.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-api-pod
  labels:
    app: myapp
    tier: frontend
  annotations:
    build: "v1.2.3"
spec:
  containers:
    - name: my-container
      image: busybox
      command: ["sh", "-c"]
      args:
        - while true; do
            echo -en '\n';
            cat /etc/podinfo/labels;
            cat /etc/podinfo/annotations;
            sleep 3600;
          done
      env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: SERVICE_ACCOUNT
          valueFrom:
            fieldRef:
              fieldPath: spec.serviceAccountName
        - name: CPU_REQUEST
          valueFrom:
            resourceFieldRef:
              containerName: my-container
              resource: requests.cpu
        - name: MEMORY_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: my-container
              resource: limits.memory
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
  volumes:
    - name: podinfo
      downwardAPI:
        items:
          - path: "labels"
            fieldRef:
              fieldPath: metadata.labels
          - path: "annotations"
            fieldRef:
              fieldPath: metadata.annotations
```

---

### 45. Priority and Preemption

Set pod priority for scheduling decisions.

#### Create PriorityClass

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "High priority class for critical applications"
```

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 1000
globalDefault: false
description: "Low priority class for batch jobs"
```

#### Use PriorityClass in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: priority-pod
spec:
  priorityClassName: high-priority
  containers:
    - name: nginx
      image: nginx
```

---

### 46. DNS Configuration

Customize DNS settings for pods.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dns-config-pod
spec:
  containers:
    - name: test
      image: nginx
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 8.8.8.8
      - 8.8.4.4
    searches:
      - my.dns.search.suffix
      - example.com
    options:
      - name: ndots
        value: "2"
      - name: edns0
```

**DNS Policies:**
- `ClusterFirst` - Default, uses cluster DNS
- `Default` - Uses node's DNS
- `ClusterFirstWithHostNet` - For pods using hostNetwork
- `None` - Custom DNS config

---

### 47. Service with Session Affinity

Maintain client session to the same pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: session-affinity-service
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
```

---

### 48. Headless Service

Service without cluster IP for direct pod-to-pod communication.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-service
spec:
  clusterIP: None
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

**Use Case:** StatefulSets, service discovery, direct pod DNS access.

---

### 49. ExternalName Service

Map a service to an external DNS name.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  type: ExternalName
  externalName: api.external-service.com
  ports:
    - port: 443
```

---

### 50. Topology Spread Constraints

Distribute pods evenly across zones/nodes.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: topology-spread-deployment
spec:
  replicas: 6
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: myapp
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: myapp
      containers:
        - name: myapp
          image: nginx
```

**Use Case:** High availability across availability zones and nodes.

---

## Environment Variables Reference Table

| Source Type | Example | Use Case |
|-------------|---------|----------|
| **Direct Value** | `value: "production"` | Static configuration |
| **ConfigMap** | `configMapKeyRef` | Non-sensitive config |
| **Secret** | `secretKeyRef` | Passwords, tokens, keys |
| **Field Reference** | `fieldRef: metadata.name` | Pod metadata |
| **Resource Field** | `resourceFieldRef: requests.cpu` | Resource info |

---

## Probe Configuration Reference

| Parameter | Description | Default |
|-----------|-------------|---------|
| `initialDelaySeconds` | Delay before first probe | 0 |
| `periodSeconds` | How often to perform probe | 10 |
| `timeoutSeconds` | Timeout for probe | 1 |
| `successThreshold` | Min consecutive successes | 1 |
| `failureThreshold` | Min consecutive failures | 3 |

---

## Volume Types Quick Reference

| Volume Type | Description | Use Case |
|-------------|-------------|----------|
| `emptyDir` | Temporary storage | Cache, scratch space |
| `hostPath` | Host node's filesystem | Node-level data access |
| `configMap` | Configuration data | App config files |
| `secret` | Sensitive data | Credentials, certificates |
| `persistentVolumeClaim` | Persistent storage | Database storage |
| `nfs` | Network File System | Shared storage |
| `awsElasticBlockStore` | AWS EBS | Cloud persistent storage |
| `gcePersistentDisk` | GCE PD | Cloud persistent storage |
| `azureDisk` | Azure Disk | Cloud persistent storage |

---

## Common Environment Variable Patterns

### Database Connection

```yaml
env:
  - name: DB_HOST
    valueFrom:
      configMapKeyRef:
        name: db-config
        key: host
  - name: DB_PORT
    valueFrom:
      configMapKeyRef:
        name: db-config
        key: port
  - name: DB_NAME
    valueFrom:
      configMapKeyRef:
        name: db-config
        key: database
  - name: DB_USER
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: username
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password
```

### API Configuration

```yaml
env:
  - name: API_KEY
    valueFrom:
      secretKeyRef:
        name: api-secret
        key: key
  - name: API_ENDPOINT
    valueFrom:
      configMapKeyRef:
        name: api-config
        key: endpoint
  - name: API_TIMEOUT
    value: "30"
  - name: MAX_RETRIES
    value: "3"
```

---

## Quick Reference

### Common kubectl Commands

```bash
# Get resources
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get all

# Describe resources
kubectl describe pod <pod-name>
kubectl describe deployment <deployment-name>

# Create resources
kubectl apply -f <file.yaml>
kubectl create -f <file.yaml>

# Delete resources
kubectl delete pod <pod-name>
kubectl delete -f <file.yaml>

# Logs and debugging
kubectl logs <pod-name>
kubectl logs -f <pod-name>  # Follow logs
kubectl exec -it <pod-name> -- /bin/bash

# Scale deployments
kubectl scale deployment <name> --replicas=5

# Update deployments
kubectl set image deployment/<name> <container>=<new-image>
kubectl rollout status deployment/<name>
kubectl rollout undo deployment/<name>
```

---

## Access Modes

| Mode | Description |
|------|-------------|
| `ReadWriteOnce` (RWO) | Volume can be mounted read-write by a single node |
| `ReadOnlyMany` (ROX) | Volume can be mounted read-only by many nodes |
| `ReadWriteMany` (RWX) | Volume can be mounted read-write by many nodes |

---

## Resource Units

### CPU
- `1000m` = 1 CPU core
- `500m` = 0.5 CPU core
- `100m` = 0.1 CPU core

### Memory
- `1Gi` = 1 Gibibyte
- `512Mi` = 512 Mebibytes
- `1G` = 1 Gigabyte (1000 MB)

---

## Best Practices

1. **Always use namespaces** for resource isolation
2. **Set resource requests and limits** to prevent resource exhaustion
3. **Use labels and selectors** for better organization
4. **Implement health checks** (liveness and readiness probes)
5. **Use ConfigMaps and Secrets** instead of hardcoding configuration
6. **Enable RBAC** for security and access control
7. **Use PodDisruptionBudgets** for high availability
8. **Implement NetworkPolicies** for network security
9. **Use Deployments** instead of bare Pods
10. **Version control your YAML files**

---

## Additional Resources

- [Official Kubernetes Documentation](https://kubernetes.io/docs/)
- [Kubernetes API Reference](https://kubernetes.io/docs/reference/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

---

**Last Updated:** February 2026  
**Version:** Kubernetes 1.28+

---

## Extended kubectl Command Reference

### Advanced Get Commands

```bash
# Get with custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName

# Get with JSONPath
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Watch resources
kubectl get pods -w

# Get with labels shown
kubectl get pods --show-labels

# Get sorted
kubectl get pods --sort-by=.metadata.creationTimestamp
```

### Debugging Commands

```bash
# View environment variables in running pod
kubectl exec <pod-name> -- env

# View mounted ConfigMap
kubectl exec <pod-name> -- cat /etc/config/key1

# Check DNS resolution
kubectl exec <pod-name> -- nslookup kubernetes.default

# Test connectivity
kubectl exec <pod-name> -- curl http://service-name:port

# Run temporary debug pod
kubectl run debug --rm -it --image=busybox -- sh
kubectl run debug --rm -it --image=nicolaka/netshoot -- bash

# Previous container logs (for crashed containers)
kubectl logs <pod-name> --previous

# All containers in a pod
kubectl logs <pod-name> --all-containers=true
```

### Resource Management

```bash
# Apply with dry-run
kubectl apply -f deployment.yaml --dry-run=client

# Diff before apply
kubectl diff -f deployment.yaml

# Force delete pod
kubectl delete pod <pod-name> --grace-period=0 --force

# Delete all pods in namespace
kubectl delete pods --all -n <namespace>

# Patch resource
kubectl patch deployment <name> -p '{"spec":{"replicas":5}}'
```

---

## Complete Environment Variables Example

### Multi-Environment Application

**ConfigMap for Development:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-dev
  namespace: development
data:
  APP_ENV: "development"
  LOG_LEVEL: "debug"
  DB_HOST: "postgres-dev.default.svc.cluster.local"
  DB_PORT: "5432"
  DB_NAME: "app_dev"
  REDIS_HOST: "redis-dev.default.svc.cluster.local"
  REDIS_PORT: "6379"
  API_ENDPOINT: "https://api-dev.example.com"
  FEATURE_FLAG_NEW_UI: "true"
  MAX_UPLOAD_SIZE: "10485760"
```

**ConfigMap for Production:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-prod
  namespace: production
data:
  APP_ENV: "production"
  LOG_LEVEL: "error"
  DB_HOST: "postgres-prod.default.svc.cluster.local"
  DB_PORT: "5432"
  DB_NAME: "app_prod"
  REDIS_HOST: "redis-prod.default.svc.cluster.local"
  REDIS_PORT: "6379"
  API_ENDPOINT: "https://api.example.com"
  FEATURE_FLAG_NEW_UI: "false"
  MAX_UPLOAD_SIZE: "52428800"
```

**Secret:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  DB_PASSWORD: cHJvZFBhc3N3MHJkITIz        # prodPassw0rd!23
  DB_USER: YXBwdXNlcg==                    # appuser
  REDIS_PASSWORD: cmVkaXNQYXNzMTIz         # redisPass123
  API_KEY: YXBpa2V5LXNlY3JldC0xMjM0NQ==    # apikey-secret-12345
  JWT_SECRET: and0LXNlY3JldC1zdXBlci1sb25nLWtleQ==  # jwt-secret-super-long-key
  ENCRYPTION_KEY: ZW5jcnlwdGlvbi1rZXktMzItYnl0ZXMtbG9uZw==  # encryption-key-32-bytes-long
```

**Deployment Using Environment Variables:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
        version: v2.0.0
    spec:
      containers:
        - name: webapp
          image: webapp:2.0.0
          ports:
            - name: http
              containerPort: 8080
          
          # Load all from ConfigMap
          envFrom:
            - configMapRef:
                name: app-config-prod
            - secretRef:
                name: app-secret
          
          # Override or add specific values
          env:
            # Application info
            - name: APP_NAME
              value: "My Web Application"
            - name: APP_VERSION
              value: "2.0.0"
            
            # Pod information
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            
            # Resource information
            - name: CPU_REQUEST
              valueFrom:
                resourceFieldRef:
                  containerName: webapp
                  resource: requests.cpu
            - name: MEMORY_REQUEST
              valueFrom:
                resourceFieldRef:
                  containerName: webapp
                  resource: requests.memory
            - name: CPU_LIMIT
              valueFrom:
                resourceFieldRef:
                  containerName: webapp
                  resource: limits.cpu
            - name: MEMORY_LIMIT
              valueFrom:
                resourceFieldRef:
                  containerName: webapp
                  resource: limits.memory
            
            # Service accounts
            - name: SERVICE_ACCOUNT
              valueFrom:
                fieldRef:
                  fieldPath: spec.serviceAccountName
          
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          
          volumeMounts:
            - name: app-config-volume
              mountPath: /etc/config
              readOnly: true
            - name: app-secret-volume
              mountPath: /etc/secrets
              readOnly: true
      
      volumes:
        - name: app-config-volume
          configMap:
            name: app-config-prod
        - name: app-secret-volume
          secret:
            secretName: app-secret
```

---

## Probe Configuration Examples

### HTTP Liveness Probe with Custom Headers

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    scheme: HTTP
    httpHeaders:
      - name: X-Custom-Header
        value: HealthCheck
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  successThreshold: 1
  failureThreshold: 3
```

### gRPC Probe (Kubernetes 1.24+)

```yaml
livenessProbe:
  grpc:
    port: 9090
  initialDelaySeconds: 10
  periodSeconds: 10
```

### Combined Startup, Liveness, and Readiness

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: complete-probes-pod
spec:
  containers:
    - name: app
      image: myapp:1.0.0
      ports:
        - containerPort: 8080
      
      # Startup probe - for slow-starting apps
      startupProbe:
        httpGet:
          path: /startup
          port: 8080
        initialDelaySeconds: 0
        periodSeconds: 10
        timeoutSeconds: 3
        failureThreshold: 30  # 5 minutes max startup time
      
      # Liveness probe - restart if unhealthy
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 10
        timeoutSeconds: 5
        failureThreshold: 3
      
      # Readiness probe - remove from service if not ready
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 5
        timeoutSeconds: 3
        successThreshold: 1
        failureThreshold: 3
```
# Kubernetes Database Setup Guide

## MySQL Setup

### Step 1: Create MySQL Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
data:
  # echo -n "root123" | base64
  root-password: cm9vdDEyMw==
  # echo -n "myuser" | base64  
  user: bXl1c2Vy
  # echo -n "pass123" | base64
  password: cGFzczEyMw==
```

### Step 2: Create MySQL Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
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
          
          env:
            # Root password - REQUIRED
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: root-password
            
            # Database name - creates this database
            - name: MYSQL_DATABASE
              value: "myapp"
            
            # User name
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: user
            
            # User password
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
```

### Step 3: Create MySQL Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
```

---

## MongoDB Setup

### Step 1: Create MongoDB Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongo-secret
type: Opaque
data:
  # echo -n "admin" | base64
  username: YWRtaW4=
  # echo -n "pass123" | base64
  password: cGFzczEyMw==
```

### Step 2: Create MongoDB Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongodb
          image: mongo:7.0
          ports:
            - containerPort: 27017
          
          env:
            # Root username - REQUIRED
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: username
            
            # Root password - REQUIRED
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: password
            
            # Database name
            - name: MONGO_INITDB_DATABASE
              value: "myapp"
```

### Step 3: Create MongoDB Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb
spec:
  selector:
    app: mongodb
  ports:
    - port: 27017
      targetPort: 27017
```

---

## How to Set Environment Variables in Deployment

### Method 1: Direct Value (Simple)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:1.0
          
          env:
            # Simple value
            - name: ENV
              value: "production"
            
            - name: PORT
              value: "8080"
            
            - name: LOG_LEVEL
              value: "info"
```

### Method 2: From ConfigMap

**ConfigMap:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_name: "myapp"
  api_url: "https://api.example.com"
```

**Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:1.0
          
          env:
            # Single value from ConfigMap
            - name: DB_NAME
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: database_name
            
            - name: API_URL
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: api_url
```

### Method 3: From Secret

**Secret:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  # echo -n "secret123" | base64
  api_key: c2VjcmV0MTIz
  # echo -n "token456" | base64
  token: dG9rZW40NTY=
```

**Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:1.0
          
          env:
            # Single value from Secret
            - name: API_KEY
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: api_key
            
            - name: TOKEN
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: token
```

### Method 4: Load All Keys from ConfigMap/Secret

**ConfigMap:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: all-config
data:
  ENV: "production"
  PORT: "8080"
  LOG_LEVEL: "info"
```

**Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:1.0
          
          # Load ALL keys from ConfigMap as env variables
          envFrom:
            - configMapRef:
                name: all-config
```

### Method 5: Mix Everything Together

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:1.0
          
          # Load all from ConfigMap and Secret
          envFrom:
            - configMapRef:
                name: all-config
            - secretRef:
                name: all-secret
          
          # Add or override specific values
          env:
            - name: APP_NAME
              value: "My Application"
            
            - name: CUSTOM_VALUE
              value: "special"
```

---

## Complete Example: App with MySQL

### 1. MySQL Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
data:
  root-password: cm9vdDEyMw==
  user: YXBwdXNlcg==
  password: YXBwcGFzczEyMw==
```

### 2. MySQL Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
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
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: root-password
            - name: MYSQL_DATABASE
              value: "myapp"
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: user
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
```

### 3. MySQL Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  selector:
    app: mysql
  ports:
    - port: 3306
```

### 4. App Deployment (Connecting to MySQL)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:1.0
          ports:
            - containerPort: 8080
          
          env:
            # Database connection
            - name: DB_HOST
              value: "mysql"
            
            - name: DB_PORT
              value: "3306"
            
            - name: DB_NAME
              value: "myapp"
            
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: user
            
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
```

---

## Complete Example: App with MongoDB

### 1. MongoDB Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongo-secret
type: Opaque
data:
  username: YWRtaW4=
  password: cGFzczEyMw==
```

### 2. MongoDB Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongodb
          image: mongo:7.0
          ports:
            - containerPort: 27017
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: username
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: password
            - name: MONGO_INITDB_DATABASE
              value: "myapp"
```

### 3. MongoDB Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb
spec:
  selector:
    app: mongodb
  ports:
    - port: 27017
```

### 4. App Deployment (Connecting to MongoDB)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nodejs-app
  template:
    metadata:
      labels:
        app: nodejs-app
    spec:
      containers:
        - name: nodejs-app
          image: nodejs-app:1.0
          ports:
            - containerPort: 3000
          
          env:
            # MongoDB connection
            - name: MONGO_HOST
              value: "mongodb"
            
            - name: MONGO_PORT
              value: "27017"
            
            - name: MONGO_DB
              value: "myapp"
            
            - name: MONGO_USER
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: username
            
            - name: MONGO_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: password
            
            # Connection string
            - name: MONGO_URL
              value: "mongodb://$(MONGO_USER):$(MONGO_PASSWORD)@$(MONGO_HOST):$(MONGO_PORT)/$(MONGO_DB)"
```

---

## Quick Reference

### MySQL Environment Variables

```yaml
env:
  - name: MYSQL_ROOT_PASSWORD    # Root password (REQUIRED)
    value: "root123"
  
  - name: MYSQL_DATABASE          # Database name (optional)
    value: "myapp"
  
  - name: MYSQL_USER             # App user (optional)
    value: "appuser"
  
  - name: MYSQL_PASSWORD         # App password (optional)
    value: "pass123"
```

### MongoDB Environment Variables

```yaml
env:
  - name: MONGO_INITDB_ROOT_USERNAME    # Root username (REQUIRED)
    value: "admin"
  
  - name: MONGO_INITDB_ROOT_PASSWORD    # Root password (REQUIRED)
    value: "pass123"
  
  - name: MONGO_INITDB_DATABASE         # Database name (optional)
    value: "myapp"
```

---

## 5 Ways to Set Environment Variables

### 1. Direct Value
```yaml
env:
  - name: MY_VAR
    value: "my-value"
```

### 2. From ConfigMap (Single Key)
```yaml
env:
  - name: MY_VAR
    valueFrom:
      configMapKeyRef:
        name: my-config
        key: my-key
```

### 3. From Secret (Single Key)
```yaml
env:
  - name: MY_VAR
    valueFrom:
      secretKeyRef:
        name: my-secret
        key: my-key
```

### 4. All Keys from ConfigMap
```yaml
envFrom:
  - configMapRef:
      name: my-config
```

### 5. All Keys from Secret
```yaml
envFrom:
  - secretRef:
      name: my-secret
```

---

## How to Create Base64 for Secrets

```bash
# Encode
echo -n "mypassword" | base64
# Output: bXlwYXNzd29yZA==

# Decode
echo "bXlwYXNzd29yZA==" | base64 -d
# Output: mypassword
```

---

## Apply Your Files

```bash
# Apply secret
kubectl apply -f mysql-secret.yaml

# Apply deployment
kubectl apply -f mysql-deployment.yaml

# Apply service
kubectl apply -f mysql-service.yaml

# Apply app
kubectl apply -f app-deployment.yaml

# Check pods
kubectl get pods

# Check env variables in pod
kubectl exec -it <pod-name> -- env
```

---

## Simple Complete Example (Copy-Paste Ready)

### File: mysql-all.yaml

```yaml
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
data:
  root-password: cm9vdDEyMw==
  user: YXBwdXNlcg==
  password: YXBwcGFzczEyMw==
---
# MySQL Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
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
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: root-password
            - name: MYSQL_DATABASE
              value: "myapp"
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: user
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
---
# MySQL Service
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  selector:
    app: mysql
  ports:
    - port: 3306
---
# App Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: nginx  # Replace with your app image
          env:
            - name: DB_HOST
              value: "mysql"
            - name: DB_PORT
              value: "3306"
            - name: DB_NAME
              value: "myapp"
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: user
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
```

**Deploy:**
```bash
kubectl apply -f mysql-all.yaml
```
