# DevOps Cheatsheet Collection

> A curated, production-ready reference library covering the essential DevOps tools and platforms — each with real-world examples, YAML manifests, and common scenarios.

---

## Cheatsheets

| Cheatsheet | Topics Covered |
|---|---|
| [Git](./git-cheatsheet.md) | Setup, branching, merging, rebasing, stashing, cherry-pick, bisect, worktrees |
| [Docker](./docker-cheatsheet.md) | Images, containers, volumes, networking, Dockerfile, Docker Compose, registry |
| [Kubernetes](./kubernetes-cheatsheet.md) | Pods, Deployments, Services, Ingress, StatefulSets, RBAC, HPA, Helm |
| [AWS CLI](./aws-cheatsheet.md) | IAM, EC2, S3, VPC, RDS, ECS/ECR, EKS, Lambda, CloudFormation, CloudWatch |
| [Google Cloud](./gcloud-cheatsheet.md) | Compute, GCS, GKE, Cloud Run, Cloud Build, Artifact Registry, Secret Manager |
| [Azure CLI](./azure-cheatsheet.md) | VMs, Storage, VNet, AKS, ACR, App Service, Key Vault, Monitor |
| [Azure DevOps](./azure-devops-cheatsheet.md) | Pipelines, Repos, Boards, Artifacts, Variable Groups, Environments, Agents |
| [Linux & Bash](./linux-cheatsheet.md) | Files, permissions, users, processes, systemd, cron, text processing, scripting |
| [Networking](./networking-cheatsheet.md) | DNS, curl, TLS/SSL, SSH, tcpdump, firewalls, Nginx, WireGuard, performance |

---

## What's Inside Each Cheatsheet

### Git
- Repository setup, staging, committing, branching
- Merging vs rebasing, remote workflows
- Undoing changes, stashing, tagging
- Cherry-picking, submodules, bisect, worktrees
- **10 scenarios** — squashing commits, recovering deleted branches, splitting commits, interactive rebase cleanup

### Docker
- Image building (multi-stage), container lifecycle
- Networking, volumes, bind mounts
- Full `docker-compose.yml` with Postgres + Redis + Nginx
- Dockerfile best practices — non-root user, `.dockerignore`, layer caching
- **10 scenarios** — debugging crashes, hot-reload dev setup, secret injection, shrinking images with `scratch`

### Kubernetes
- Pod, Deployment, Service, Ingress, StatefulSet, DaemonSet, Job, CronJob
- ConfigMaps, Secrets, PersistentVolumes, StorageClasses
- RBAC, HPA, LimitRange, ResourceQuota, probes
- Complete YAML manifests for every major resource
- **10 scenarios** — zero-downtime rollouts, blue/green, canary, DB migrations via init containers, node drain

### AWS CLI
- IAM users, roles, policies, access key rotation
- EC2 instances, security groups, AMIs, Auto Scaling
- S3 sync, pre-signed URLs, static hosting, versioning
- VPC, subnets, NAT, Elastic IP, Route 53
- RDS, ECS/ECR, EKS, Lambda, CloudFormation, CloudWatch, Secrets Manager

### Google Cloud (gcloud)
- IAM, service accounts, workload identity
- Compute Engine, Cloud SQL, VPC, Cloud NAT
- GKE cluster management, Cloud Run deployments
- Artifact Registry, Cloud Build (`cloudbuild.yaml`)
- Pub/Sub, Secret Manager, Cloud Logging

### Azure CLI
- Resource groups, subscriptions, tagging
- IAM, RBAC, service principals, managed identities
- VMs, VM Scale Sets, NSGs, VNets, Load Balancers
- AKS, ACR, App Service deployment slots, Azure Functions
- Key Vault, Azure SQL, Cosmos DB, Monitor alerts

### Azure DevOps
- Repos — branch policies, pull requests, review enforcement
- Boards — work items, sprints, relations
- Full CI/CD YAML pipelines with stages, jobs, deployments
- Pipeline templates, matrix strategy, caching, artifacts
- Variable groups, Key Vault integration, Environments with approvals
- Self-hosted agents (bare metal + Docker)

### Linux & Bash
- Navigation, file permissions, users, groups
- Process management, signals, system info, disk & storage
- `systemd` services, custom unit files, `journalctl`
- Cron scheduling, archives, text processing (`grep`, `awk`, `sed`)
- Bash scripting — argument parsing, error handling, retry logic, arrays
- **7 scenarios** — log watching, watchdog script, parallel execution, safe `.env` loading

### Networking
- Interface config, routing, ARP — `ip`, `ss`, `netstat`
- DNS deep-dive — `dig`, `nslookup`, record types, propagation checks
- HTTP with `curl` — all methods, auth, timeouts, response timing
- TLS/SSL — cert inspection, expiry, CSR generation, format conversion
- SSH — tunneling, jump hosts, `~/.ssh/config`, `sshd` hardening
- Port scanning — `nmap`, `nc`, `lsof`
- Packet capture — `tcpdump` filters and analysis
- Firewalls — `ufw`, `iptables`, `nftables`
- Nginx — reverse proxy, SSL termination, WebSocket, load balancing
- WireGuard VPN, `iperf3`, `nethogs`, TCP tuning
- **7 scenarios** — debug 502, diagnose SSL issues, find bandwidth hogs, block brute-force SSH

---

## Quick Picks

**Just want to run a container?** → [Docker: Running Containers](./docker-cheatsheet.md#containers)

**Deploying to Kubernetes?** → [K8s: Deployments](./kubernetes-cheatsheet.md#deployments) · [Helm](./kubernetes-cheatsheet.md#helm)

**Setting up a CI/CD pipeline?** → [Azure DevOps: Multi-Stage Pipelines](./azure-devops-cheatsheet.md#multi-stage-pipelines)

**Managing cloud infrastructure?** → [AWS](./aws-cheatsheet.md) · [GCP](./gcloud-cheatsheet.md) · [Azure](./azure-cheatsheet.md)

**Git went wrong?** → [Git: Undoing Changes](./git-cheatsheet.md#undoing-changes) · [Reflog Recovery](./git-cheatsheet.md#advanced-scenarios)

**Debugging network issues?** → [Networking: Advanced Scenarios](./networking-cheatsheet.md#advanced-scenarios)

**Writing a deploy script?** → [Linux: Bash Scripting](./linux-cheatsheet.md#bash-scripting)

**SSL cert expired?** → [Networking: TLS/SSL](./networking-cheatsheet.md#tls--ssl)

---

## Structure

```
devops-cheatsheet/
├── README.md
├── git-cheatsheet.md
├── docker-cheatsheet.md
├── kubernetes-cheatsheet.md
├── aws-cheatsheet.md
├── gcloud-cheatsheet.md
├── azure-cheatsheet.md
├── azure-devops-cheatsheet.md
├── linux-cheatsheet.md
└── networking-cheatsheet.md
```

---

> Each cheatsheet follows the same pattern: commands with context, annotated YAML, reference tables, and real-world scenarios.
