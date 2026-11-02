# Helm: The Kubernetes Package Manager

> Helm packages Kubernetes YAML into a **chart** with configurable **values**,
> and manages installs, upgrades and rollbacks as **releases**.

---

## The Big Picture

**Problem:** a real app needs many YAML files (Deployment, Service, HPA, Ingress...). Changing replicas, image tags or resources per environment means copying and editing files by hand, and there is no easy rollback.

**Solution:** write the YAML once as templates, put the changing parts in a values file, and let Helm render and apply it.

| Linux | Kubernetes |
|---|---|
| `apt` / `yum` | **Helm** |
| a package | a **chart** |
| an installed package | a **release** |
| a package repository | a **chart repository** |

```
Chart (templates) + Values  ->  Helm renders final YAML  ->  applied to cluster  ->  Release (with history)
```

| Term | Meaning |
|---|---|
| **Chart** | The package: templates + default values |
| **Values** | The settings you can change (`values.yaml`, `-f file`, `--set`) |
| **Release** | One installed copy of a chart, with a name. Install the same chart twice and you get two releases |
| **Repository** | A place that stores and shares charts |

Helm keeps each release's history in the cluster (as Secrets in the release's namespace). That history is what makes rollback possible. There is no server-side component to install.

---

## Step 1: Check Helm

```bash
helm version
```

If the command is missing, install it from [helm.sh/docs/intro/install](https://helm.sh/docs/intro/install/).

> **Helm 3 vs 4:** Helm 4.0.0 was released on 12 November 2025. Charts with `apiVersion: v2` (almost all charts, including the one below) still work. Some CLI flags and outputs changed, so if a flag from an older tutorial is rejected, run `helm <command> --help` to see the name in your version.

---

## Step 2: Build a Chart (your nginx app)

### Folder structure

```
webapp/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    └── service.yaml
```

### Chart.yaml

```yaml
apiVersion: v2
name: webapp
description: nginx app with a Service
version: 0.1.0          # version of the chart
appVersion: "1.0"       # version of the app
```

### values.yaml (the settings you change)

```yaml
replicaCount: 1

image:
  repository: nginx
  tag: latest

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 128Mi
```

### templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app
  namespace: {{ .Release.Namespace }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: nginx
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 80
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

### templates/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-service
  namespace: {{ .Release.Namespace }}
spec:
  selector:
    app: {{ .Release.Name }}
  ports:
    - port: 80
      targetPort: 80
```

### The placeholders

| Placeholder | Where it comes from |
|---|---|
| `{{ .Values.replicaCount }}` | `values.yaml`, `-f` file or `--set` |
| `{{ .Release.Name }}` | The release name you give at install |
| `{{ .Release.Namespace }}` | The namespace you install into |
| `{{- toYaml ... \| nindent 12 }}` | Inserts a block of YAML with the right indentation |

---

## Step 3: Check Before Installing

```bash
helm lint ./webapp                  # catches chart mistakes
helm template my-nginx ./webapp     # prints the final YAML, installs nothing
```

Always run `helm template` first. Indentation errors show up here instead of at install time.

---

## Step 4: Release Life Cycle

```bash
# Install
helm install my-nginx ./webapp -n state --create-namespace

# See it
helm list -n state
kubectl get deploy,svc -n state

# Upgrade with a changed value
helm upgrade my-nginx ./webapp -n state --set replicaCount=3

# History and rollback
helm history my-nginx -n state
helm rollback my-nginx 1 -n state

# Remove everything the release created
helm uninstall my-nginx -n state
```

Install if new, upgrade if it already exists:

```bash
helm upgrade --install my-nginx ./webapp -n state --create-namespace
```

---

## Step 5: Overriding Values

```bash
helm install my-nginx ./webapp --set replicaCount=2 --set image.tag=1.27
helm install my-nginx ./webapp -f prod-values.yaml
```

Order of priority, later wins: **chart defaults, then `-f` files, then `--set`**.

```yaml
# prod-values.yaml (only the values that differ)
replicaCount: 3
resources:
  requests:
    cpu: 200m
    memory: 256Mi
```

---

## Step 6: Use a Ready-Made Chart

Example: metrics-server, which you set up with `kubectl apply` for HPA.

```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo update

helm install metrics-server metrics-server/metrics-server -n kube-system \
  --set "args={--kubelet-insecure-tls}"        # the kind fix
```

Inspect any chart:

```bash
helm search repo metrics-server
helm show values metrics-server/metrics-server    # every setting you can change
helm get values metrics-server -n kube-system     # what you changed
helm get manifest metrics-server -n kube-system   # the YAML that was applied
```

`helm show values` is the most useful habit. Read it before installing any chart.

---

## Command Cheat Sheet

| Task | Command |
|---|---|
| Add a repository | `helm repo add <name> <url>` |
| Refresh repositories | `helm repo update` |
| Search | `helm search repo <word>` |
| Show settings | `helm show values <chart>` |
| Check a chart | `helm lint <chart>` |
| Preview YAML | `helm template <release> <chart>` |
| Dry run | `helm install <release> <chart> --dry-run` |
| Install | `helm install <release> <chart> -n <ns>` |
| Install or upgrade | `helm upgrade --install <release> <chart> -n <ns>` |
| List releases | `helm list -A` |
| History | `helm history <release> -n <ns>` |
| Roll back | `helm rollback <release> <revision> -n <ns>` |
| Remove | `helm uninstall <release> -n <ns>` |

---

## Common Mistakes

| Mistake | Result |
|---|---|
| Forgetting `-n <namespace>` | The release goes in `default`, and `helm list` shows nothing in your namespace |
| Skipping `helm repo update` | You get old chart versions |
| Changing resources with `kubectl edit` after a Helm install | The next `helm upgrade` overwrites your change. Change values instead |
| Skipping `helm template` | Indentation errors appear at install time |
| Installing the same release name twice | Error. Use `helm upgrade --install` |
| Hard-coding names in templates | Two releases of the chart collide. Use `{{ .Release.Name }}` |
| Secrets in `values.yaml` committed to Git | Passwords exposed. Use Kubernetes Secrets or an external secrets tool |

---

## Interview Answers

- **What is Helm?** The package manager for Kubernetes. It bundles manifests as a chart with templates and values, and manages installs, upgrades and rollbacks as releases.
- **Chart vs release?** A chart is the package. A release is one installed instance of it in a cluster.
- **How do you roll back?** `helm history <release>` to find the revision, then `helm rollback <release> <revision>`.
- **Why Helm over `kubectl apply`?** Templating for different environments, versioned releases with history and rollback, and reusable, shareable packages.
- **Helm vs Kustomize?** Helm uses templates and values and tracks releases. Kustomize patches plain YAML with overlays and has no templating. Many teams use both.
