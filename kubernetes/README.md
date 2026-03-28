# Kubernetes Cheatsheet

Kubernetes (K8s) is an open-source container orchestration platform that automates deployment, scaling, and management of containerised workloads. It groups containers into logical units called Pods and provides declarative configuration, self-healing, service discovery, load balancing, automated rollouts and rollbacks, and horizontal scaling. Kubernetes runs on public cloud, on-premises, and edge environments, making it the de-facto standard for production container infrastructure.

---

## Table of Contents

1. [kubectl Basics](#kubectl-basics)
2. [Pods](#pods)
3. [Deployments](#deployments)
4. [Services](#services)
5. [Ingress](#ingress)
6. [ConfigMaps & Secrets](#configmaps--secrets)
7. [Persistent Volumes](#persistent-volumes)
8. [StatefulSets](#statefulsets)
9. [DaemonSets](#daemonsets)
10. [Jobs & CronJobs](#jobs--cronjobs)
11. [RBAC](#rbac)
12. [HPA & Autoscaling](#hpa--autoscaling)
13. [Resource Management](#resource-management)
14. [Probes & Health Checks](#probes--health-checks)
15. [Namespaces](#namespaces)
16. [Contexts & Config](#contexts--config)
17. [Common Patterns & Tips](#common-patterns--tips)

---

## kubectl Basics

`kubectl` is the command-line tool for interacting with the Kubernetes API server. It reads cluster configuration from `~/.kube/config`.

### General Syntax

```
kubectl <verb> <resource-type> [<name>] [flags]
```

Common verbs: `get`, `describe`, `create`, `apply`, `delete`, `edit`, `patch`, `scale`, `rollout`, `exec`, `logs`, `port-forward`, `top`, `explain`

### Global Flags

| Flag | Description |
|---|---|
| `-n <namespace>` | Target namespace (default: `default`) |
| `--all-namespaces` / `-A` | Operate across all namespaces |
| `-o yaml` | Output as YAML |
| `-o json` | Output as JSON |
| `-o wide` | Extra columns (e.g., node name, IP) |
| `--dry-run=client` | Validate locally without sending to API server |
| `--dry-run=server` | Validate via server without persisting |
| `--watch` / `-w` | Stream updates |
| `-l <selector>` | Filter by label selector |
| `--field-selector` | Filter by field (e.g., `status.phase=Running`) |
| `--show-labels` | Include labels column in output |
| `--context <ctx>` | Override active context |
| `-v <level>` | Verbosity (0-9); `-v=8` shows HTTP requests |

```bash
kubectl get pods -n kube-system
kubectl get all -A
kubectl get pod web-abc -o yaml
kubectl get nodes -o wide
kubectl api-resources                       # List all resource types
kubectl api-versions                        # List API versions
kubectl explain pod.spec.containers         # Inline API documentation
```

### `kubectl apply` vs `kubectl create`

- `kubectl apply` — Declarative; merges desired state with the live object. Safe for repeated runs. **Preferred for GitOps.**
- `kubectl create` — Imperative; fails if the resource already exists.

```bash
kubectl apply -f deployment.yaml
kubectl apply -f ./manifests/             # Apply entire directory
kubectl apply -k ./overlays/prod/         # Apply Kustomize overlay
kubectl create deployment nginx --image=nginx:alpine
```

### `kubectl delete`

```bash
kubectl delete -f deployment.yaml
kubectl delete pod web-abc
kubectl delete deployment,service -l app=web
kubectl delete pod web-abc --grace-period=0 --force  # Emergency force delete
```

---

## Pods

A Pod is the smallest deployable unit in Kubernetes. It wraps one or more tightly coupled containers that share network, storage, and lifecycle. In practice, Pods are managed by higher-level controllers (Deployments, StatefulSets) rather than created directly.

### Imperative Pod Creation

```bash
kubectl run nginx --image=nginx:alpine --port=80
kubectl run busybox --image=busybox:latest --rm -it -- sh    # Debug pod
kubectl run debug --image=nicolaka/netshoot --rm -it -- bash # Network debug
```

### Pod Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  namespace: default
  labels:
    app: web
    tier: frontend
spec:
  containers:
    - name: nginx
      image: nginx:1.25-alpine
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: "100m"
          memory: "64Mi"
        limits:
          cpu: "500m"
          memory: "128Mi"
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 10
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 15
        periodSeconds: 20
  restartPolicy: Always
```

### Inspecting Pods

```bash
kubectl get pods -o wide
kubectl describe pod web                        # Full detail including events
kubectl logs web                                # Container logs
kubectl logs web -c sidecar                     # Specific container in multi-container pod
kubectl logs web -f --tail 100                  # Stream last 100 lines
kubectl logs web --previous                     # Logs from previous (crashed) instance
kubectl exec -it web -- bash                    # Interactive shell
kubectl exec web -- cat /etc/nginx/nginx.conf   # One-off command
kubectl port-forward pod/web 8080:80            # Forward local port to pod
kubectl cp web:/var/log/app.log ./app.log       # Copy file from pod
kubectl top pod web                             # CPU/memory usage
```

---

## Deployments

A Deployment manages a ReplicaSet, which in turn manages a set of identical Pods. Deployments enable declarative updates, rolling rollouts, and rollbacks.

### Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: production
  labels:
    app: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # Extra pods during update
      maxUnavailable: 0     # Zero downtime
  template:
    metadata:
      labels:
        app: api
        version: "1.2.0"
    spec:
      containers:
        - name: api
          image: myorg/api:1.2.0
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: api-config
            - secretRef:
                name: api-secrets
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "512Mi"
```

### Managing Deployments

```bash
kubectl apply -f deployment.yaml

# Scaling
kubectl scale deployment api --replicas=5
kubectl autoscale deployment api --min=2 --max=10 --cpu-percent=70

# Rolling updates
kubectl set image deployment/api api=myorg/api:1.3.0
kubectl rollout status deployment/api               # Watch rollout progress

# Rollback
kubectl rollout undo deployment/api
kubectl rollout undo deployment/api --to-revision=3
kubectl rollout history deployment/api
kubectl rollout history deployment/api --revision=2

# Pause / Resume rollout
kubectl rollout pause deployment/api
kubectl rollout resume deployment/api

# Restart all pods (e.g., to pick up a new secret)
kubectl rollout restart deployment/api
```

---

## Services

A Service provides a stable virtual IP and DNS name for accessing a set of Pods. It load-balances traffic across Pod endpoints and decouples consumers from specific Pod IPs.

### Service Types

| Type | Description |
|---|---|
| `ClusterIP` | Default; accessible only within the cluster |
| `NodePort` | Exposes service on each node's IP at a static port (30000–32767) |
| `LoadBalancer` | Provisions an external cloud load balancer |
| `ExternalName` | Maps service to a DNS name (CNAME) |
| `Headless` | `clusterIP: None`; returns Pod IPs directly; used by StatefulSets |

### Service Manifest

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: api               # Must match Pod labels
  ports:
    - name: http
      port: 80             # Service port (cluster-internal)
      targetPort: 3000     # Container port
      protocol: TCP
---
# LoadBalancer for external traffic
apiVersion: v1
kind: Service
metadata:
  name: api-external
spec:
  type: LoadBalancer
  selector:
    app: api
  ports:
    - port: 443
      targetPort: 3000
```

### Working with Services

```bash
kubectl get svc -A
kubectl describe svc api
kubectl get endpoints api          # See which pods are behind the service

# Quick expose
kubectl expose deployment api --port=80 --target-port=3000 --type=ClusterIP

# Port forward to a service
kubectl port-forward svc/api 8080:80
```

---

## Ingress

An Ingress exposes HTTP/HTTPS routes from outside the cluster to services within it. It requires an Ingress Controller (e.g., nginx-ingress, Traefik, AWS ALB Ingress Controller) to be installed.

### Ingress Manifest

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls-cert
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
          - path: /v2
            pathType: Prefix
            backend:
              service:
                name: api-v2
                port:
                  number: 80
```

```bash
kubectl get ingress -A
kubectl describe ingress api-ingress
```

---

## ConfigMaps & Secrets

### ConfigMap

ConfigMaps store non-sensitive configuration data as key-value pairs or files. Pods can consume them as environment variables, command-line arguments, or mounted files.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: production
data:
  LOG_LEVEL: "info"
  PORT: "3000"
  DATABASE_HOST: "db.production.svc.cluster.local"
  app.properties: |
    feature.flag.dark-mode=true
    feature.flag.beta=false
```

```bash
# Imperative creation
kubectl create configmap api-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=PORT=3000 \
  --from-file=app.properties=./config/app.properties

kubectl get configmap api-config -o yaml
kubectl describe configmap api-config
```

### Secret

Secrets store sensitive data (passwords, tokens, TLS certificates). Values are base64-encoded at rest (encrypt at rest with etcd encryption for production).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: api-secrets
  namespace: production
type: Opaque
stringData:                   # Plain text; Kubernetes base64-encodes automatically
  DATABASE_PASSWORD: "s3cr3t"
  JWT_SECRET: "my-jwt-secret"
```

```bash
# Imperative creation
kubectl create secret generic api-secrets \
  --from-literal=DATABASE_PASSWORD=s3cr3t \
  --from-literal=JWT_SECRET=my-jwt-secret

# TLS secret
kubectl create secret tls api-tls-cert \
  --cert=server.crt \
  --key=server.key

# Docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=token \
  --docker-email=ops@example.com

kubectl get secret api-secrets -o jsonpath='{.data.DATABASE_PASSWORD}' | base64 -d
```

### Consuming ConfigMaps and Secrets in Pods

```yaml
spec:
  containers:
    - name: api
      image: myorg/api:1.0
      # As individual env vars
      env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: api-config
              key: LOG_LEVEL
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: DATABASE_PASSWORD
      # Load all keys as env vars
      envFrom:
        - configMapRef:
            name: api-config
        - secretRef:
            name: api-secrets
      # Mount as files
      volumeMounts:
        - name: config-vol
          mountPath: /etc/app
          readOnly: true
  volumes:
    - name: config-vol
      configMap:
        name: api-config
```

---

## Persistent Volumes

### Storage Concepts

| Resource | Description |
|---|---|
| `PersistentVolume (PV)` | Cluster-scoped storage resource provisioned by an admin or dynamically by a StorageClass |
| `PersistentVolumeClaim (PVC)` | Namespace-scoped request for storage by a user |
| `StorageClass` | Defines provisioner, parameters, and reclaim policy for dynamic provisioning |

### Access Modes

| Mode | Description |
|---|---|
| `ReadWriteOnce (RWO)` | Mounted read-write by a single node |
| `ReadOnlyMany (ROX)` | Mounted read-only by many nodes |
| `ReadWriteMany (RWX)` | Mounted read-write by many nodes |
| `ReadWriteOncePod (RWOP)` | Mounted read-write by a single pod (K8s ≥ 1.22) |

### PVC Manifest

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 20Gi
```

### StorageClass Example

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com        # AWS EBS CSI driver
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

```bash
kubectl get pv
kubectl get pvc -A
kubectl describe pvc postgres-pvc
kubectl get storageclass
```

---

## StatefulSets

StatefulSets manage stateful applications — databases, message queues — that require stable network identities, ordered deployment/scaling/deletion, and persistent storage per Pod.

### StatefulSet Manifest

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres-headless   # Must match a Headless Service
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
                  name: postgres-secrets
                  key: POSTGRES_PASSWORD
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:            # Each pod gets its own PVC
    - metadata:
        name: data
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None               # Headless — returns pod IPs
  selector:
    app: postgres
  ports:
    - port: 5432
```

Pods in a StatefulSet get stable identities: `postgres-0`, `postgres-1`, `postgres-2`. DNS: `postgres-0.postgres-headless.production.svc.cluster.local`.

---

## DaemonSets

A DaemonSet ensures that every (or selected) node runs exactly one copy of a Pod. Used for node-level services: log collectors (Fluentd, Filebeat), monitoring agents (Prometheus Node Exporter), network plugins, and security agents.

### DaemonSet Manifest

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true
      hostPID: true
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      containers:
        - name: node-exporter
          image: prom/node-exporter:v1.7.0
          ports:
            - containerPort: 9100
              hostPort: 9100
          securityContext:
            privileged: true
          volumeMounts:
            - name: proc
              mountPath: /host/proc
              readOnly: true
      volumes:
        - name: proc
          hostPath:
            path: /proc
```

```bash
kubectl get daemonset -A
kubectl rollout status daemonset/node-exporter -n monitoring
kubectl rollout restart daemonset/node-exporter -n monitoring
```

---

## Jobs & CronJobs

### Job

A Job creates one or more Pods and ensures a specified number of them successfully terminate. Used for batch processing, database migrations, and one-off tasks.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  namespace: production
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 3               # Retry up to 3 times on failure
  activeDeadlineSeconds: 300    # Fail if not done in 5 minutes
  template:
    spec:
      restartPolicy: OnFailure  # Required for Jobs (Never or OnFailure)
      containers:
        - name: migrate
          image: myorg/api:1.2.0
          command: ["node", "dist/migrate.js"]
          envFrom:
            - secretRef:
                name: api-secrets
```

```bash
kubectl apply -f job.yaml
kubectl get jobs
kubectl get pods --selector=job-name=db-migrate
kubectl logs job/db-migrate
kubectl delete job db-migrate
```

### CronJob

Runs Jobs on a repeating schedule using standard cron syntax.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
  namespace: production
spec:
  schedule: "0 2 * * *"            # Every day at 02:00 UTC
  timeZone: "UTC"
  concurrencyPolicy: Forbid         # Skip if previous run still active
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  startingDeadlineSeconds: 120
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: myorg/backup-tool:latest
              command: ["/bin/sh", "-c", "backup.sh"]
```

```bash
kubectl get cronjob
kubectl describe cronjob nightly-backup

# Manually trigger a CronJob immediately
kubectl create job --from=cronjob/nightly-backup manual-backup-$(date +%s)
```

---

## RBAC

Role-Based Access Control (RBAC) controls which users, groups, and service accounts can perform which operations on which resources.

### Core Resources

| Resource | Scope | Description |
|---|---|---|
| `Role` | Namespace | Grants permissions within a namespace |
| `ClusterRole` | Cluster | Grants permissions cluster-wide or across namespaces |
| `RoleBinding` | Namespace | Binds a Role or ClusterRole to subjects within a namespace |
| `ClusterRoleBinding` | Cluster | Binds a ClusterRole to subjects cluster-wide |

### Role & RoleBinding Example

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
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: production
subjects:
  - kind: User
    name: jane@example.com
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: ci-deployer
    namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-deployer
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/deployer   # IRSA on EKS
```

```bash
kubectl create serviceaccount ci-deployer -n production
kubectl get rolebindings -A
kubectl auth can-i create deployments --as=jane@example.com -n production
kubectl auth can-i --list --as=system:serviceaccount:production:ci-deployer
```

---

## HPA & Autoscaling

### Horizontal Pod Autoscaler (HPA)

HPA automatically scales the number of Pod replicas based on observed CPU/memory utilisation or custom metrics.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 20
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
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300    # Wait 5m before scaling down
      policies:
        - type: Percent
          value: 25
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 30
```

```bash
kubectl get hpa -A
kubectl describe hpa api-hpa
kubectl autoscale deployment api --min=2 --max=10 --cpu-percent=70
```

### Vertical Pod Autoscaler (VPA)

VPA recommends or automatically adjusts CPU and memory requests/limits based on historical usage. Requires VPA controller installation.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  updatePolicy:
    updateMode: "Auto"    # Or "Off" for recommendations only
```

### Cluster Autoscaler

Adjusts the number of nodes in a node group when Pods cannot be scheduled (scale up) or nodes are underutilised (scale down). Configured at the cluster level, not via manifests applied by users.

---

## Resource Management

### Requests & Limits

- **Request**: The amount of CPU/memory Kubernetes guarantees and uses for scheduling decisions.
- **Limit**: The maximum CPU/memory the container may use; exceeding the memory limit causes an OOM kill.

```yaml
resources:
  requests:
    cpu: "250m"      # 250 millicores = 0.25 CPU
    memory: "256Mi"
  limits:
    cpu: "1000m"     # 1 CPU
    memory: "512Mi"
```

### LimitRange

Sets default requests/limits and enforces min/max constraints for a namespace.

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
        memory: "64Mi"
      max:
        cpu: "4"
        memory: "4Gi"
      min:
        cpu: "50m"
        memory: "32Mi"
```

### ResourceQuota

Limits total resource consumption within a namespace.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
    services: "20"
    persistentvolumeclaims: "30"
    secrets: "50"
    configmaps: "50"
```

```bash
kubectl describe resourcequota -n production
kubectl describe limitrange -n production
kubectl top pods -n production --sort-by=memory
kubectl top nodes
```

---

## Probes & Health Checks

Kubernetes uses three probe types to manage container health. All probes support three mechanisms: `httpGet`, `tcpSocket`, and `exec`.

| Probe | Purpose | Effect if Failing |
|---|---|---|
| `livenessProbe` | Is the container still alive? | Container is killed and restarted |
| `readinessProbe` | Is the container ready to serve traffic? | Pod removed from Service endpoints |
| `startupProbe` | Has the application started? | Liveness/readiness probes disabled until success |

```yaml
containers:
  - name: api
    image: myorg/api:1.0
    # Startup probe: gives slow-starting apps time to initialise
    startupProbe:
      httpGet:
        path: /health/startup
        port: 3000
      failureThreshold: 30        # 30 * 10s = up to 5 minutes to start
      periodSeconds: 10

    # Liveness: restart if the app deadlocks
    livenessProbe:
      httpGet:
        path: /health/live
        port: 3000
      initialDelaySeconds: 0      # Start checking immediately after startupProbe
      periodSeconds: 15
      timeoutSeconds: 5
      failureThreshold: 3

    # Readiness: only route traffic when truly ready
    readinessProbe:
      httpGet:
        path: /health/ready
        port: 3000
      initialDelaySeconds: 5
      periodSeconds: 10
      timeoutSeconds: 3
      successThreshold: 1
      failureThreshold: 3
```

### TCP and Exec Probes

```yaml
# TCP socket probe (for non-HTTP services)
livenessProbe:
  tcpSocket:
    port: 5432
  periodSeconds: 10

# Exec probe (runs a command)
livenessProbe:
  exec:
    command:
      - redis-cli
      - ping
  periodSeconds: 10
```

---

## Namespaces

Namespaces partition cluster resources, enabling multi-tenancy, environment isolation (dev/staging/prod), and quota enforcement. Most resources are namespace-scoped; some (Nodes, PersistentVolumes, ClusterRoles) are cluster-scoped.

```bash
kubectl get namespaces
kubectl create namespace staging
kubectl apply -f app.yaml -n staging

# Set a default namespace for the current context
kubectl config set-context --current --namespace=production

# Delete a namespace and all resources within it
kubectl delete namespace staging
```

### Namespace Manifest

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    env: production
    team: platform
```

---

## Contexts & Config

A kubectl context combines a cluster, a user, and a namespace into a named entry. Multiple contexts let you switch between clusters with a single command.

### `kubectl config`

```bash
kubectl config view                             # View kubeconfig
kubectl config view --minify                    # View current context only
kubectl config get-contexts                     # List all contexts
kubectl config current-context                  # Show active context
kubectl config use-context prod-cluster         # Switch context
kubectl config set-context --current --namespace=production  # Set default namespace

# Add a cluster, user, and context
kubectl config set-cluster dev-cluster --server=https://dev-api.example.com
kubectl config set-credentials dev-admin --token=<token>
kubectl config set-context dev --cluster=dev-cluster --user=dev-admin
kubectl config delete-context old-cluster
```

### Merging kubeconfigs

```bash
# Merge multiple kubeconfig files
KUBECONFIG=~/.kube/config:~/.kube/prod.yaml kubectl config view --flatten > ~/.kube/merged.yaml
mv ~/.kube/merged.yaml ~/.kube/config
```

### Tools for Context Management

- **kubectx** — Fast context switching: `kubectx prod-cluster`
- **kubens** — Fast namespace switching: `kubens production`
- **kube-ps1** — Show current context/namespace in shell prompt

---

## Common Patterns & Tips

1. **Use labels and selectors consistently.** Labels are the primary mechanism for Services, HPAs, and selectors to find Pods. Adopt a standard set of labels across all workloads (`app`, `version`, `component`, `part-of`, `managed-by`) and enforce them via LimitRange or admission webhooks.

2. **Always define resource requests and limits.** Without requests, the scheduler cannot make informed placement decisions. Without limits, a single noisy container can starve neighbours. The `requests` value is the scheduling guarantee; the `limits` value is the hard cap.

3. **Use `kubectl diff` before applying changes in production.** `kubectl diff -f manifest.yaml` shows a human-readable diff between what is currently deployed and what you are about to apply, preventing accidental destructive changes.

4. **Use namespaces to isolate environments, not just teams.** Running `dev`, `staging`, and `production` in separate namespaces (or separate clusters for production) simplifies RBAC, quota management, and network policy, reducing the blast radius of misapplied changes.

5. **Use `kubectl rollout restart` to trigger a rolling restart.** If Pods need to reload a secret or configmap that has changed, `kubectl rollout restart deployment/<name>` performs a rolling restart without changing the Deployment spec, causing zero downtime.

6. **Set `terminationGracePeriodSeconds` appropriately.** The default grace period is 30 seconds. Long-running request handlers, database connections, or batch jobs may need more time to drain. Configure `preStop` lifecycle hooks to delay SIGTERM until the load balancer has drained the Pod.

   ```yaml
   lifecycle:
     preStop:
       exec:
         command: ["/bin/sh", "-c", "sleep 10"]
   terminationGracePeriodSeconds: 60
   ```

7. **Prefer `Deployment` over naked Pods.** Bare Pods are not rescheduled if a node fails. Always wrap Pods in a Deployment, StatefulSet, or DaemonSet so Kubernetes can self-heal them.

8. **Use `readinessProbe` to implement zero-downtime deployments.** A Deployment will not terminate old Pods during a rolling update until new Pods pass their readiness probe. A properly configured readiness check is the single most important thing for achieving real zero-downtime deployments.

9. **Store manifests in Git and use GitOps.** Tools like ArgoCD or Flux synchronise the cluster state from a Git repository, providing an audit trail, easy rollbacks, and drift detection. Never manually `kubectl edit` resources in production.

10. **Use `kubectl explain` to explore the API without leaving the terminal.** `kubectl explain deployment.spec.strategy.rollingUpdate` shows inline API documentation for any field, including defaults, types, and descriptions — far faster than browsing the web docs.

11. **Network policies default to allow-all.** Without NetworkPolicy resources, all Pods can communicate freely. Adopt a default-deny posture: create a NetworkPolicy that denies all ingress/egress in each namespace, then explicitly allow only necessary traffic.

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: default-deny-all
    spec:
      podSelector: {}
      policyTypes: [Ingress, Egress]
    ```

12. **Use `kubectl top` and metrics-server for quick capacity checks.** Install metrics-server in your cluster to enable `kubectl top pods` and `kubectl top nodes`. This gives immediate insight into CPU and memory usage without needing a full observability stack.
