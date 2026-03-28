# Google Cloud (gcloud) Cheatsheet

> A comprehensive reference for the Google Cloud CLI — from compute and storage to GKE, Cloud Run, and beyond.

---

## Table of Contents

- [Setup & Configuration](#setup--configuration)
- [IAM & Service Accounts](#iam--service-accounts)
- [Compute Engine](#compute-engine)
- [Cloud Storage (GCS)](#cloud-storage-gcs)
- [VPC & Networking](#vpc--networking)
- [Cloud SQL](#cloud-sql)
- [GKE](#gke)
- [Cloud Run](#cloud-run)
- [Artifact Registry & Container Registry](#artifact-registry--container-registry)
- [Cloud Functions](#cloud-functions)
- [Cloud Build](#cloud-build)
- [Pub/Sub](#pubsub)
- [Cloud Logging & Monitoring](#cloud-logging--monitoring)
- [Secret Manager](#secret-manager)
- [Advanced Scenarios](#advanced-scenarios)

---

## Setup & Configuration

```bash
# Install gcloud SDK
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
gcloud init

# Check version
gcloud version

# Login
gcloud auth login

# Application Default Credentials (for SDKs/apps)
gcloud auth application-default login

# List authenticated accounts
gcloud auth list

# Set active account
gcloud config set account user@example.com

# Set default project
gcloud config set project my-project-id

# Set default region and zone
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a

# List all config
gcloud config list

# Use named configurations (like AWS profiles)
gcloud config configurations create production
gcloud config configurations activate production
gcloud config configurations list

# Get current project
gcloud config get-value project

# List all projects
gcloud projects list
```

---

## IAM & Service Accounts

```bash
# List IAM policy for a project
gcloud projects get-iam-policy my-project-id

# Add IAM binding (grant a role)
gcloud projects add-iam-policy-binding my-project-id \
  --member="user:jane@example.com" \
  --role="roles/editor"

# Remove IAM binding
gcloud projects remove-iam-policy-binding my-project-id \
  --member="user:jane@example.com" \
  --role="roles/editor"

# List predefined roles
gcloud iam roles list

# Describe a role
gcloud iam roles describe roles/container.admin

# Create a service account
gcloud iam service-accounts create my-sa \
  --display-name "My Service Account"

# List service accounts
gcloud iam service-accounts list

# Grant role to service account
gcloud projects add-iam-policy-binding my-project-id \
  --member="serviceAccount:my-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"

# Create and download a key file
gcloud iam service-accounts keys create key.json \
  --iam-account=my-sa@my-project.iam.gserviceaccount.com

# List keys for a service account
gcloud iam service-accounts keys list \
  --iam-account=my-sa@my-project.iam.gserviceaccount.com

# Impersonate a service account
gcloud config set auth/impersonate_service_account my-sa@my-project.iam.gserviceaccount.com
```

---

## Compute Engine

### Instances

```bash
# List instances
gcloud compute instances list

# Create an instance
gcloud compute instances create web-server \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --zone=us-central1-a \
  --tags=http-server,https-server \
  --boot-disk-size=20GB

# Create with startup script
gcloud compute instances create web-server \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --metadata-from-file startup-script=startup.sh

# SSH into instance
gcloud compute ssh web-server --zone=us-central1-a

# SSH and run a command
gcloud compute ssh web-server --command="sudo systemctl status nginx"

# SCP files to/from instance
gcloud compute scp ./app.tar.gz web-server:/tmp/ --zone=us-central1-a
gcloud compute scp web-server:/var/log/app.log ./ --zone=us-central1-a

# Start / stop / delete
gcloud compute instances start web-server --zone=us-central1-a
gcloud compute instances stop web-server --zone=us-central1-a
gcloud compute instances delete web-server --zone=us-central1-a

# Resize machine type
gcloud compute instances stop web-server --zone=us-central1-a
gcloud compute instances set-machine-type web-server \
  --machine-type=e2-standard-4 --zone=us-central1-a
gcloud compute instances start web-server --zone=us-central1-a

# List machine types
gcloud compute machine-types list --filter="zone:us-central1-a"
```

### Disks & Snapshots

```bash
# List disks
gcloud compute disks list

# Create a disk
gcloud compute disks create my-disk \
  --size=100GB --type=pd-ssd --zone=us-central1-a

# Attach disk
gcloud compute instances attach-disk web-server \
  --disk=my-disk --zone=us-central1-a

# Create snapshot
gcloud compute disks snapshot my-disk \
  --snapshot-names=my-disk-snap-$(date +%Y%m%d) \
  --zone=us-central1-a

# List snapshots
gcloud compute snapshots list
```

### Firewall Rules

```bash
# List firewall rules
gcloud compute firewall-rules list

# Allow HTTP/HTTPS
gcloud compute firewall-rules create allow-web \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:80,tcp:443 \
  --target-tags=http-server \
  --source-ranges=0.0.0.0/0

# Allow SSH from specific IP
gcloud compute firewall-rules create allow-ssh \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=MY.IP.ADD.RESS/32

# Delete a rule
gcloud compute firewall-rules delete allow-ssh
```

---

## Cloud Storage (GCS)

```bash
# List buckets
gcloud storage buckets list
gsutil ls                               # gsutil alternative

# Create bucket
gcloud storage buckets create gs://my-unique-bucket \
  --location=US \
  --uniform-bucket-level-access

# List objects
gcloud storage ls gs://my-bucket/
gcloud storage ls gs://my-bucket/ --recursive

# Upload
gcloud storage cp file.txt gs://my-bucket/
gcloud storage cp -r ./dist gs://my-bucket/app/

# Download
gcloud storage cp gs://my-bucket/file.txt ./

# Sync
gcloud storage rsync ./dist gs://my-bucket/app/ --delete-unmatched-destination-objects

# Delete object
gcloud storage rm gs://my-bucket/file.txt

# Delete all in folder
gcloud storage rm gs://my-bucket/folder/** --recursive

# Generate signed URL (1 hour)
gcloud storage sign-url gs://my-bucket/private.pdf \
  --duration=1h \
  --private-key-file=key.json \
  --service-account=my-sa@my-project.iam.gserviceaccount.com

# Set bucket IAM policy (make public)
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member=allUsers \
  --role=roles/storage.objectViewer

# Enable versioning
gcloud storage buckets update gs://my-bucket --versioning
```

---

## VPC & Networking

```bash
# List VPCs
gcloud compute networks list

# Create VPC
gcloud compute networks create my-vpc --subnet-mode=custom

# Create subnet
gcloud compute networks subnets create my-subnet \
  --network=my-vpc \
  --region=us-central1 \
  --range=10.0.0.0/24

# List subnets
gcloud compute networks subnets list

# Create Cloud NAT (outbound internet for private VMs)
gcloud compute routers create my-router \
  --network=my-vpc --region=us-central1

gcloud compute routers nats create my-nat \
  --router=my-router \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges \
  --region=us-central1

# Reserve static external IP
gcloud compute addresses create my-static-ip --region=us-central1

# List reserved IPs
gcloud compute addresses list
```

---

## Cloud SQL

```bash
# List instances
gcloud sql instances list

# Create PostgreSQL instance
gcloud sql instances create mydb \
  --database-version=POSTGRES_16 \
  --tier=db-f1-micro \
  --region=us-central1 \
  --no-assign-ip \
  --network=my-vpc

# Create MySQL instance
gcloud sql instances create mydb \
  --database-version=MYSQL_8_0 \
  --tier=db-f1-micro \
  --region=us-central1

# Set root password
gcloud sql users set-password root \
  --host=% \
  --instance=mydb \
  --password=secret123

# Create a database
gcloud sql databases create myappdb --instance=mydb

# Create a user
gcloud sql users create appuser \
  --instance=mydb \
  --password=apppass

# Connect to instance
gcloud sql connect mydb --user=root --database=myappdb

# Export to GCS
gcloud sql export sql mydb gs://my-bucket/backup.sql \
  --database=myappdb

# Import from GCS
gcloud sql import sql mydb gs://my-bucket/backup.sql \
  --database=myappdb

# Create a backup
gcloud sql backups create --instance=mydb

# List backups
gcloud sql backups list --instance=mydb
```

---

## GKE

```bash
# Create a cluster
gcloud container clusters create my-cluster \
  --zone=us-central1-a \
  --num-nodes=3 \
  --machine-type=e2-medium \
  --enable-autoscaling --min-nodes=2 --max-nodes=5

# Create Autopilot cluster (fully managed)
gcloud container clusters create-auto my-autopilot \
  --region=us-central1

# Get credentials for kubectl
gcloud container clusters get-credentials my-cluster \
  --zone=us-central1-a

# List clusters
gcloud container clusters list

# Describe cluster
gcloud container clusters describe my-cluster --zone=us-central1-a

# Upgrade cluster
gcloud container clusters upgrade my-cluster \
  --master --cluster-version=1.29 --zone=us-central1-a

# Resize node pool
gcloud container clusters resize my-cluster \
  --node-pool=default-pool \
  --num-nodes=5 \
  --zone=us-central1-a

# Add a node pool
gcloud container node-pools create high-mem-pool \
  --cluster=my-cluster \
  --machine-type=n2-highmem-4 \
  --num-nodes=2 \
  --zone=us-central1-a

# Delete cluster
gcloud container clusters delete my-cluster --zone=us-central1-a
```

---

## Cloud Run

```bash
# Deploy from container image
gcloud run deploy myapp \
  --image=us-central1-docker.pkg.dev/my-project/my-repo/myapp:latest \
  --platform=managed \
  --region=us-central1 \
  --allow-unauthenticated \
  --port=3000 \
  --memory=512Mi \
  --cpu=1 \
  --min-instances=1 \
  --max-instances=10

# Deploy from source (builds automatically)
gcloud run deploy myapp --source . --region=us-central1

# List services
gcloud run services list --region=us-central1

# Describe service
gcloud run services describe myapp --region=us-central1

# Get service URL
gcloud run services describe myapp \
  --region=us-central1 \
  --format='value(status.url)'

# Update env variables
gcloud run services update myapp \
  --update-env-vars NODE_ENV=production,PORT=3000 \
  --region=us-central1

# Update secrets as env vars
gcloud run services update myapp \
  --update-secrets DB_PASSWORD=my-secret:latest \
  --region=us-central1

# View traffic split (canary)
gcloud run services describe myapp --region=us-central1

# Split traffic between revisions
gcloud run services update-traffic myapp \
  --to-revisions=myapp-v1=90,myapp-v2=10 \
  --region=us-central1

# View logs
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=myapp" --limit=50

# Delete service
gcloud run services delete myapp --region=us-central1
```

---

## Artifact Registry & Container Registry

```bash
# Create a repository
gcloud artifacts repositories create my-repo \
  --repository-format=docker \
  --location=us-central1 \
  --description="Docker images"

# List repositories
gcloud artifacts repositories list

# Configure Docker auth
gcloud auth configure-docker us-central1-docker.pkg.dev

# Tag and push image
docker tag myapp:latest us-central1-docker.pkg.dev/my-project/my-repo/myapp:latest
docker push us-central1-docker.pkg.dev/my-project/my-repo/myapp:latest

# List images
gcloud artifacts docker images list \
  us-central1-docker.pkg.dev/my-project/my-repo

# Delete image
gcloud artifacts docker images delete \
  us-central1-docker.pkg.dev/my-project/my-repo/myapp:old-tag
```

---

## Cloud Functions

```bash
# Deploy a function (Gen 2)
gcloud functions deploy my-function \
  --gen2 \
  --runtime=nodejs20 \
  --region=us-central1 \
  --source=. \
  --entry-point=handler \
  --trigger-http \
  --allow-unauthenticated \
  --memory=256MB \
  --timeout=60s

# Deploy with Pub/Sub trigger
gcloud functions deploy my-function \
  --gen2 \
  --runtime=python312 \
  --region=us-central1 \
  --source=. \
  --entry-point=process_message \
  --trigger-topic=my-topic

# List functions
gcloud functions list

# Invoke a function
gcloud functions call my-function \
  --region=us-central1 \
  --data='{"name":"world"}'

# View logs
gcloud functions logs read my-function --region=us-central1

# Delete function
gcloud functions delete my-function --region=us-central1
```

---

## Cloud Build

```bash
# Submit a build (uses cloudbuild.yaml)
gcloud builds submit --config=cloudbuild.yaml .

# Submit with substitutions
gcloud builds submit \
  --config=cloudbuild.yaml \
  --substitutions=_ENV=production,_IMAGE_TAG=v1.2.0 .

# List builds
gcloud builds list

# Stream a build's logs
gcloud builds log <BUILD_ID> --stream

# Cancel a build
gcloud builds cancel <BUILD_ID>
```

### `cloudbuild.yaml`

```yaml
steps:
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/myapp:$COMMIT_SHA', '.']

  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/myapp:$COMMIT_SHA']

  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: gcloud
    args:
      - run
      - deploy
      - myapp
      - --image=us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/myapp:$COMMIT_SHA
      - --region=us-central1
      - --platform=managed

images:
  - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/myapp:$COMMIT_SHA'
```

---

## Pub/Sub

```bash
# Create a topic
gcloud pubsub topics create my-topic

# Create a subscription
gcloud pubsub subscriptions create my-sub --topic=my-topic

# Publish a message
gcloud pubsub topics publish my-topic --message="hello world"

# Pull messages
gcloud pubsub subscriptions pull my-sub --auto-ack --limit=10

# List topics
gcloud pubsub topics list

# List subscriptions
gcloud pubsub subscriptions list

# Delete topic
gcloud pubsub topics delete my-topic
```

---

## Cloud Logging & Monitoring

```bash
# Read logs (most recent 50)
gcloud logging read "resource.type=gce_instance" --limit=50

# Read logs for a specific project with filter
gcloud logging read 'severity>=ERROR AND timestamp>="2024-01-01T00:00:00Z"' \
  --limit=100 --format=json

# Stream logs in real time
gcloud alpha logging tail 'resource.type=cloud_run_revision'

# Write a log entry
gcloud logging write my-log "Application started" --severity=INFO

# List log sinks
gcloud logging sinks list

# Create a sink (export logs to GCS)
gcloud logging sinks create my-sink \
  storage.googleapis.com/my-log-bucket \
  --log-filter='severity>=WARNING'

# List metrics
gcloud monitoring metrics list \
  --filter="metric.type:compute.googleapis.com"

# Create uptime check
gcloud monitoring uptime create \
  --display-name="API Health" \
  --http-check-path="/health" \
  --monitored-resource-type=uptime_url \
  --hostname=api.example.com
```

---

## Secret Manager

```bash
# Create a secret
echo -n "my-secret-value" | \
  gcloud secrets create my-secret --data-file=-

# Create from file
gcloud secrets create db-password --data-file=./password.txt

# Add a new version
echo -n "new-value" | \
  gcloud secrets versions add my-secret --data-file=-

# Access latest version
gcloud secrets versions access latest --secret=my-secret

# Access a specific version
gcloud secrets versions access 3 --secret=my-secret

# List secrets
gcloud secrets list

# List versions of a secret
gcloud secrets versions list my-secret

# Disable a version
gcloud secrets versions disable 2 --secret=my-secret

# Delete a secret
gcloud secrets delete my-secret
```

---

## Advanced Scenarios

### Scenario 1 — Deploy to Cloud Run on every git push (Cloud Build trigger)

```bash
gcloud builds triggers create github \
  --repo-name=my-repo \
  --repo-owner=my-org \
  --branch-pattern="^main$" \
  --build-config=cloudbuild.yaml
```

### Scenario 2 — Rotate service account keys automatically

```bash
# Delete old key, create new one, update app
OLD_KEY=$(gcloud iam service-accounts keys list \
  --iam-account=my-sa@my-project.iam.gserviceaccount.com \
  --format="value(name)" | tail -1)

gcloud iam service-accounts keys create new-key.json \
  --iam-account=my-sa@my-project.iam.gserviceaccount.com

gcloud iam service-accounts keys delete $OLD_KEY \
  --iam-account=my-sa@my-project.iam.gserviceaccount.com
```

### Scenario 3 — Access Cloud SQL from Cloud Run via private IP

```bash
gcloud run services update myapp \
  --add-cloudsql-instances=my-project:us-central1:mydb \
  --update-env-vars DB_HOST=/cloudsql/my-project:us-central1:mydb \
  --region=us-central1
```

### Scenario 4 — List all resources consuming budget (cost audit)

```bash
gcloud billing budgets list \
  --billing-account=BILLING_ACCOUNT_ID

# Check costs per service
gcloud billing accounts describe BILLING_ACCOUNT_ID
```

### Scenario 5 — GKE workload identity (no key files needed)

```bash
# Bind Kubernetes SA to Google SA
gcloud iam service-accounts add-iam-policy-binding \
  my-sa@my-project.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="serviceAccount:my-project.svc.id.goog[default/my-k8s-sa]"

kubectl annotate serviceaccount my-k8s-sa \
  iam.gke.io/gcp-service-account=my-sa@my-project.iam.gserviceaccount.com
```

---

## Quick Reference Card

| Task                          | Command                                                   |
|-------------------------------|-----------------------------------------------------------|
| Set project                   | `gcloud config set project my-project`                    |
| List instances                | `gcloud compute instances list`                           |
| SSH to VM                     | `gcloud compute ssh vm-name --zone=us-central1-a`         |
| GCS sync                      | `gcloud storage rsync ./dist gs://bucket/ --delete-unmatched-destination-objects` |
| GKE credentials               | `gcloud container clusters get-credentials cluster-name`  |
| Cloud Run deploy              | `gcloud run deploy app --image=IMAGE --region=us-central1`|
| Get secret                    | `gcloud secrets versions access latest --secret=name`     |
| Tail logs                     | `gcloud alpha logging tail 'resource.type=cloud_run_revision'` |
| Build & deploy                | `gcloud builds submit --config=cloudbuild.yaml .`         |

---

> Use service accounts with workload identity — never bake key files into containers.
