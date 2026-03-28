# Helm

Helm is the package manager for Kubernetes. It allows you to define, install, and upgrade complex Kubernetes applications using reusable, versioned packages called **charts**. Helm abstracts the complexity of managing multiple Kubernetes manifests by treating them as a single deployable unit with configurable values, lifecycle hooks, and rollback support. Version 3 (current) removed the server-side Tiller component, making Helm purely client-side and improving security.

---

## Table of Contents

1. [Installation](#installation)
2. [Core Concepts](#core-concepts)
3. [Repository Management](#repository-management)
4. [Installing Charts](#installing-charts)
5. [Upgrading & Rollback](#upgrading--rollback)
6. [Uninstalling Releases](#uninstalling-releases)
7. [Working with Values](#working-with-values)
8. [Chart Development](#chart-development)
9. [Testing Charts](#testing-charts)
10. [Helm Secrets & Security](#helm-secrets--security)
11. [Helmfile](#helmfile)
12. [OCI Registries](#oci-registries)
13. [Common Patterns & Tips](#common-patterns--tips)

---

## Installation

```bash
# macOS (Homebrew)
brew install helm

# Linux – official install script
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Linux – manual binary install
HELM_VERSION=v3.14.0
curl -LO "https://get.helm.sh/helm-${HELM_VERSION}-linux-amd64.tar.gz"
tar -zxvf helm-${HELM_VERSION}-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm

# Windows (Chocolatey)
choco install kubernetes-helm

# Windows (Scoop)
scoop install helm

# Verify installation
helm version
# version.BuildInfo{Version:"v3.14.0", ...}

# Enable shell completion (bash)
helm completion bash > /etc/bash_completion.d/helm
source /etc/bash_completion.d/helm
```

---

## Core Concepts

Understanding these concepts is essential for working with Helm effectively.

### Chart

A **chart** is a Helm package — a directory (or `.tgz` archive) of files that describe a related set of Kubernetes resources.

```
mychart/
├── Chart.yaml          # Chart metadata (name, version, description)
├── values.yaml         # Default configuration values
├── charts/             # Dependencies (sub-charts)
├── templates/          # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl    # Named template helpers (not rendered directly)
│   └── NOTES.txt       # Post-install instructions (printed to user)
└── .helmignore         # Files to exclude from packaging
```

### Release

A **release** is a specific installation of a chart in a Kubernetes cluster. Installing the same chart twice creates two independent releases (e.g., `myapp-staging` and `myapp-production`). Each release has its own history and can be rolled back independently.

```bash
# List all releases in the current namespace
helm list

# List releases in all namespaces
helm list --all-namespaces
```

### Repository

A **repository** is an HTTP server that stores packaged charts (`.tgz` files) and an `index.yaml` manifest. Helm searches configured repos when installing charts by name.

### Values

**Values** are the configuration inputs for a chart. They are defined in `values.yaml` and can be overridden at install/upgrade time via `--set` flags or `-f` override files. Templates access them via the `.Values` object.

### Revision

Every install or upgrade creates a new **revision** number for that release. Helm stores the full history, enabling rollbacks.

```bash
# Show revision history for a release
helm history my-release
```

---

## Repository Management

```bash
# Add a repository
helm repo add stable https://charts.helm.sh/stable
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add jetstack https://charts.jetstack.io

# List configured repositories
helm repo list

# Update the local cache (fetch latest index from all repos)
helm repo update

# Search for charts in configured repos
helm search repo nginx
helm search repo nginx --versions       # show all available versions

# Search Artifact Hub (public chart registry)
helm search hub wordpress

# Remove a repository
helm repo remove stable
```

---

## Installing Charts

```bash
# Basic install
helm install <release-name> <chart>

# Install from a repo
helm install my-nginx ingress-nginx/ingress-nginx

# Install into a specific namespace (create if it doesn't exist)
helm install my-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# Install a specific chart version
helm install cert-manager jetstack/cert-manager --version v1.14.0

# Install from a local chart directory
helm install my-app ./mychart

# Install from a packaged .tgz
helm install my-app ./mychart-0.1.0.tgz

# Dry run – render templates and show what would be deployed, no actual changes
helm install my-app ./mychart --dry-run --debug

# Override values inline
helm install my-nginx ingress-nginx/ingress-nginx \
  --set controller.replicaCount=2 \
  --set controller.service.type=LoadBalancer

# Override values with a file
helm install my-nginx ingress-nginx/ingress-nginx \
  -f custom-values.yaml

# Wait for all resources to be ready before returning
helm install my-app ./mychart --wait --timeout 5m

# Generate a name automatically
helm install ./mychart --generate-name
```

| Flag | Description |
|---|---|
| `--namespace` / `-n` | Target Kubernetes namespace |
| `--create-namespace` | Create namespace if it does not exist |
| `--version` | Specific chart version to install |
| `--values` / `-f` | Path to a values override file (can repeat) |
| `--set` | Override a single value (repeatable) |
| `--set-string` | Force value to be treated as a string |
| `--set-file` | Set a value from a file's contents |
| `--dry-run` | Render templates but don't apply to cluster |
| `--debug` | Print rendered templates and verbose output |
| `--wait` | Wait until all pods/services are ready |
| `--timeout` | Time to wait for `--wait` (default `5m0s`) |
| `--atomic` | Roll back automatically on failed install |
| `--render-subchart-notes` | Print sub-chart NOTES.txt |

---

## Upgrading & Rollback

### Upgrade

```bash
# Upgrade an existing release to the latest chart version
helm upgrade my-nginx ingress-nginx/ingress-nginx

# Upgrade to a specific chart version
helm upgrade cert-manager jetstack/cert-manager --version v1.15.0 \
  --namespace cert-manager

# Upgrade with new values
helm upgrade my-app ./mychart \
  -f production-values.yaml \
  --set image.tag=v2.3.0

# Install if not present, upgrade if it is (idempotent)
helm upgrade --install my-app ./mychart \
  --namespace production \
  --create-namespace

# Upgrade and wait; roll back automatically on failure
helm upgrade my-app ./mychart --atomic --timeout 10m

# Cleanup old release history to limit revisions kept
helm upgrade my-app ./mychart --history-max 5

# Dry run an upgrade
helm upgrade my-app ./mychart --dry-run --debug
```

### Rollback

```bash
# Show release history
helm history my-app
# REVISION  STATUS      CHART         DESCRIPTION
# 1         superseded  mychart-0.1.0 Install complete
# 2         deployed    mychart-0.2.0 Upgrade complete

# Roll back to the previous revision
helm rollback my-app

# Roll back to a specific revision
helm rollback my-app 1

# Roll back and wait for resources
helm rollback my-app 1 --wait --timeout 5m

# Dry run rollback
helm rollback my-app 1 --dry-run
```

---

## Uninstalling Releases

```bash
# Uninstall a release (removes all Kubernetes resources managed by Helm)
helm uninstall my-app

# Uninstall from a specific namespace
helm uninstall my-app --namespace production

# Keep release history after uninstall (enables rollback to it later)
helm uninstall my-app --keep-history

# Dry run uninstall
helm uninstall my-app --dry-run
```

---

## Working with Values

### Inspecting Default Values

```bash
# Show the default values.yaml for a chart
helm show values ingress-nginx/ingress-nginx
helm show values ingress-nginx/ingress-nginx > default-values.yaml

# Show all chart information (Chart.yaml + values)
helm show all ingress-nginx/ingress-nginx

# Show only Chart.yaml metadata
helm show chart ingress-nginx/ingress-nginx

# Show NOTES.txt (post-install instructions)
helm show readme ingress-nginx/ingress-nginx
```

### Value Overrides

Values are merged from multiple sources in order (later sources win):

```
chart defaults (values.yaml) < -f file(s) < --set flags
```

```bash
# Single key override
helm upgrade my-app ./mychart --set replicaCount=3

# Nested key (dot notation)
helm upgrade my-app ./mychart --set image.repository=myorg/myapp

# Array element
helm upgrade my-app ./mychart --set "ingress.hosts[0].host=myapp.example.com"

# Multiple overrides
helm upgrade my-app ./mychart \
  --set replicaCount=3 \
  --set image.tag=v2.0 \
  --set resources.requests.memory=256Mi

# Override from a file
helm upgrade my-app ./mychart -f prod-values.yaml

# Multiple override files (merged in order)
helm upgrade my-app ./mychart \
  -f base-values.yaml \
  -f prod-values.yaml

# Force a value to be a string (avoid YAML type coercion)
helm upgrade my-app ./mychart --set-string "labels.version=1.0"

# Load a value from a file (e.g., a certificate)
helm upgrade my-app ./mychart \
  --set-file "tls.cert=./certs/server.crt"
```

### Viewing Computed Values for a Release

```bash
# Show the computed values used in the current release
helm get values my-app

# Show all values (including chart defaults)
helm get values my-app --all

# Show the rendered manifests for a deployed release
helm get manifest my-app

# Show the NOTES.txt output for a deployed release
helm get notes my-app

# Full info about a release
helm get all my-app
```

---

## Chart Development

### Creating a New Chart

```bash
# Scaffold a new chart
helm create mychart

# Resulting structure:
# mychart/
# ├── Chart.yaml
# ├── values.yaml
# ├── charts/
# └── templates/
#     ├── NOTES.txt
#     ├── _helpers.tpl
#     ├── deployment.yaml
#     ├── hpa.yaml
#     ├── ingress.yaml
#     ├── service.yaml
#     ├── serviceaccount.yaml
#     └── tests/
#         └── test-connection.yaml
```

### Chart.yaml

```yaml
apiVersion: v2                          # Helm 3 uses v2
name: mychart
description: A sample Helm chart
type: application                       # application or library
version: 0.1.0                          # chart version (SemVer)
appVersion: "1.2.3"                     # version of the application being packaged
keywords:
  - webapp
  - api
home: https://github.com/myorg/mychart
sources:
  - https://github.com/myorg/myapp
maintainers:
  - name: Alice
    email: alice@example.com

dependencies:
  - name: postgresql
    version: "~13.0.0"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled        # only install if this value is true
  - name: redis
    version: ">=18.0.0"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

```bash
# Download dependencies defined in Chart.yaml
helm dependency update mychart/

# List dependencies
helm dependency list mychart/
```

### Templates

Templates are Go templates with Helm-specific functions (using the [Sprig](http://masterminds.github.io/sprig/) library).

```yaml
# templates/deployment.yaml
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
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.port }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

**Key template objects:**

| Object | Description |
|---|---|
| `.Values` | Values from `values.yaml` and overrides |
| `.Chart` | Contents of `Chart.yaml` |
| `.Release.Name` | Name of the current release |
| `.Release.Namespace` | Namespace of the current release |
| `.Release.IsInstall` | True on first install |
| `.Release.IsUpgrade` | True on upgrades |
| `.Files` | Access to non-template files in the chart |
| `.Capabilities` | Info about the Kubernetes cluster capabilities |

### Helpers and Named Templates (`_helpers.tpl`)

Files prefixed with `_` are not rendered as Kubernetes manifests — they hold reusable template definitions.

```yaml
# templates/_helpers.tpl

{{/*
Expand the name of the chart.
*/}}
{{- define "mychart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "mychart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "mychart.labels" -}}
helm.sh/chart: {{ include "mychart.chart" . }}
{{ include "mychart.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "mychart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### Conditionals in Templates

```yaml
# Simple if/else
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "mychart.fullname" . }}
  {{- if .Values.ingress.annotations }}
  annotations:
    {{- toYaml .Values.ingress.annotations | nindent 4 }}
  {{- end }}
spec:
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ include "mychart.fullname" $ }}
                port:
                  number: {{ $.Values.service.port }}
          {{- end }}
    {{- end }}
{{- end }}
```

### Loops in Templates

```yaml
# Loop over a list
env:
  {{- range .Values.env }}
  - name: {{ .name }}
    value: {{ .value | quote }}
  {{- end }}

# Loop over a dict
{{- range $key, $val := .Values.configMap }}
  {{ $key }}: {{ $val | quote }}
{{- end }}

# Loop with index
{{- range $index, $host := .Values.hosts }}
  - host: {{ $host }}
{{- end }}
```

### values.yaml Best Practices

```yaml
# values.yaml
replicaCount: 1

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: ""                        # defaults to .Chart.AppVersion if empty

serviceAccount:
  create: true
  annotations: {}
  name: ""

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []

resources: {}
  # Uncomment to set resource limits:
  # limits:
  #   cpu: 100m
  #   memory: 128Mi
  # requests:
  #   cpu: 100m
  #   memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80

nodeSelector: {}
tolerations: []
affinity: {}
```

### Rendering Templates Locally

```bash
# Render all templates to stdout
helm template my-release ./mychart

# Render with value overrides
helm template my-release ./mychart \
  -f prod-values.yaml \
  --set image.tag=v2.0

# Render a specific template file
helm template my-release ./mychart \
  --show-only templates/deployment.yaml

# Render and validate against the cluster API
helm template my-release ./mychart | kubectl apply --dry-run=client -f -

# Lint the chart
helm lint ./mychart
helm lint ./mychart -f prod-values.yaml    # lint with specific values
```

---

## Testing Charts

Helm supports tests that run as Kubernetes Jobs or Pods after a release is installed.

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "mychart.fullname" . }}-test-connection"
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test           # marks this as a test hook
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args:
        - '--spider'
        - 'http://{{ include "mychart.fullname" . }}:{{ .Values.service.port }}'
  restartPolicy: Never
```

```bash
# Run tests for a deployed release
helm test my-release

# Run tests and show logs
helm test my-release --logs

# Common test hook annotations:
# "helm.sh/hook": test
# "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded,hook-failed
```

### Hooks

Hooks allow you to run actions at specific points in the release lifecycle:

| Hook | When it runs |
|---|---|
| `pre-install` | Before any resources are rendered and installed |
| `post-install` | After all resources are loaded into Kubernetes |
| `pre-upgrade` | Before an upgrade begins |
| `post-upgrade` | After an upgrade completes |
| `pre-delete` | Before any resources are deleted on uninstall |
| `post-delete` | After all resources have been deleted |
| `pre-rollback` | Before a rollback begins |
| `post-rollback` | After a rollback completes |
| `test` | When `helm test` is executed |

---

## Helm Secrets & Security

### helm-secrets Plugin

`helm-secrets` integrates with `sops` or `vals` to manage encrypted secret files.

```bash
# Install the plugin
helm plugin install https://github.com/jkroepke/helm-secrets

# Install sops (required backend)
brew install sops               # macOS
# or download binary from GitHub

# Encrypt a secrets file (using AWS KMS, GCP KMS, age, or PGP)
sops -e secrets.yaml > secrets.enc.yaml

# Use encrypted secrets in helm commands
helm secrets upgrade my-app ./mychart \
  -f values.yaml \
  -f secrets.enc.yaml    # helm-secrets transparently decrypts this

# Decrypt for inspection
helm secrets decrypt secrets.enc.yaml
```

### RBAC and Chart Security

```bash
# Use a dedicated service account with minimal permissions
# Limit which namespaces Helm can manage

# Verify chart signatures (OCI charts)
helm pull oci://registry-1.docker.io/bitnamicharts/nginx \
  --verify                      # requires chart to be signed

# Scan rendered manifests for security issues
helm template my-release ./mychart | kubesec scan -
helm template my-release ./mychart | trivy config -
```

---

## Helmfile

[Helmfile](https://github.com/helmfile/helmfile) is a declarative spec for deploying Helm charts, enabling you to manage multiple releases across environments in a single file.

```yaml
# helmfile.yaml
repositories:
  - name: ingress-nginx
    url: https://kubernetes.github.io/ingress-nginx
  - name: bitnami
    url: https://charts.bitnami.com/bitnami
  - name: jetstack
    url: https://charts.jetstack.io

environments:
  staging:
    values:
      - environments/staging/values.yaml
  production:
    values:
      - environments/production/values.yaml

releases:
  - name: ingress-nginx
    namespace: ingress-nginx
    chart: ingress-nginx/ingress-nginx
    version: "4.9.0"
    values:
      - controller:
          replicaCount: {{ if eq .Environment.Name "production" }}3{{ else }}1{{ end }}

  - name: cert-manager
    namespace: cert-manager
    chart: jetstack/cert-manager
    version: "v1.14.0"
    set:
      - name: installCRDs
        value: "true"

  - name: my-app
    namespace: {{ .Environment.Name }}
    chart: ./charts/my-app
    values:
      - values/my-app.yaml
      - values/my-app-{{ .Environment.Name }}.yaml
    secrets:
      - secrets/my-app-{{ .Environment.Name }}.enc.yaml
```

```bash
# Install helmfile
brew install helmfile                 # macOS
# or download binary from GitHub releases

# Apply all releases (install/upgrade as needed)
helmfile apply

# Apply to a specific environment
helmfile --environment production apply

# Diff (preview changes without applying)
helmfile diff

# Sync (apply without waiting for completion)
helmfile sync

# Destroy all releases
helmfile destroy

# Apply only specific releases
helmfile apply --selector name=my-app

# Lint all charts
helmfile lint
```

---

## OCI Registries

Helm 3 supports storing and pulling charts from OCI-compliant container registries (e.g., Docker Hub, GitHub Container Registry, ECR, ACR).

```bash
# Log in to an OCI registry
helm registry login registry-1.docker.io
helm registry login ghcr.io --username myuser --password-stdin < token.txt
helm registry login 123456789.dkr.ecr.us-east-1.amazonaws.com  # AWS ECR

# Pull a chart from an OCI registry
helm pull oci://registry-1.docker.io/bitnamicharts/nginx
helm pull oci://registry-1.docker.io/bitnamicharts/nginx --version 15.0.0

# Push a chart to an OCI registry
helm package ./mychart                              # creates mychart-0.1.0.tgz
helm push mychart-0.1.0.tgz oci://ghcr.io/myorg/charts

# Install directly from OCI registry
helm install my-nginx oci://registry-1.docker.io/bitnamicharts/nginx
helm install my-nginx oci://registry-1.docker.io/bitnamicharts/nginx \
  --version 15.0.0

# Inspect a chart in an OCI registry without downloading
helm show chart oci://registry-1.docker.io/bitnamicharts/nginx
helm show values oci://registry-1.docker.io/bitnamicharts/nginx

# Log out
helm registry logout registry-1.docker.io
```

---

## Common Patterns & Tips

1. **Always use `--atomic` in CI/CD pipelines.**  
   `helm upgrade --install my-app ./mychart --atomic` will automatically roll back to the last successful revision if the upgrade fails, preventing a half-deployed broken state in production.

2. **Prefer `upgrade --install` over separate install/upgrade logic.**  
   The `--install` flag on `helm upgrade` makes the command idempotent — it installs the release if it doesn't exist, or upgrades it if it does. This simplifies CI/CD pipelines.

3. **Version your chart and your app separately.**  
   `Chart.yaml` has both `version` (the chart packaging version) and `appVersion` (the application version inside). Keep them independently versioned so chart improvements don't force app version bumps.

4. **Use `helm template | kubectl apply` for GitOps workflows.**  
   Render manifests to files for GitOps tools (ArgoCD, Flux) to apply. This provides full visibility into what's deployed without requiring Helm in the GitOps agent itself.

5. **Store environment-specific values in separate files.**  
   Maintain `values-dev.yaml`, `values-staging.yaml`, `values-prod.yaml` rather than using `--set` for every difference. Commit these files to version control for an auditable configuration history.

6. **Use `helm lint` in every CI pipeline.**  
   `helm lint` catches YAML syntax errors, missing required values, and schema violations before you ever hit the cluster. Run it as the first step in any chart-related pipeline.

7. **Use `lookup` function carefully.**  
   The `lookup` function queries live cluster resources during template rendering. Avoid it in templates that need to work with `--dry-run` or in offline CI environments — it silently returns empty results instead of failing.

8. **Pin chart dependency versions with a tilde `~` range.**  
   In `Chart.yaml` dependencies, use `"~13.0.0"` to allow patch-level updates (13.0.x) but not minor version jumps. Always run `helm dependency update` and commit the updated `Chart.lock` file.

9. **Use `helm diff` plugin for upgrade previews.**  
   Install the `helm-diff` plugin (`helm plugin install https://github.com/databus23/helm-diff`) and run `helm diff upgrade` to see a kubectl-diff-style preview of every Kubernetes resource change before applying.

10. **Namespace your releases consistently.**  
    Always deploy with an explicit `--namespace` flag. Avoid using the `default` namespace for applications; isolate workloads with dedicated namespaces to improve security, resource quotas, and operational clarity.
