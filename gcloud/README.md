# Google Cloud CLI (gcloud) Cheatsheet

The `gcloud` command-line tool is the primary CLI for interacting with Google Cloud Platform services. It is bundled as part of the **Google Cloud SDK** and provides commands for provisioning resources, managing IAM, deploying applications, and much more. Most commands follow the pattern `gcloud <group> <command> [flags]`. Alongside `gcloud`, the SDK also includes `gsutil` for Cloud Storage and `bq` for BigQuery.

---

## Table of Contents

1. [Installation & Auth](#installation--auth)
2. [IAM & Service Accounts](#iam--service-accounts)
3. [Compute Engine](#compute-engine)
4. [Cloud Storage (gsutil)](#cloud-storage-gsutil)
5. [GKE](#gke)
6. [Cloud Run](#cloud-run)
7. [Cloud Build](#cloud-build)
8. [Artifact Registry](#artifact-registry)
9. [Cloud SQL](#cloud-sql)
10. [Pub/Sub](#pubsub)
11. [Secret Manager](#secret-manager)
12. [Cloud Logging & Monitoring](#cloud-logging--monitoring)
13. [Common Patterns & Tips](#common-patterns--tips)

---

## Installation & Auth

### Install Google Cloud SDK

Downloads and runs the official SDK installer, which installs `gcloud`, `gsutil`, `bq`, and `kubectl` (optional).

```bash
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
gcloud init
```

For a non-interactive install (CI/CD):
```bash
curl -sSL https://sdk.cloud.google.com | bash -s -- --disable-prompts
export PATH="$HOME/google-cloud-sdk/bin:$PATH"
```

### `gcloud init`

Runs the interactive setup wizard — authenticates your user, sets a default project, and optionally sets a default compute region/zone.

```bash
gcloud init
```

### `gcloud auth login`

Opens a browser-based OAuth2 flow to authenticate as your Google account. The token is stored locally.

**Key flags:**
- `--no-launch-browser` — prints a URL to open manually (useful for headless environments)
- `--update-adc` — also updates Application Default Credentials

```bash
gcloud auth login
gcloud auth login --no-launch-browser
```

### `gcloud auth application-default login`

Sets up **Application Default Credentials (ADC)** used by client libraries and tools that call Google APIs. Separate from user login credentials.

```bash
gcloud auth application-default login
```

### `gcloud auth activate-service-account`

Authenticates using a service account key file — the standard approach for CI/CD pipelines.

**Key flags:**
- `--key-file` — path to the service account JSON key file (required)

```bash
gcloud auth activate-service-account \
  --key-file=/path/to/sa-key.json
```

### `gcloud config set` / `gcloud config get`

Sets or gets a named configuration property. The most important are `project`, `compute/region`, and `compute/zone`.

```bash
gcloud config set project my-project-id
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a
gcloud config get project
```

### `gcloud config configurations`

Manages named configurations — each holds a different set of defaults (project, account, region). Useful when switching between projects/environments.

```bash
gcloud config configurations create staging
gcloud config configurations activate staging
gcloud config configurations list
```

### `gcloud info`

Displays the SDK installation path, active account, project, and active configuration — great for diagnosing environment issues.

```bash
gcloud info
```

---

## IAM & Service Accounts

### `gcloud projects get-iam-policy`

Retrieves the full IAM policy for a project, showing all bindings (member → role).

**Key flags:**
- `--format` — e.g. `json`, `yaml`, `table(...)`

```bash
gcloud projects get-iam-policy my-project-id
gcloud projects get-iam-policy my-project-id --format=json > policy.json
```

### `gcloud projects add-iam-policy-binding`

Grants a role to a member on a project. Uses an additive binding (does not replace existing bindings).

**Key flags:**
- `--member` — identity in the form `user:`, `serviceAccount:`, `group:`, or `domain:`
- `--role` — IAM role name (e.g. `roles/storage.objectViewer`)
- `--condition` — optional condition expression for attribute-based access

```bash
gcloud projects add-iam-policy-binding my-project-id \
  --member="serviceAccount:deploy-bot@my-project-id.iam.gserviceaccount.com" \
  --role="roles/run.admin"
```

### `gcloud projects remove-iam-policy-binding`

Removes a specific role binding from a project.

```bash
gcloud projects remove-iam-policy-binding my-project-id \
  --member="user:alice@example.com" \
  --role="roles/editor"
```

### `gcloud iam service-accounts create`

Creates a new service account. Service accounts are the recommended way to authenticate workloads.

**Key flags:**
- `--display-name` — human-friendly name
- `--description` — longer description of purpose

```bash
gcloud iam service-accounts create deploy-bot \
  --display-name="Deployment Bot" \
  --description="Used by CI pipeline to deploy to Cloud Run"
```

### `gcloud iam service-accounts list`

Lists all service accounts in the active project.

```bash
gcloud iam service-accounts list --format="table(email,displayName,disabled)"
```

### `gcloud iam service-accounts keys create`

Creates and downloads a JSON key file for a service account. Treat this file as a secret.

**Key flags:**
- `--iam-account` — service account email
- positional arg — output file path

```bash
gcloud iam service-accounts keys create sa-key.json \
  --iam-account=deploy-bot@my-project-id.iam.gserviceaccount.com
```

### `gcloud iam roles create`

Creates a custom IAM role with a specific set of permissions.

**Key flags:**
- `--project` — scope to a project (vs. organization)
- `--permissions` — comma-separated list of permissions
- `--title` — display title
- `--stage` — `ALPHA`, `BETA`, or `GA`

```bash
gcloud iam roles create customDeployer \
  --project=my-project-id \
  --permissions=run.services.update,run.services.get \
  --title="Custom Deployer" \
  --stage=GA
```

---

## Compute Engine

### `gcloud compute instances list`

Lists all VM instances in the project with their zone, status, and external IP.

**Key flags:**
- `--filter` — server-side filter expression (e.g. `status=RUNNING`)
- `--format` — output format
- `--zones` — limit to specific zones

```bash
gcloud compute instances list
gcloud compute instances list --filter="status=RUNNING" --format="table(name,zone,status,networkInterfaces[0].accessConfigs[0].natIP)"
```

### `gcloud compute instances create`

Creates one or more new VM instances.

**Key flags:**
- `--zone` — GCP zone (e.g. `us-central1-a`)
- `--machine-type` — e.g. `e2-medium`, `n2-standard-4`
- `--image-family` / `--image-project` — source image
- `--boot-disk-size` — size of boot disk (e.g. `50GB`)
- `--tags` — network tags for firewall rules
- `--metadata` / `--metadata-from-file` — instance metadata (startup scripts)
- `--service-account` — service account email to attach
- `--scopes` — OAuth scopes
- `--no-address` — launch without external IP (private-only)

```bash
gcloud compute instances create web-server-1 \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --boot-disk-size=20GB \
  --tags=http-server,https-server \
  --service-account=my-sa@my-project-id.iam.gserviceaccount.com \
  --scopes=cloud-platform
```

### `gcloud compute instances start` / `stop` / `delete`

Starts, stops, or deletes VM instances.

```bash
gcloud compute instances start web-server-1 --zone=us-central1-a
gcloud compute instances stop web-server-1 --zone=us-central1-a
gcloud compute instances delete web-server-1 --zone=us-central1-a --quiet
```

### `gcloud compute ssh`

Opens an SSH connection to a VM, creating an ephemeral key pair if needed.

**Key flags:**
- `--zone` — required if not set as default
- `--tunnel-through-iap` — use Identity-Aware Proxy (no public IP needed)
- `--command` — run a single command and exit

```bash
gcloud compute ssh web-server-1 --zone=us-central1-a
gcloud compute ssh web-server-1 --zone=us-central1-a --tunnel-through-iap
gcloud compute ssh web-server-1 --zone=us-central1-a --command="uptime"
```

### `gcloud compute firewall-rules create`

Creates a VPC firewall rule to allow or deny traffic.

**Key flags:**
- `--network` — target VPC network
- `--action` — `ALLOW` or `DENY`
- `--rules` — protocol:port (e.g. `tcp:443`)
- `--source-ranges` — CIDR source ranges
- `--target-tags` — apply to instances with these network tags

```bash
gcloud compute firewall-rules create allow-https \
  --network=default \
  --action=ALLOW \
  --rules=tcp:443 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=https-server
```

---

## Cloud Storage (gsutil)

### `gsutil ls`

Lists buckets (no args) or objects within a bucket.

**Key flags:**
- `-r` — recursive listing
- `-l` — long listing with size and time
- `-h` — human-readable sizes

```bash
gsutil ls                          # list all buckets
gsutil ls gs://my-bucket/          # list objects in bucket
gsutil ls -r -l -h gs://my-bucket/ # recursive long listing
```

### `gsutil cp`

Copies files to/from/within Cloud Storage.

**Key flags:**
- `-r` — recursive (for directories)
- `-m` — parallel multi-threaded transfer (much faster for many files)
- `-z <ext>` — gzip-compress files with these extensions during upload
- `-a <acl>` — set predefined ACL (e.g. `public-read`)

```bash
gsutil cp ./dist/app.tar.gz gs://my-bucket/releases/
gsutil -m cp -r ./website/ gs://my-bucket/
gsutil cp gs://my-bucket/backup.sql.gz ./backup.sql.gz
```

### `gsutil rsync`

Synchronizes a local directory tree with a GCS bucket prefix, copying only changed files.

**Key flags:**
- `-r` — recursive (required for directories)
- `-d` — delete destination objects not in source
- `-m` — parallel transfers
- `-x <pattern>` — exclude paths matching regex

```bash
gsutil -m rsync -r -d ./website/ gs://my-bucket/
gsutil -m rsync -r -x "\.git/|node_modules/" ./project/ gs://my-bucket/project/
```

### `gsutil rm`

Deletes objects or buckets from Cloud Storage.

**Key flags:**
- `-r` — recursive (delete all objects and the bucket)
- `-m` — parallel deletion

```bash
gsutil rm gs://my-bucket/old-file.txt
gsutil -m rm -r gs://my-bucket/old-prefix/
```

### `gsutil mb` (make bucket)

Creates a new Cloud Storage bucket.

**Key flags:**
- `-l <location>` — bucket location (e.g. `US`, `EU`, `us-central1`)
- `-c <class>` — storage class (`STANDARD`, `NEARLINE`, `COLDLINE`, `ARCHIVE`)
- `-b on` — enable uniform bucket-level access (recommended)

```bash
gsutil mb -l us-central1 -c STANDARD -b on gs://my-new-bucket/
```

### `gsutil iam ch`

Grants or revokes IAM bindings on a bucket.

```bash
# Grant objectViewer to a service account
gsutil iam ch \
  serviceAccount:my-sa@my-project.iam.gserviceaccount.com:objectViewer \
  gs://my-bucket/

# Remove a binding
gsutil iam ch -d \
  serviceAccount:my-sa@my-project.iam.gserviceaccount.com:objectViewer \
  gs://my-bucket/
```

---

## GKE

### `gcloud container clusters create`

Creates a new GKE cluster.

**Key flags:**
- `--zone` / `--region` — use `--region` for regional (multi-zone) clusters
- `--num-nodes` — nodes per zone
- `--machine-type` — e.g. `e2-standard-4`
- `--enable-autoscaling` + `--min-nodes` / `--max-nodes` — cluster autoscaler
- `--enable-autorepair` / `--enable-autoupgrade` — node maintenance
- `--workload-pool` — enable Workload Identity (recommended)
- `--release-channel` — `rapid`, `regular`, `stable`

```bash
gcloud container clusters create prod-cluster \
  --region=us-central1 \
  --num-nodes=3 \
  --machine-type=e2-standard-4 \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=10 \
  --enable-autorepair \
  --enable-autoupgrade \
  --workload-pool=my-project-id.svc.id.goog \
  --release-channel=regular
```

### `gcloud container clusters get-credentials`

Configures `kubectl` to connect to the specified cluster.

**Key flags:**
- `--region` / `--zone` — location of the cluster
- `--internal-ip` — use internal IP (for private clusters accessed via VPN/IAP)

```bash
gcloud container clusters get-credentials prod-cluster --region=us-central1
kubectl get nodes
```

### `gcloud container clusters list`

Lists all GKE clusters in the project.

```bash
gcloud container clusters list --format="table(name,location,status,currentNodeCount)"
```

### `gcloud container node-pools create`

Adds a new node pool to an existing cluster.

**Key flags:**
- `--cluster` — target cluster name
- `--num-nodes` / `--min-nodes` / `--max-nodes` — size and autoscaling
- `--machine-type` — node machine type
- `--node-taints` — apply Kubernetes taints to all nodes in the pool

```bash
gcloud container node-pools create gpu-pool \
  --cluster=prod-cluster \
  --region=us-central1 \
  --machine-type=n1-standard-4 \
  --accelerator=type=nvidia-tesla-t4,count=1 \
  --num-nodes=1 \
  --enable-autoscaling \
  --min-nodes=0 \
  --max-nodes=4
```

---

## Cloud Run

### `gcloud run deploy`

Deploys a container image to Cloud Run, creating or updating a service.

**Key flags:**
- `--image` — container image URI (required)
- `--region` — deployment region
- `--platform` — `managed` (default), `gke`, or `kubernetes`
- `--allow-unauthenticated` — make the service publicly accessible
- `--no-allow-unauthenticated` — require authentication
- `--set-env-vars` — environment variables
- `--set-secrets` — mount secrets from Secret Manager
- `--memory` — memory limit (e.g. `512Mi`, `2Gi`)
- `--cpu` — CPU allocation
- `--min-instances` / `--max-instances` — concurrency bounds
- `--port` — port the container listens on

```bash
gcloud run deploy my-service \
  --image=us-central1-docker.pkg.dev/my-project/my-repo/my-app:v1.0 \
  --region=us-central1 \
  --allow-unauthenticated \
  --memory=512Mi \
  --set-env-vars=ENV=production,LOG_LEVEL=info \
  --set-secrets=DB_PASSWORD=my-db-secret:latest \
  --min-instances=1 \
  --max-instances=10
```

### `gcloud run services list`

Lists all Cloud Run services in the project.

```bash
gcloud run services list --region=us-central1 \
  --format="table(metadata.name,status.url,status.conditions[0].status)"
```

### `gcloud run services describe`

Shows detailed information about a Cloud Run service including its URL, environment, and traffic splits.

```bash
gcloud run services describe my-service --region=us-central1
```

### `gcloud run services update-traffic`

Splits traffic between revisions — used for canary releases and gradual rollouts.

**Key flags:**
- `--to-revisions` — revision=percentage pairs
- `--to-latest` — send 100% traffic to the latest revision

```bash
# 90% to stable, 10% to canary
gcloud run services update-traffic my-service \
  --region=us-central1 \
  --to-revisions=my-service-00010-abc=90,my-service-00011-xyz=10

# Full cutover
gcloud run services update-traffic my-service \
  --region=us-central1 \
  --to-latest
```

---

## Cloud Build

### `gcloud builds submit`

Submits a build to Cloud Build. Can use a `cloudbuild.yaml` config or build a Dockerfile automatically.

**Key flags:**
- `--config` — path to `cloudbuild.yaml` (default is root `cloudbuild.yaml`)
- `--tag` — shorthand to build and push to a single image tag
- `--substitutions` — override substitution variables defined in the config
- `--no-source` — don't upload local source (use if source is already in GCS)
- `--gcs-log-dir` — store build logs in a GCS bucket

```bash
# Build using cloudbuild.yaml
gcloud builds submit --config=cloudbuild.yaml \
  --substitutions=_IMAGE_TAG=v1.2.3,_ENV=production

# Quick build and push (no cloudbuild.yaml needed)
gcloud builds submit --tag=us-central1-docker.pkg.dev/my-project/my-repo/my-app:latest .
```

### `gcloud builds list`

Lists recent builds with their status, trigger, and duration.

```bash
gcloud builds list --limit=10 \
  --format="table(id,status,createTime,duration,source.repoSource.branchName)"
```

### `gcloud builds log`

Streams the log output from a specific build.

```bash
gcloud builds log <BUILD_ID> --stream
```

---

## Artifact Registry

### `gcloud artifacts repositories create`

Creates a new Artifact Registry repository for storing container images or language packages.

**Key flags:**
- `--repository-format` — `docker`, `npm`, `maven`, `python`, `apt`, `yum`
- `--location` — region (e.g. `us-central1`) or multi-region (`us`, `eu`)
- `--description` — human-readable description

```bash
gcloud artifacts repositories create my-repo \
  --repository-format=docker \
  --location=us-central1 \
  --description="Production container images"
```

### `gcloud artifacts repositories list`

Lists all Artifact Registry repositories in the project.

```bash
gcloud artifacts repositories list --location=us-central1
```

### `gcloud auth configure-docker`

Configures Docker credential helpers for Artifact Registry so `docker push`/`pull` work seamlessly.

```bash
gcloud auth configure-docker us-central1-docker.pkg.dev
```

### `gcloud artifacts docker images list`

Lists container images in an Artifact Registry Docker repository.

**Key flags:**
- `--include-tags` — show image tags alongside digests

```bash
gcloud artifacts docker images list \
  us-central1-docker.pkg.dev/my-project/my-repo \
  --include-tags
```

---

## Cloud SQL

### `gcloud sql instances create`

Provisions a new Cloud SQL instance (MySQL, PostgreSQL, or SQL Server).

**Key flags:**
- `--database-version` — e.g. `POSTGRES_15`, `MYSQL_8_0`
- `--tier` — machine type (e.g. `db-f1-micro`, `db-n1-standard-2`)
- `--region` — deployment region
- `--availability-type` — `REGIONAL` (HA) or `ZONAL`
- `--storage-size` — initial storage in GB
- `--storage-auto-increase` — automatically grow storage
- `--no-backup` — disable automated backups (not recommended for prod)

```bash
gcloud sql instances create my-db \
  --database-version=POSTGRES_15 \
  --tier=db-n1-standard-2 \
  --region=us-central1 \
  --availability-type=REGIONAL \
  --storage-size=50 \
  --storage-auto-increase
```

### `gcloud sql instances list`

Lists all Cloud SQL instances in the project.

```bash
gcloud sql instances list \
  --format="table(name,databaseVersion,region,status,settings.tier)"
```

### `gcloud sql databases create`

Creates a database within a Cloud SQL instance.

```bash
gcloud sql databases create my-app-db --instance=my-db
```

### `gcloud sql users create`

Creates a database user on a Cloud SQL instance.

```bash
gcloud sql users create app-user \
  --instance=my-db \
  --password=SecurePassword123!
```

### `gcloud sql connect`

Opens an interactive SQL session to a Cloud SQL instance via the Cloud SQL Proxy.

```bash
gcloud sql connect my-db --user=app-user --database=my-app-db
```

---

## Pub/Sub

### `gcloud pubsub topics create`

Creates a new Pub/Sub topic for publishing messages.

```bash
gcloud pubsub topics create my-topic
```

### `gcloud pubsub subscriptions create`

Creates a subscription to pull or push messages from a topic.

**Key flags:**
- `--topic` — the topic to subscribe to (required)
- `--push-endpoint` — URL for push delivery (omit for pull)
- `--ack-deadline` — seconds to ack before redelivery (10–600)
- `--message-retention-duration` — how long undelivered messages are retained

```bash
# Pull subscription
gcloud pubsub subscriptions create my-sub --topic=my-topic --ack-deadline=60

# Push subscription
gcloud pubsub subscriptions create my-push-sub \
  --topic=my-topic \
  --push-endpoint=https://my-service.example.com/pubsub/push
```

### `gcloud pubsub topics publish`

Publishes a message to a topic — useful for testing.

**Key flags:**
- `--message` — message body

```bash
gcloud pubsub topics publish my-topic --message='{"event": "user.created", "id": "123"}'
```

### `gcloud pubsub subscriptions pull`

Pulls pending messages from a subscription.

**Key flags:**
- `--limit` — max number of messages to return
- `--auto-ack` — acknowledge messages after pulling

```bash
gcloud pubsub subscriptions pull my-sub --limit=5 --auto-ack
```

---

## Secret Manager

### `gcloud secrets create`

Creates a new secret. The secret value is stored as a version.

**Key flags:**
- `--data-file` — file containing the secret value (`-` reads from stdin)
- `--replication-policy` — `automatic` or `user-managed`
- `--labels` — key-value metadata

```bash
echo -n "my-super-secret-value" | gcloud secrets create my-secret --data-file=-

gcloud secrets create db-password \
  --data-file=./db-password.txt \
  --replication-policy=automatic
```

### `gcloud secrets versions access`

Retrieves the value of a secret version. The most frequently used Secret Manager command.

**Key flags:**
- positional arg — version number or `latest`

```bash
gcloud secrets versions access latest --secret=my-secret

# Decode if needed and use in a script
DB_PASS=$(gcloud secrets versions access latest --secret=db-password)
```

### `gcloud secrets versions add`

Adds a new version to an existing secret (rotates the value).

```bash
echo -n "new-secret-value" | gcloud secrets versions add my-secret --data-file=-
```

### `gcloud secrets list`

Lists all secrets in the project.

```bash
gcloud secrets list --format="table(name,createTime,replication.automatic)"
```

---

## Cloud Logging & Monitoring

### `gcloud logging read`

Queries Cloud Logging using a filter expression. Powerful for searching logs across all services.

**Key flags:**
- first positional arg — log filter (uses Logging query language)
- `--limit` — max number of log entries to return
- `--freshness` — only return logs from the last N duration (e.g. `1h`, `30m`)
- `--format` — output format (`json`, `text`, `table`)
- `--order` — `asc` or `desc`

```bash
# Recent errors from Cloud Run
gcloud logging read \
  'resource.type="cloud_run_revision" severity>=ERROR' \
  --limit=50 \
  --freshness=1h \
  --format=json

# Logs for a specific function
gcloud logging read \
  'resource.type="cloud_function" resource.labels.function_name="my-func"' \
  --limit=20
```

### `gcloud logging logs list`

Lists all log names available in the project.

```bash
gcloud logging logs list --limit=20
```

### `gcloud monitoring dashboards create`

Creates a Cloud Monitoring dashboard from a JSON definition.

```bash
gcloud monitoring dashboards create --config-from-file=dashboard.json
```

### `gcloud monitoring channel-descriptors list`

Lists available notification channel types (email, Slack, PagerDuty, etc.).

```bash
gcloud monitoring channel-descriptors list \
  --format="table(type,displayName)"
```

### `gcloud alpha monitoring policies create`

Creates an alerting policy from a JSON/YAML definition.

```bash
gcloud alpha monitoring policies create --policy-from-file=alert-policy.json
```

---

## Common Patterns & Tips

1. **Always set a default project and region to avoid repeating `--project` and `--region` flags.**
   ```bash
   gcloud config set project my-project-id
   gcloud config set compute/region us-central1
   ```
   Or use the `CLOUDSDK_CORE_PROJECT` and `CLOUDSDK_COMPUTE_REGION` environment variables in CI.

2. **Use named configurations to switch between environments without re-running `gcloud init`.**
   ```bash
   gcloud config configurations create prod
   gcloud config set project my-prod-project --configuration=prod
   gcloud config configurations activate prod
   ```

3. **Prefer Workload Identity over service account key files for GKE workloads.**
   Key files are long-lived credentials that must be rotated manually. Workload Identity maps a Kubernetes service account to a GCP service account, eliminating the need for key files entirely:
   ```bash
   gcloud iam service-accounts add-iam-policy-binding my-sa@project.iam.gserviceaccount.com \
     --role=roles/iam.workloadIdentityUser \
     --member="serviceAccount:project.svc.id.goog[namespace/ksa-name]"
   ```

4. **Use `--format="value(field)"` for clean single-value extraction in scripts.**
   ```bash
   IMAGE_URI=$(gcloud run services describe my-service \
     --region=us-central1 \
     --format="value(spec.template.spec.containers[0].image)")
   ```

5. **Use `--filter` for server-side filtering to reduce output size and improve speed.**
   ```bash
   gcloud compute instances list --filter="status=RUNNING AND zone:us-central1"
   ```
   `--filter` is evaluated server-side (unlike `grep`), so it reduces API response size.

6. **Use `gsutil -m` for parallel transfers when copying many files.**
   The `-m` flag enables multi-threaded and multi-process operations, dramatically improving throughput when uploading or downloading hundreds of files.

7. **Authenticate Cloud Build by granting it the necessary IAM roles rather than using key files.**
   The Cloud Build service account (`<project-number>@cloudbuild.gserviceaccount.com`) can be granted specific roles:
   ```bash
   gcloud projects add-iam-policy-binding my-project-id \
     --member="serviceAccount:$(gcloud projects describe my-project-id --format='value(projectNumber)')@cloudbuild.gserviceaccount.com" \
     --role="roles/run.admin"
   ```

8. **Tag Cloud Run revisions with `--tag` to get unique preview URLs per branch.**
   ```bash
   gcloud run deploy my-service \
     --image=... \
     --tag=feature-branch-name \
     --no-traffic  # don't shift production traffic yet
   # Access at: https://feature-branch-name---my-service-xxx.run.app
   ```

9. **Use `gcloud secrets versions access latest` in startup scripts to avoid hardcoded credentials.**
   Applications running on GCE, GKE, or Cloud Run can fetch secrets at startup without storing them in environment variables or config files, using the instance's attached service account for authentication.

10. **Enable `--verbosity=debug` to diagnose authentication and API errors.**
    ```bash
    gcloud --verbosity=debug run deploy my-service ...
    ```
    This prints the raw HTTP requests and responses, which is invaluable when debugging IAM permission errors or network issues.
