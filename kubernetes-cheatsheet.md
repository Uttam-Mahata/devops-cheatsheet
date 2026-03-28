# Kubernetes Cheatsheet

> A comprehensive reference for Kubernetes — from cluster basics to production-grade workloads.

---

## Table of Contents

- [Core Concepts](#core-concepts)
- [kubectl Setup & Config](#kubectl-setup--config)
- [Namespaces](#namespaces)
- [Pods](#pods)
- [Deployments](#deployments)
- [Services](#services)
- [ConfigMaps & Secrets](#configmaps--secrets)
- [Persistent Volumes & Storage](#persistent-volumes--storage)
- [Ingress](#ingress)
- [StatefulSets](#statefulsets)
- [DaemonSets](#daemonsets)
- [Jobs & CronJobs](#jobs--cronjobs)
- [RBAC](#rbac)
- [Resource Management & Autoscaling](#resource-management--autoscaling)
- [Probes & Health Checks](#probes--health-checks)
- [Logging & Debugging](#logging--debugging)
- [Helm](#helm)
- [Advanced Scenarios](#advanced-scenarios)

---

## Core Concepts

| Object            | Purpose                                                         |
|-------------------|-----------------------------------------------------------------|
| **Pod**           | Smallest deployable unit — one or more containers               |
| **Deployment**    | Manages stateless pods with rollout & rollback                  |
| **StatefulSet**   | Like Deployment but for stateful apps (stable IDs, storage)     |
| **DaemonSet**     | Runs one pod per node (e.g., log collectors, monitoring agents) |
| **Job**           | Runs a task to completion                                       |
| **CronJob**       | Schedules Jobs on a cron expression                             |
| **Service**       | Stable network endpoint for a set of pods                       |
| **Ingress**       | HTTP/HTTPS routing rules into the cluster                       |
| **ConfigMap**     | Non-sensitive config key-value pairs                            |
| **Secret**        | Sensitive config (passwords, tokens, certs)                     |
| **PV / PVC**      | Persistent storage abstraction                                  |
| **Namespace**     | Virtual cluster for resource isolation                          |
| **ServiceAccount**| Identity for pods to interact with the API server               |
| **HPA**           | Horizontal Pod Autoscaler — scales pods by metrics              |
| **Node**          | Worker machine (VM or physical)                                 |

---

## kubectl Setup & Config

```bash
# Check kubectl version
kubectl version --client

# View current context
kubectl config current-context

# List all contexts
kubectl config get-contexts

# Switch context
kubectl config use-context my-cluster

# Set default namespace for current context
kubectl config set-context --current --namespace=production

# View full kubeconfig
kubectl config view

# Merge kubeconfigs
KUBECONFIG=~/.kube/config:~/new-cluster.yaml kubectl config view --merge --flatten > ~/.kube/merged

# Get cluster info
kubectl cluster-info

# Check all API resources available
kubectl api-resources

# Explain a resource (built-in docs)
kubectl explain pod
kubectl explain pod.spec.containers
```

### Useful Aliases

```bash
alias k=kubectl
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deployments'
alias kns='kubectl config set-context --current --namespace'
alias kctx='kubectl config use-context'
```

---

## Namespaces

```bash
# List namespaces
kubectl get namespaces

# Create a namespace
kubectl create namespace staging

# Set current namespace
kubectl config set-context --current --namespace=staging

# Run any command in a specific namespace
kubectl get pods -n production

# Run across all namespaces
kubectl get pods --all-namespaces
kubectl get pods -A

# Delete a namespace (and everything inside it)
kubectl delete namespace staging
```

### `namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    env: staging
```

---

## Pods

```bash
# List pods in current namespace
kubectl get pods

# Wide output (node, IP)
kubectl get pods -o wide

# Watch pods in real time
kubectl get pods -w

# Describe pod (events, conditions, resource usage)
kubectl describe pod my-pod

# Get pod YAML
kubectl get pod my-pod -o yaml

# Run a temporary pod for debugging
kubectl run debug --image=busybox --rm -it --restart=Never -- sh

# Run a pod with a command
kubectl run nginx --image=nginx --port=80

# Delete a pod
kubectl delete pod my-pod

# Delete pod immediately (skip graceful shutdown)
kubectl delete pod my-pod --grace-period=0 --force
```

### `pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: default
  labels:
    app: my-app
    tier: backend
spec:
  containers:
    - name: app
      image: myapp:1.0.0
      ports:
        - containerPort: 3000
      env:
        - name: NODE_ENV
          value: production
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "512Mi"
      livenessProbe:
        httpGet:
          path: /health
          port: 3000
        initialDelaySeconds: 15
        periodSeconds: 20
      readinessProbe:
        httpGet:
          path: /ready
          port: 3000
        initialDelaySeconds: 5
        periodSeconds: 10
  restartPolicy: Always
```

---

## Deployments

```bash
# Create a deployment imperatively
kubectl create deployment myapp --image=myapp:1.0.0 --replicas=3

# Apply a manifest
kubectl apply -f deployment.yaml

# View deployments
kubectl get deployments

# Scale a deployment
kubectl scale deployment myapp --replicas=5

# Update the image (triggers rolling update)
kubectl set image deployment/myapp app=myapp:2.0.0

# View rollout status
kubectl rollout status deployment/myapp

# View rollout history
kubectl rollout history deployment/myapp

# Rollback to previous version
kubectl rollout undo deployment/myapp

# Rollback to a specific revision
kubectl rollout undo deployment/myapp --to-revision=2

# Pause rollout
kubectl rollout pause deployment/myapp

# Resume rollout
kubectl rollout resume deployment/myapp

# Delete deployment
kubectl delete deployment myapp
```

### `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # max extra pods during update
      maxUnavailable: 0     # zero downtime
  template:
    metadata:
      labels:
        app: myapp
        version: "2.0.0"
    spec:
      containers:
        - name: app
          image: myapp:2.0.0
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: myapp-config
            - secretRef:
                name: myapp-secrets
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 20
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
      affinity:
        podAntiAffinity:               # spread pods across nodes
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values: [myapp]
                topologyKey: kubernetes.io/hostname
```

---

## Services

```bash
# List services
kubectl get svc

# Expose a deployment as a service
kubectl expose deployment myapp --port=80 --target-port=3000

# Describe a service
kubectl describe svc myapp

# Delete a service
kubectl delete svc myapp

# Port-forward a service to localhost (dev access)
kubectl port-forward svc/myapp 8080:80

# Port-forward a specific pod
kubectl port-forward pod/myapp-xyz 8080:3000
```

### Service Types

```yaml
# ── ClusterIP (internal only — default) ─────────────────────
apiVersion: v1
kind: Service
metadata:
  name: myapp-clusterip
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP

---
# ── NodePort (exposes on each node's IP) ────────────────────
apiVersion: v1
kind: Service
metadata:
  name: myapp-nodeport
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 3000
      nodePort: 30080      # range: 30000–32767
  type: NodePort

---
# ── LoadBalancer (cloud provider LB) ────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: myapp-lb
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 3000
  type: LoadBalancer

---
# ── Headless (direct pod DNS — used by StatefulSets) ────────
apiVersion: v1
kind: Service
metadata:
  name: myapp-headless
spec:
  clusterIP: None
  selector:
    app: myapp
  ports:
    - port: 3000
```

| Type           | Accessible From         | Use Case                          |
|----------------|-------------------------|-----------------------------------|
| `ClusterIP`    | Inside cluster only     | Internal microservice traffic     |
| `NodePort`     | Outside via node IP     | Dev/testing, on-prem              |
| `LoadBalancer` | Outside via cloud LB IP | Production (GKE, EKS, AKS)       |
| `Headless`     | Direct pod DNS          | StatefulSets, custom discovery    |

---

## ConfigMaps & Secrets

### ConfigMaps

```bash
# Create from literal values
kubectl create configmap app-config \
  --from-literal=NODE_ENV=production \
  --from-literal=PORT=3000

# Create from a file
kubectl create configmap nginx-config --from-file=nginx.conf

# Create from a directory
kubectl create configmap app-config --from-file=./config/

# View configmap
kubectl get configmap app-config -o yaml

# Edit in-place
kubectl edit configmap app-config
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  NODE_ENV: production
  PORT: "3000"
  DATABASE_HOST: postgres-service
  app.properties: |
    max.connections=100
    timeout=30
```

### Secrets

```bash
# Create a generic secret
kubectl create secret generic app-secrets \
  --from-literal=DB_PASSWORD=s3cr3t \
  --from-literal=JWT_SECRET=myjwtsecret

# Create from a file
kubectl create secret generic tls-certs \
  --from-file=tls.crt --from-file=tls.key

# Create a TLS secret
kubectl create secret tls myapp-tls \
  --cert=tls.crt --key=tls.key

# Create a Docker registry pull secret
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass \
  --docker-email=user@example.com

# Decode a secret value
kubectl get secret app-secrets -o jsonpath='{.data.DB_PASSWORD}' | base64 --decode
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  DB_PASSWORD: czNjcjN0          # base64 encoded
  JWT_SECRET: bXlqd3RzZWNyZXQ=
```

### Using ConfigMaps & Secrets in Pods

```yaml
spec:
  containers:
    - name: app
      image: myapp:latest
      # ── Inject all keys as env vars ───────────────────────
      envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secrets
      # ── Inject specific keys ──────────────────────────────
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: DB_PASSWORD
      # ── Mount as files ────────────────────────────────────
      volumeMounts:
        - name: config-volume
          mountPath: /app/config
  volumes:
    - name: config-volume
      configMap:
        name: app-config
```

---

## Persistent Volumes & Storage

```bash
# List PersistentVolumes
kubectl get pv

# List PersistentVolumeClaims
kubectl get pvc

# Describe a PVC
kubectl describe pvc my-pvc

# Delete a PVC
kubectl delete pvc my-pvc
```

### PersistentVolume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce          # RWO: one node | RWX: many nodes | ROX: many nodes read-only
  persistentVolumeReclaimPolicy: Retain   # Retain | Delete | Recycle
  storageClassName: standard
  hostPath:
    path: /data/my-pv        # for local/dev only
```

### PersistentVolumeClaim

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
      storage: 5Gi
  storageClassName: standard
```

### Dynamic Provisioning — StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
```

### Mount PVC in a Pod

```yaml
spec:
  containers:
    - name: app
      volumeMounts:
        - mountPath: /data
          name: storage
  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: my-pvc
```

---

## Ingress

```bash
# List ingresses
kubectl get ingress

# Describe ingress
kubectl describe ingress myapp-ingress
```

### `ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
      secretName: myapp-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

---

## StatefulSets

> Use StatefulSets for stateful apps that need stable network IDs and persistent storage (databases, Kafka, Zookeeper).

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless   # must match a headless service
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: pg-secrets
                  key: password
          volumeMounts:
            - name: pgdata
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:            # each pod gets its own PVC
    - metadata:
        name: pgdata
      spec:
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 10Gi
        storageClassName: fast-ssd
```

```bash
# Pods are named with stable ordinal index
# postgres-0, postgres-1, postgres-2

# Scale StatefulSet
kubectl scale statefulset postgres --replicas=5

# Delete without deleting PVCs
kubectl delete statefulset postgres --cascade=orphan
```

---

## DaemonSets

> Runs exactly one pod on every node (or a subset via nodeSelector/tolerations). Used for log collectors, monitoring agents, network plugins.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule           # run on control-plane nodes too
      containers:
        - name: fluentd
          image: fluent/fluentd:v1.16
          volumeMounts:
            - name: varlog
              mountPath: /var/log
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
```

---

## Jobs & CronJobs

### Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 3              # retry up to 3 times on failure
  template:
    spec:
      restartPolicy: Never     # Never | OnFailure
      containers:
        - name: migrate
          image: myapp:latest
          command: ["npx", "prisma", "migrate", "deploy"]
          envFrom:
            - secretRef:
                name: app-secrets
```

### CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-report
spec:
  schedule: "0 2 * * *"        # every day at 02:00 UTC
  concurrencyPolicy: Forbid    # Allow | Forbid | Replace
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: reporter
              image: myapp:latest
              command: ["node", "scripts/report.js"]
```

```bash
# List jobs
kubectl get jobs

# List cronjobs
kubectl get cronjobs

# Trigger a CronJob manually
kubectl create job --from=cronjob/nightly-report manual-run

# View job logs
kubectl logs job/db-migration
```

---

## RBAC

### ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
```

### Role & RoleBinding (namespace-scoped)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
  - kind: ServiceAccount
    name: myapp-sa
    namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRole & ClusterRoleBinding (cluster-wide)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes
subjects:
  - kind: ServiceAccount
    name: myapp-sa
    namespace: production
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
# Check what a user/SA can do
kubectl auth can-i list pods --as=system:serviceaccount:production:myapp-sa
kubectl auth can-i create deployments --namespace=production

# List all RBAC roles
kubectl get roles -A
kubectl get clusterroles
```

---

## Resource Management & Autoscaling

### Requests & Limits

```yaml
resources:
  requests:            # minimum guaranteed resources
    cpu: "100m"        # 100 millicores = 0.1 CPU
    memory: "128Mi"
  limits:              # maximum allowed resources
    cpu: "500m"
    memory: "512Mi"
```

### LimitRange (default limits for a namespace)

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
```

### ResourceQuota (cap total usage per namespace)

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    pods: "50"
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
```

### Horizontal Pod Autoscaler (HPA)

```bash
# Create HPA imperatively
kubectl autoscale deployment myapp --cpu-percent=70 --min=2 --max=10

# View HPA
kubectl get hpa
```

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

---

## Probes & Health Checks

```yaml
containers:
  - name: app
    image: myapp:latest

    # ── Liveness: restart pod if failing ──────────────────
    livenessProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 15    # wait before first probe
      periodSeconds: 20          # probe interval
      timeoutSeconds: 5          # probe timeout
      failureThreshold: 3        # failures before restart

    # ── Readiness: remove from LB if failing ──────────────
    readinessProbe:
      httpGet:
        path: /ready
        port: 3000
      initialDelaySeconds: 5
      periodSeconds: 10

    # ── Startup: protect slow-starting containers ─────────
    startupProbe:
      httpGet:
        path: /health
        port: 3000
      failureThreshold: 30       # allow up to 30 * 10s = 5 min startup
      periodSeconds: 10
```

### Probe Types

| Type      | How                                               |
|-----------|---------------------------------------------------|
| `httpGet` | HTTP GET — success if status 200–399              |
| `tcpSocket`| TCP connection — success if port opens           |
| `exec`    | Run a command — success if exit code is 0         |
| `grpc`    | gRPC health check protocol                        |

---

## Logging & Debugging

```bash
# View pod logs
kubectl logs my-pod

# Stream logs
kubectl logs -f my-pod

# Logs from a specific container in a multi-container pod
kubectl logs my-pod -c sidecar

# Logs from a previous (crashed) container instance
kubectl logs my-pod --previous

# Tail last N lines
kubectl logs my-pod --tail=100

# Describe pod (great for scheduling/startup issues)
kubectl describe pod my-pod

# Get events in a namespace (sorted by time)
kubectl get events --sort-by='.lastTimestamp'

# Get events for a specific pod
kubectl get events --field-selector involvedObject.name=my-pod

# Execute into a running pod
kubectl exec -it my-pod -- bash
kubectl exec -it my-pod -c sidecar -- sh

# Run a temporary debug pod
kubectl run debug --image=busybox --rm -it --restart=Never -- sh

# Debug with same node as a pod (node-level access)
kubectl debug node/my-node -it --image=ubuntu

# Copy files from a pod
kubectl cp my-pod:/app/logs/error.log ./error.log

# Check API server connectivity from inside a pod
kubectl exec -it my-pod -- curl http://kubernetes.default.svc/healthz

# Top nodes / pods (requires metrics-server)
kubectl top nodes
kubectl top pods
kubectl top pods --sort-by=memory
```

---

## Helm

```bash
# Add a chart repository
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami

# Update repos
helm repo update

# Search for charts
helm search repo nginx
helm search hub wordpress

# Install a chart
helm install my-nginx bitnami/nginx

# Install with custom values
helm install my-app ./mychart -f values.prod.yaml

# Install with inline overrides
helm install my-app ./mychart --set image.tag=2.0.0 --set replicas=3

# Install into a namespace (create if needed)
helm install my-app bitnami/nginx -n production --create-namespace

# List releases
helm list
helm list -A                    # all namespaces

# Upgrade a release
helm upgrade my-app ./mychart -f values.prod.yaml

# Upgrade or install if not exists
helm upgrade --install my-app ./mychart -f values.prod.yaml

# Rollback to previous release
helm rollback my-app

# Rollback to specific revision
helm rollback my-app 2

# View release history
helm history my-app

# Dry run (show manifests without applying)
helm install my-app ./mychart --dry-run --debug

# Render templates locally
helm template my-app ./mychart -f values.yaml

# Uninstall a release
helm uninstall my-app

# Get values of a deployed release
helm get values my-app

# Package a chart
helm package ./mychart
```

---

## Advanced Scenarios

### Scenario 1 — Zero-downtime rolling deployment

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # spin up 1 extra pod first
      maxUnavailable: 0  # never take a pod down before new one is ready
```

```bash
kubectl set image deployment/myapp app=myapp:3.0.0
kubectl rollout status deployment/myapp   # wait and watch
```

### Scenario 2 — Blue/Green deployment

```bash
# Deploy "green" alongside "blue"
kubectl apply -f deployment-green.yaml    # label: version=green

# Test green internally
kubectl port-forward svc/myapp-green 8080:80

# Switch traffic by updating the service selector
kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'

# Remove old "blue" deployment when satisfied
kubectl delete deployment myapp-blue
```

### Scenario 3 — Run a DB migration before rolling out new pods

```yaml
# Use an init container to run migrations before the app starts
spec:
  initContainers:
    - name: migrate
      image: myapp:3.0.0
      command: ["npx", "prisma", "migrate", "deploy"]
      envFrom:
        - secretRef:
            name: app-secrets
  containers:
    - name: app
      image: myapp:3.0.0
```

### Scenario 4 — Force reschedule all pods in a deployment

```bash
kubectl rollout restart deployment/myapp
```

### Scenario 5 — Drain a node for maintenance

```bash
# Cordon (prevent new pods from scheduling)
kubectl cordon my-node

# Drain (evict all pods gracefully)
kubectl drain my-node --ignore-daemonsets --delete-emptydir-data

# Perform maintenance...

# Uncordon when done
kubectl uncordon my-node
```

### Scenario 6 — Debug a CrashLoopBackOff pod

```bash
# Check the logs of the crashed instance
kubectl logs my-pod --previous

# Describe pod for events and exit codes
kubectl describe pod my-pod

# Override the entrypoint to prevent crash and inspect
kubectl debug my-pod -it --copy-to=debug-pod --image=busybox -- sh
```

### Scenario 7 — Canary deployment (send 10% traffic to new version)

```yaml
# myapp-stable: 9 replicas (label version=stable)
# myapp-canary: 1 replica  (label version=canary)
# Service selector matches only "app=myapp" — hits both proportionally

kubectl scale deployment myapp-stable --replicas=9
kubectl scale deployment myapp-canary --replicas=1
```

### Scenario 8 — Inject a sidecar for logging

```yaml
spec:
  containers:
    - name: app
      image: myapp:latest
      volumeMounts:
        - name: logs
          mountPath: /app/logs
    - name: log-shipper              # sidecar container
      image: fluent/fluent-bit
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
  volumes:
    - name: logs
      emptyDir: {}
```

### Scenario 9 — Force delete a stuck namespace

```bash
# Patch the finalizers to remove the block
kubectl get namespace stuck-ns -o json \
  | jq '.spec.finalizers = []' \
  | kubectl replace --raw "/api/v1/namespaces/stuck-ns/finalize" -f -
```

### Scenario 10 — Multi-container pod with shared process namespace

```yaml
spec:
  shareProcessNamespace: true       # containers can see each other's processes
  containers:
    - name: app
      image: myapp:latest
    - name: debugger
      image: busybox
      command: ["sleep", "3600"]
```

```bash
# From the debugger container, inspect the app process
kubectl exec -it my-pod -c debugger -- ps aux
```

---

## Quick Reference Card

| Task                              | Command                                                   |
|-----------------------------------|-----------------------------------------------------------|
| Get all resources in namespace    | `kubectl get all -n <ns>`                                 |
| Apply a manifest                  | `kubectl apply -f file.yaml`                              |
| Delete from manifest              | `kubectl delete -f file.yaml`                             |
| Shell into a pod                  | `kubectl exec -it <pod> -- bash`                          |
| Stream pod logs                   | `kubectl logs -f <pod>`                                   |
| Port-forward a service            | `kubectl port-forward svc/<name> 8080:80`                 |
| Scale a deployment                | `kubectl scale deployment <name> --replicas=5`            |
| Update image                      | `kubectl set image deployment/<name> app=<image>:<tag>`   |
| Rollback deployment               | `kubectl rollout undo deployment/<name>`                  |
| Watch pods live                   | `kubectl get pods -w`                                     |
| Drain a node                      | `kubectl drain <node> --ignore-daemonsets`                |
| Force restart all pods            | `kubectl rollout restart deployment/<name>`               |
| Top pods by memory                | `kubectl top pods --sort-by=memory`                       |

---

> In Kubernetes, everything is a resource. Describe it, watch it, and let the control loop do the rest.
