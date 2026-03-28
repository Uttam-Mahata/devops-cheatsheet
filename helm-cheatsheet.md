# Helm Cheatsheet

## Installation
```bash
# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# macOS
brew install helm

# Verify
helm version
```

---

## Core Concepts
| Term | Description |
|------|-------------|
| **Chart** | Package of Kubernetes manifests (like a deb/rpm) |
| **Release** | Deployed instance of a chart |
| **Repository** | Collection of charts |
| **Values** | Configuration injected into chart templates |
| **Revision** | Version of a release (increments on each upgrade) |

---

## Repository Management
```bash
# Add a repo
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# List repos
helm repo list

# Update repo cache
helm repo update

# Remove a repo
helm repo remove bitnami

# Search charts in repos
helm search repo nginx
helm search repo bitnami/wordpress --versions

# Search Artifact Hub
helm search hub nginx
```

---

## Installing Charts
```bash
# Basic install
helm install my-nginx ingress-nginx/ingress-nginx

# Install in specific namespace (create if not exists)
helm install my-nginx ingress-nginx/ingress-nginx \
  --namespace ingress \
  --create-namespace

# Install with custom values
helm install my-app bitnami/wordpress \
  --set wordpressUsername=admin \
  --set wordpressPassword=secret

# Install with values file
helm install my-app bitnami/wordpress -f values.yml

# Install specific chart version
helm install my-app bitnami/wordpress --version 15.2.3

# Dry run (preview manifests)
helm install my-app bitnami/wordpress --dry-run --debug

# Generate manifests without installing
helm template my-app bitnami/wordpress -f values.yml
helm template my-app bitnami/wordpress > rendered.yml
```

---

## Managing Releases
```bash
# List releases
helm list
helm list -n ingress              # specific namespace
helm list --all-namespaces        # all namespaces
helm list -a                      # include failed releases

# Status of a release
helm status my-nginx
helm status my-nginx -n ingress

# Get release info
helm get all my-nginx             # everything
helm get values my-nginx          # current values
helm get values my-nginx --all    # all values (including defaults)
helm get manifest my-nginx        # rendered manifests
helm get notes my-nginx           # release notes
helm get hooks my-nginx           # hooks

# Release history
helm history my-nginx
```

---

## Upgrading & Rollback
```bash
# Upgrade release
helm upgrade my-nginx ingress-nginx/ingress-nginx

# Upgrade with new values
helm upgrade my-app bitnami/wordpress -f values.yml
helm upgrade my-app bitnami/wordpress --set image.tag=2.0

# Install if not exists, upgrade if exists
helm upgrade --install my-nginx ingress-nginx/ingress-nginx \
  --namespace ingress \
  --create-namespace

# Rollback to previous revision
helm rollback my-nginx

# Rollback to specific revision
helm rollback my-nginx 2

# Rollback with wait
helm rollback my-nginx 1 --wait

# View history before rollback
helm history my-nginx
```

---

## Uninstalling
```bash
helm uninstall my-nginx
helm uninstall my-nginx -n ingress

# Keep history for rollback
helm uninstall my-nginx --keep-history
```

---

## Chart Development

### Create a Chart
```bash
helm create mychart
```

### Chart Structure
```
mychart/
  Chart.yaml          # Chart metadata
  values.yaml         # Default configuration values
  charts/             # Chart dependencies
  templates/          # Kubernetes manifest templates
    deployment.yaml
    service.yaml
    ingress.yaml
    _helpers.tpl       # Template helpers/partials
    NOTES.txt          # Post-install notes
  .helmignore
```

### Chart.yaml
```yaml
apiVersion: v2
name: mychart
description: A Helm chart for my app
type: application
version: 0.1.0         # Chart version
appVersion: "1.0.0"    # App version
```

---

## Templates

### Basic Template (`templates/deployment.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          ports:
            - containerPort: {{ .Values.service.port }}
          env:
            {{- range .Values.env }}
            - name: {{ .name }}
              value: {{ .value | quote }}
            {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

### values.yaml
```yaml
replicaCount: 1

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: ""

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: nginx
  annotations: {}
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix

resources:
  limits:
    cpu: 500m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 64Mi

env:
  - name: ENV
    value: production
```

---

## Template Functions & Directives

### Common Functions
```yaml
# Quote string
value: {{ .Values.name | quote }}

# Default value
tag: {{ .Values.image.tag | default "latest" }}

# Upper/lower case
name: {{ .Values.name | upper }}

# Indent (for embedding YAML blocks)
{{- toYaml .Values.resources | nindent 12 }}

# Trim whitespace
{{- .Values.name | trim }}

# Base64 encode (for Secrets)
password: {{ .Values.password | b64enc | quote }}

# Join list
hosts: {{ join "," .Values.hostList }}

# Required value (fails if missing)
name: {{ required "name is required" .Values.name }}
```

### Conditionals
```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}

{{- if eq .Values.service.type "LoadBalancer" }}
  loadBalancerIP: {{ .Values.service.loadBalancerIP }}
{{- else if eq .Values.service.type "NodePort" }}
  nodePort: {{ .Values.service.nodePort }}
{{- end }}
```

### Loops
```yaml
env:
  {{- range .Values.env }}
  - name: {{ .name }}
    value: {{ .value | quote }}
  {{- end }}

# Range over map
{{- range $key, $val := .Values.annotations }}
  {{ $key }}: {{ $val | quote }}
{{- end }}
```

### Named Templates (`_helpers.tpl`)
```yaml
{{- define "mychart.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "mychart.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```

---

## Built-in Objects
```yaml
{{ .Release.Name }}        # Release name
{{ .Release.Namespace }}   # Namespace
{{ .Release.Service }}     # "Helm"
{{ .Release.IsUpgrade }}   # true on upgrade
{{ .Release.IsInstall }}   # true on install

{{ .Chart.Name }}          # Chart name
{{ .Chart.Version }}       # Chart version
{{ .Chart.AppVersion }}    # App version

{{ .Values.myKey }}        # Value from values.yaml

{{ .Files.Get "config.txt" }}    # Read file from chart
```

---

## Chart Dependencies
```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
  - name: redis
    version: "17.x.x"
    repository: https://charts.bitnami.com/bitnami
    alias: cache
```
```bash
helm dependency update        # Download dependencies to charts/
helm dependency list          # List dependencies
helm dependency build         # Rebuild charts/ from Chart.lock
```

---

## Hooks
```yaml
# templates/job-migrate.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}-migrate"
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: myapp:latest
          command: ["python", "manage.py", "migrate"]
      restartPolicy: Never
```

| Hook | Trigger |
|------|---------|
| `pre-install` | Before manifests are rendered |
| `post-install` | After all resources are loaded |
| `pre-upgrade` | Before upgrade |
| `post-upgrade` | After upgrade |
| `pre-rollback` | Before rollback |
| `post-rollback` | After rollback |
| `pre-delete` | Before deletion |
| `post-delete` | After deletion |

---

## Packaging & Publishing
```bash
# Package chart into .tgz
helm package mychart/
helm package mychart/ --destination ./dist

# Lint before packaging
helm lint mychart/
helm lint mychart/ --strict

# Host a local chart repo
helm repo index ./dist --url https://charts.example.com
# Serve with any HTTP server

# Push to OCI registry (Helm 3.8+)
helm push mychart-0.1.0.tgz oci://registry.example.com/charts
helm pull oci://registry.example.com/charts/mychart --version 0.1.0
```

---

## Useful Flags
```bash
--namespace / -n      # Target namespace
--create-namespace    # Create namespace if missing
--wait                # Wait until all resources are ready
--timeout 5m          # Timeout for --wait (default 5m)
--atomic              # Roll back on failure (implies --wait)
--force               # Force resource update (delete & recreate)
--debug               # Enable verbose output
--dry-run             # Simulate, don't apply
--no-hooks            # Skip hooks
--set key=val         # Override individual value
--set-string key=val  # Force value as string
-f values.yml         # Load values from file
```

---

## Common Patterns

### Override image tag in CI/CD
```bash
helm upgrade --install myapp ./chart \
  --set image.tag=$CI_COMMIT_SHA \
  --namespace production \
  --create-namespace \
  --atomic \
  --timeout 5m
```

### Multi-environment values
```bash
helm upgrade --install myapp ./chart \
  -f values.yaml \
  -f values.production.yaml \
  --set image.tag=$TAG
```

### Debug rendered templates
```bash
helm template myapp ./chart -f values.yaml | kubectl apply --dry-run=client -f -
```
