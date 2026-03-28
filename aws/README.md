# AWS CLI Cheatsheet

The AWS Command Line Interface (CLI) is a unified tool that lets you manage AWS services directly from your terminal. It provides direct access to AWS service APIs, enabling automation, scripting, and faster workflows compared to the web console. The CLI supports all major AWS services and can be used interactively or within CI/CD pipelines. All commands follow the pattern `aws <service> <action> [options]`.

---

## Table of Contents

1. [Installation & Configuration](#installation--configuration)
2. [IAM](#iam)
3. [EC2](#ec2)
4. [S3](#s3)
5. [VPC & Networking](#vpc--networking)
6. [RDS](#rds)
7. [ECS & ECR](#ecs--ecr)
8. [EKS](#eks)
9. [Lambda](#lambda)
10. [CloudFormation](#cloudformation)
11. [CloudWatch](#cloudwatch)
12. [Secrets Manager](#secrets-manager)
13. [SSM Parameter Store](#ssm-parameter-store)
14. [Common Patterns & Tips](#common-patterns--tips)

---

## Installation & Configuration

### Install AWS CLI v2

Downloads and installs the AWS CLI v2 on Linux. The installer bundles all dependencies so no Python environment is required.

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

### `aws configure`

Interactively sets up your default credentials and region. Writes values to `~/.aws/credentials` and `~/.aws/config`.

**Key flags:**
- `--profile <name>` — write to a named profile instead of `[default]`

```bash
aws configure
# Prompts for: AWS Access Key ID, Secret Access Key, Default region, Output format

aws configure --profile staging
# Sets up a named profile for a staging account
```

### `aws configure list`

Displays the currently active configuration (source, key, value) without revealing secrets.

```bash
aws configure list
aws configure list --profile staging
```

### `aws configure set`

Programmatically sets a single configuration value — useful in scripts and CI pipelines.

**Key flags:**
- First positional arg — the key (e.g. `region`, `output`, `role_arn`)
- `--profile <name>` — target a specific profile

```bash
aws configure set region us-west-2
aws configure set output json --profile prod
aws configure set role_arn arn:aws:iam::123456789012:role/MyRole --profile cross-account
```

### `aws sts get-caller-identity`

Returns the AWS account ID, user/role ARN, and user ID of the currently authenticated principal. Essential for confirming you are operating in the correct account.

```bash
aws sts get-caller-identity
# Output: { "UserId": "...", "Account": "123456789012", "Arn": "arn:aws:iam::..." }
```

### `aws sts assume-role`

Assumes an IAM role and returns temporary credentials (access key, secret, session token).

**Key flags:**
- `--role-arn` — ARN of the role to assume
- `--role-session-name` — identifier for the session (required)
- `--duration-seconds` — lifetime of temp credentials (default 3600, max 43200)
- `--external-id` — required if the role trust policy demands it

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/DeployRole \
  --role-session-name deploy-session \
  --duration-seconds 3600
```

---

## IAM

### `aws iam list-users`

Lists all IAM users in the account.

**Key flags:**
- `--max-items <n>` — limit results
- `--path-prefix <prefix>` — filter by path (e.g. `/engineering/`)

```bash
aws iam list-users
aws iam list-users --path-prefix /engineering/ --output table
```

### `aws iam create-user`

Creates a new IAM user. Does not create access keys or console password — those are separate steps.

**Key flags:**
- `--user-name` — name of the new user (required)
- `--tags` — key-value metadata tags

```bash
aws iam create-user --user-name alice
aws iam create-user --user-name ci-bot --tags Key=Team,Value=Platform
```

### `aws iam create-access-key`

Generates a new programmatic access key pair for an IAM user. Returns the secret once — store it immediately.

**Key flags:**
- `--user-name` — the user to generate keys for

```bash
aws iam create-access-key --user-name alice
```

### `aws iam attach-user-policy`

Attaches a managed policy to an IAM user.

**Key flags:**
- `--user-name` — target user
- `--policy-arn` — ARN of the managed policy to attach

```bash
aws iam attach-user-policy \
  --user-name alice \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
```

### `aws iam create-role`

Creates an IAM role with a trust policy that specifies who can assume it.

**Key flags:**
- `--role-name` — name for the new role
- `--assume-role-policy-document` — JSON trust policy (file path or inline)
- `--description` — human-readable description

```bash
aws iam create-role \
  --role-name LambdaExecRole \
  --assume-role-policy-document file://trust-policy.json \
  --description "Role for Lambda execution"
```

### `aws iam list-attached-role-policies`

Shows all managed policies attached to a role.

```bash
aws iam list-attached-role-policies --role-name LambdaExecRole
```

### `aws iam get-role`

Retrieves details about a specific role including the trust policy.

```bash
aws iam get-role --role-name LambdaExecRole
```

---

## EC2

### `aws ec2 describe-instances`

Returns details about one or more EC2 instances. One of the most commonly used EC2 commands.

**Key flags:**
- `--instance-ids` — filter to specific instances
- `--filters` — filter by attribute (e.g. `Name=instance-state-name,Values=running`)
- `--query` — JMESPath expression to extract fields
- `--output` — `json`, `table`, or `text`

```bash
# List all running instances showing ID, type, state, and public IP
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].[InstanceId,InstanceType,State.Name,PublicIpAddress]' \
  --output table
```

### `aws ec2 run-instances`

Launches one or more new EC2 instances.

**Key flags:**
- `--image-id` — AMI ID (required)
- `--instance-type` — e.g. `t3.micro` (required)
- `--key-name` — name of the EC2 key pair for SSH
- `--security-group-ids` — one or more security group IDs
- `--subnet-id` — VPC subnet to launch in
- `--count` — number of instances to launch
- `--iam-instance-profile` — attach an instance profile
- `--user-data` — bootstrap script run at launch
- `--tag-specifications` — apply tags at launch time

```bash
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.micro \
  --key-name my-keypair \
  --security-group-ids sg-0123456789abcdef0 \
  --subnet-id subnet-0123456789abcdef0 \
  --count 1 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=web-server}]'
```

### `aws ec2 stop-instances` / `aws ec2 start-instances` / `aws ec2 terminate-instances`

Stops, starts, or permanently terminates EC2 instances.

```bash
aws ec2 stop-instances --instance-ids i-0123456789abcdef0
aws ec2 start-instances --instance-ids i-0123456789abcdef0
aws ec2 terminate-instances --instance-ids i-0123456789abcdef0
```

### `aws ec2 describe-security-groups`

Lists security groups and their inbound/outbound rules.

**Key flags:**
- `--group-ids` — filter by security group IDs
- `--filters` — e.g. `Name=vpc-id,Values=vpc-xxx`

```bash
aws ec2 describe-security-groups \
  --filters "Name=vpc-id,Values=vpc-0123456789abcdef0" \
  --query 'SecurityGroups[*].[GroupId,GroupName,Description]' \
  --output table
```

### `aws ec2 authorize-security-group-ingress`

Adds an inbound rule to a security group.

**Key flags:**
- `--group-id` — security group to modify
- `--protocol` — `tcp`, `udp`, `icmp`, or `-1` (all)
- `--port` — port number or range
- `--cidr` — source IP CIDR

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-0123456789abcdef0 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0
```

### `aws ec2 describe-key-pairs` / `aws ec2 create-key-pair`

Lists existing key pairs or creates a new one.

```bash
aws ec2 describe-key-pairs --output table

aws ec2 create-key-pair \
  --key-name my-new-keypair \
  --query 'KeyMaterial' \
  --output text > my-new-keypair.pem
chmod 400 my-new-keypair.pem
```

### `aws ec2 describe-images`

Searches the AMI catalog. Use filters to narrow results.

```bash
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
            "Name=state,Values=available" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text
```

---

## S3

### `aws s3 ls`

Lists S3 buckets (no args) or objects within a bucket prefix.

**Key flags:**
- `--recursive` — list all objects including nested prefixes
- `--human-readable` — show sizes in human-friendly format
- `--summarize` — show total object count and size

```bash
aws s3 ls                                      # list all buckets
aws s3 ls s3://my-bucket/                      # list top-level objects
aws s3 ls s3://my-bucket/logs/ --recursive --human-readable --summarize
```

### `aws s3 cp`

Copies files between local filesystem and S3, or between S3 locations.

**Key flags:**
- `--recursive` — copy a directory tree
- `--exclude` / `--include` — glob-based filters
- `--sse` — server-side encryption (`AES256` or `aws:kms`)
- `--storage-class` — `STANDARD`, `INTELLIGENT_TIERING`, `GLACIER`, etc.
- `--acl` — canned ACL (e.g. `private`, `public-read`)

```bash
aws s3 cp ./dist/ s3://my-bucket/releases/v1.2.3/ --recursive
aws s3 cp s3://my-bucket/backup.tar.gz ./backup.tar.gz
aws s3 cp report.pdf s3://my-bucket/reports/ --sse AES256 --storage-class INTELLIGENT_TIERING
```

### `aws s3 sync`

Syncs a local directory to/from S3, copying only new or changed files.

**Key flags:**
- `--delete` — remove destination files that don't exist in source
- `--exclude` / `--include` — glob filters
- `--dryrun` — show what would be transferred without doing it

```bash
aws s3 sync ./website/ s3://my-bucket/ --delete --exclude "*.DS_Store"
aws s3 sync s3://my-bucket/backups/ ./local-backups/ --dryrun
```

### `aws s3 rm`

Removes objects from S3.

**Key flags:**
- `--recursive` — delete all objects under a prefix
- `--exclude` / `--include` — glob filters

```bash
aws s3 rm s3://my-bucket/old-logs/2022/ --recursive
aws s3 rm s3://my-bucket/temp.txt
```

### `aws s3api create-bucket`

Creates a new S3 bucket with full control over options.

**Key flags:**
- `--bucket` — bucket name (globally unique)
- `--region` — AWS region
- `--create-bucket-configuration` — required for non-us-east-1 regions

```bash
aws s3api create-bucket \
  --bucket my-unique-bucket-name \
  --region eu-west-1 \
  --create-bucket-configuration LocationConstraint=eu-west-1
```

### `aws s3api put-bucket-versioning`

Enables versioning on a bucket so that overwritten objects are preserved.

```bash
aws s3api put-bucket-versioning \
  --bucket my-unique-bucket-name \
  --versioning-configuration Status=Enabled
```

### `aws s3api put-public-access-block`

Blocks all public access to a bucket — best practice for sensitive data.

```bash
aws s3api put-public-access-block \
  --bucket my-unique-bucket-name \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

---

## VPC & Networking

### `aws ec2 describe-vpcs`

Lists all VPCs in the current region with their CIDR blocks and state.

```bash
aws ec2 describe-vpcs \
  --query 'Vpcs[*].[VpcId,CidrBlock,State,Tags[?Key==`Name`].Value|[0]]' \
  --output table
```

### `aws ec2 create-vpc`

Creates a new VPC with a specified IPv4 CIDR block.

**Key flags:**
- `--cidr-block` — IPv4 CIDR for the VPC (required)
- `--enable-dns-support` / `--enable-dns-hostnames` — enable DNS resolution

```bash
aws ec2 create-vpc --cidr-block 10.0.0.0/16
```

### `aws ec2 create-subnet`

Creates a subnet inside a VPC.

**Key flags:**
- `--vpc-id` — VPC to create the subnet in
- `--cidr-block` — subnet CIDR (must be within VPC CIDR)
- `--availability-zone` — target AZ

```bash
aws ec2 create-subnet \
  --vpc-id vpc-0123456789abcdef0 \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a
```

### `aws ec2 describe-route-tables`

Lists route tables and their routes. Useful for diagnosing routing issues.

```bash
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=vpc-0123456789abcdef0" \
  --output table
```

### `aws ec2 describe-nat-gateways`

Lists NAT gateways and their status. Use to confirm private subnet outbound connectivity.

```bash
aws ec2 describe-nat-gateways \
  --filter "Name=state,Values=available" \
  --output table
```

---

## RDS

### `aws rds describe-db-instances`

Lists all RDS database instances with their endpoint, engine, and status.

```bash
aws rds describe-db-instances \
  --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceClass,Engine,DBInstanceStatus,Endpoint.Address]' \
  --output table
```

### `aws rds create-db-instance`

Provisions a new RDS database instance.

**Key flags:**
- `--db-instance-identifier` — unique name for the instance
- `--db-instance-class` — e.g. `db.t3.micro`
- `--engine` — `mysql`, `postgres`, `mariadb`, `oracle-se2`, `sqlserver-ex`
- `--master-username` / `--master-user-password` — admin credentials
- `--allocated-storage` — storage size in GiB
- `--vpc-security-group-ids` — security groups to associate
- `--db-subnet-group-name` — subnet group for VPC placement
- `--multi-az` — enable Multi-AZ for high availability
- `--storage-encrypted` — encrypt storage at rest

```bash
aws rds create-db-instance \
  --db-instance-identifier mydb \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --master-username admin \
  --master-user-password SuperSecret123! \
  --allocated-storage 20 \
  --vpc-security-group-ids sg-0123456789abcdef0 \
  --db-subnet-group-name my-subnet-group \
  --storage-encrypted
```

### `aws rds create-db-snapshot`

Creates a manual snapshot of an RDS instance for backup or migration.

```bash
aws rds create-db-snapshot \
  --db-instance-identifier mydb \
  --db-snapshot-identifier mydb-snapshot-20240101
```

### `aws rds describe-db-snapshots`

Lists RDS snapshots for a given instance.

```bash
aws rds describe-db-snapshots \
  --db-instance-identifier mydb \
  --query 'DBSnapshots[*].[DBSnapshotIdentifier,Status,SnapshotCreateTime]' \
  --output table
```

---

## ECS & ECR

### `aws ecr get-login-password`

Retrieves an ECR authentication token for use with Docker login.

```bash
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin \
    123456789012.dkr.ecr.us-east-1.amazonaws.com
```

### `aws ecr create-repository`

Creates a new ECR container image repository.

**Key flags:**
- `--repository-name` — name of the repository
- `--image-scanning-configuration` — enable scan-on-push
- `--encryption-configuration` — use KMS encryption

```bash
aws ecr create-repository \
  --repository-name my-app \
  --image-scanning-configuration scanOnPush=true
```

### `aws ecr describe-images`

Lists images in a repository with their tags and digest.

```bash
aws ecr describe-images \
  --repository-name my-app \
  --query 'imageDetails[*].[imageTags[0],imagePushedAt,imageSizeInBytes]' \
  --output table
```

### `aws ecs list-clusters` / `aws ecs describe-clusters`

Lists or describes ECS clusters.

```bash
aws ecs list-clusters
aws ecs describe-clusters --clusters my-cluster
```

### `aws ecs list-services`

Lists services running in an ECS cluster.

```bash
aws ecs list-services --cluster my-cluster
```

### `aws ecs update-service`

Updates a service — commonly used to force a new deployment or scale the task count.

**Key flags:**
- `--cluster` — target cluster
- `--service` — service name
- `--desired-count` — number of tasks
- `--force-new-deployment` — replace running tasks with fresh ones

```bash
aws ecs update-service \
  --cluster my-cluster \
  --service my-service \
  --desired-count 3 \
  --force-new-deployment
```

---

## EKS

### `aws eks list-clusters`

Lists all EKS clusters in the current region.

```bash
aws eks list-clusters --output table
```

### `aws eks create-cluster`

Creates a new EKS cluster (control plane only).

**Key flags:**
- `--name` — cluster name
- `--role-arn` — IAM role ARN for the cluster
- `--resources-vpc-config` — subnet IDs and security group IDs
- `--kubernetes-version` — e.g. `1.29`

```bash
aws eks create-cluster \
  --name prod-cluster \
  --role-arn arn:aws:iam::123456789012:role/EKSClusterRole \
  --resources-vpc-config subnetIds=subnet-abc,subnet-def,securityGroupIds=sg-xyz \
  --kubernetes-version 1.29
```

### `aws eks update-kubeconfig`

Updates (or creates) the local `~/.kube/config` to grant `kubectl` access to the cluster.

**Key flags:**
- `--name` — cluster name (required)
- `--region` — cluster region
- `--role-arn` — assume a role when authenticating

```bash
aws eks update-kubeconfig --name prod-cluster --region us-east-1
kubectl get nodes  # verify connectivity
```

### `aws eks create-nodegroup`

Adds a managed node group to an EKS cluster.

**Key flags:**
- `--cluster-name` — target cluster
- `--nodegroup-name` — name for the node group
- `--node-role` — IAM role ARN for nodes
- `--subnets` — subnet IDs for nodes
- `--instance-types` — EC2 instance types
- `--scaling-config` — min/max/desired node counts

```bash
aws eks create-nodegroup \
  --cluster-name prod-cluster \
  --nodegroup-name workers \
  --node-role arn:aws:iam::123456789012:role/EKSNodeRole \
  --subnets subnet-abc subnet-def \
  --instance-types t3.medium \
  --scaling-config minSize=2,maxSize=10,desiredSize=3
```

---

## Lambda

### `aws lambda list-functions`

Lists all Lambda functions in the region.

```bash
aws lambda list-functions \
  --query 'Functions[*].[FunctionName,Runtime,LastModified]' \
  --output table
```

### `aws lambda create-function`

Deploys a new Lambda function from a ZIP file.

**Key flags:**
- `--function-name` — name for the function
- `--runtime` — e.g. `python3.12`, `nodejs20.x`, `java21`
- `--role` — execution role ARN
- `--handler` — entry point (e.g. `index.handler`)
- `--zip-file` — path to deployment package
- `--timeout` — max execution time in seconds (default 3, max 900)
- `--memory-size` — MB of memory (128–10240)
- `--environment` — environment variables

```bash
aws lambda create-function \
  --function-name my-function \
  --runtime python3.12 \
  --role arn:aws:iam::123456789012:role/LambdaExecRole \
  --handler app.handler \
  --zip-file fileb://function.zip \
  --timeout 30 \
  --memory-size 256 \
  --environment Variables={ENV=production,LOG_LEVEL=info}
```

### `aws lambda update-function-code`

Deploys updated code to an existing Lambda function.

**Key flags:**
- `--function-name` — target function
- `--zip-file` — new deployment package
- `--image-uri` — container image URI (for image-based functions)
- `--publish` — publish a new version after updating

```bash
aws lambda update-function-code \
  --function-name my-function \
  --zip-file fileb://function.zip \
  --publish
```

### `aws lambda invoke`

Synchronously invokes a Lambda function and captures the response.

**Key flags:**
- `--function-name` — function to invoke
- `--payload` — JSON input event
- `--log-type Tail` — include last 4KB of logs in response

```bash
aws lambda invoke \
  --function-name my-function \
  --payload '{"key": "value"}' \
  --log-type Tail \
  output.json
cat output.json
```

---

## CloudFormation

### `aws cloudformation deploy`

High-level command that creates or updates a stack using change sets. Preferred for CI/CD.

**Key flags:**
- `--template-file` — local path to the CloudFormation template
- `--stack-name` — name of the stack
- `--parameter-overrides` — key=value pairs to override template parameters
- `--capabilities` — `CAPABILITY_IAM`, `CAPABILITY_NAMED_IAM`, `CAPABILITY_AUTO_EXPAND`
- `--no-fail-on-empty-changeset` — succeed even if no changes are needed

```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name my-app-stack \
  --parameter-overrides Environment=production ImageTag=v1.2.3 \
  --capabilities CAPABILITY_NAMED_IAM \
  --no-fail-on-empty-changeset
```

### `aws cloudformation describe-stacks`

Returns details about one or all stacks including status and outputs.

```bash
aws cloudformation describe-stacks --stack-name my-app-stack \
  --query 'Stacks[0].Outputs'
```

### `aws cloudformation describe-stack-events`

Lists events for a stack — essential for diagnosing failed deployments.

```bash
aws cloudformation describe-stack-events \
  --stack-name my-app-stack \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`].[LogicalResourceId,ResourceStatusReason]' \
  --output table
```

### `aws cloudformation delete-stack`

Deletes a CloudFormation stack and all its managed resources.

```bash
aws cloudformation delete-stack --stack-name my-app-stack
aws cloudformation wait stack-delete-complete --stack-name my-app-stack
```

---

## CloudWatch

### `aws cloudwatch describe-alarms`

Lists CloudWatch alarms and their current state.

**Key flags:**
- `--state-value` — filter by `OK`, `ALARM`, or `INSUFFICIENT_DATA`
- `--alarm-names` — filter to specific alarms

```bash
aws cloudwatch describe-alarms \
  --state-value ALARM \
  --query 'MetricAlarms[*].[AlarmName,StateValue,MetricName]' \
  --output table
```

### `aws cloudwatch put-metric-alarm`

Creates or updates a CloudWatch alarm.

**Key flags:**
- `--alarm-name` — unique name
- `--metric-name` — metric to monitor
- `--namespace` — metric namespace (e.g. `AWS/EC2`)
- `--statistic` — `Average`, `Sum`, `Maximum`, etc.
- `--period` — evaluation period in seconds
- `--evaluation-periods` — number of periods to evaluate
- `--threshold` — value to compare against
- `--comparison-operator` — e.g. `GreaterThanThreshold`
- `--alarm-actions` — SNS ARN to notify when in ALARM state

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name high-cpu \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --dimensions Name=InstanceId,Value=i-0123456789abcdef0 \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:alerts
```

### `aws logs filter-log-events`

Searches CloudWatch Logs for matching log events across a log group.

**Key flags:**
- `--log-group-name` — target log group
- `--filter-pattern` — CloudWatch Logs Insights filter syntax
- `--start-time` / `--end-time` — time range in milliseconds since epoch
- `--log-stream-name-prefix` — narrow to specific streams

```bash
aws logs filter-log-events \
  --log-group-name /aws/lambda/my-function \
  --filter-pattern "ERROR" \
  --start-time $(date -d '1 hour ago' +%s000)
```

### `aws logs tail`

Streams new log events from a log group in real time (similar to `tail -f`).

**Key flags:**
- `--follow` — continuously poll for new events
- `--format` — `short`, `medium`, `long`, `detailed`
- `--filter-pattern` — only show matching events

```bash
aws logs tail /aws/lambda/my-function --follow --format short --filter-pattern ERROR
```

---

## Secrets Manager

### `aws secretsmanager create-secret`

Creates a new secret. Can store string or binary values.

**Key flags:**
- `--name` — secret name or path (e.g. `/prod/db/password`)
- `--secret-string` — the secret value (string or JSON)
- `--kms-key-id` — KMS key for encryption (default uses AWS-managed key)
- `--description` — human-readable description

```bash
aws secretsmanager create-secret \
  --name /prod/db/password \
  --description "Production database password" \
  --secret-string '{"username":"admin","password":"SuperSecret123!"}'
```

### `aws secretsmanager get-secret-value`

Retrieves a secret's value. The most commonly used Secrets Manager command.

**Key flags:**
- `--secret-id` — name or ARN of the secret
- `--version-stage` — retrieve a specific version stage (default `AWSCURRENT`)

```bash
aws secretsmanager get-secret-value --secret-id /prod/db/password
# Extract just the value:
aws secretsmanager get-secret-value --secret-id /prod/db/password \
  --query SecretString --output text | jq -r '.password'
```

### `aws secretsmanager update-secret`

Updates the value of an existing secret, automatically rotating the version.

```bash
aws secretsmanager update-secret \
  --secret-id /prod/db/password \
  --secret-string '{"username":"admin","password":"NewPassword456!"}'
```

### `aws secretsmanager list-secrets`

Lists all secrets in the account with their names and ARNs.

```bash
aws secretsmanager list-secrets \
  --query 'SecretList[*].[Name,ARN,LastChangedDate]' \
  --output table
```

---

## SSM Parameter Store

### `aws ssm put-parameter`

Creates or updates a parameter in SSM Parameter Store.

**Key flags:**
- `--name` — parameter name/path (e.g. `/myapp/prod/db_host`)
- `--value` — the parameter value
- `--type` — `String`, `StringList`, or `SecureString`
- `--key-id` — KMS key for `SecureString` encryption
- `--overwrite` — required to update an existing parameter

```bash
aws ssm put-parameter \
  --name /myapp/prod/db_host \
  --value "rds.example.com" \
  --type String

aws ssm put-parameter \
  --name /myapp/prod/db_password \
  --value "SuperSecret!" \
  --type SecureString \
  --overwrite
```

### `aws ssm get-parameter`

Retrieves a single parameter. Use `--with-decryption` for `SecureString` values.

**Key flags:**
- `--name` — parameter name
- `--with-decryption` — decrypt SecureString values

```bash
aws ssm get-parameter --name /myapp/prod/db_host --query Parameter.Value --output text
aws ssm get-parameter --name /myapp/prod/db_password --with-decryption --query Parameter.Value --output text
```

### `aws ssm get-parameters-by-path`

Retrieves all parameters under a given path prefix — ideal for loading a full application config.

**Key flags:**
- `--path` — the path prefix
- `--recursive` — include parameters in sub-paths
- `--with-decryption` — decrypt SecureString values

```bash
aws ssm get-parameters-by-path \
  --path /myapp/prod/ \
  --recursive \
  --with-decryption \
  --query 'Parameters[*].[Name,Value]' \
  --output table
```

### `aws ssm delete-parameter`

Deletes a parameter from the store.

```bash
aws ssm delete-parameter --name /myapp/prod/old-setting
```

---

## Common Patterns & Tips

1. **Use `--output table` for human reading, `--output json` for scripting.**
   The default JSON output is great for `jq` pipelines, but `--output table` is much easier to scan in the terminal. Use `--output text` when extracting a single scalar value.

2. **Master `--query` with JMESPath to avoid `grep`/`awk`.**
   JMESPath lets you reshape API responses in-place. Example: extract only running instance IDs:
   ```bash
   aws ec2 describe-instances \
     --filters Name=instance-state-name,Values=running \
     --query 'Reservations[*].Instances[*].InstanceId' \
     --output text
   ```

3. **Use named profiles to manage multiple AWS accounts safely.**
   ```bash
   export AWS_PROFILE=staging
   aws s3 ls  # operates on staging account
   ```
   Or pass `--profile` per-command to avoid accidental cross-account actions.

4. **Use `aws configure set` in CI/CD instead of writing credential files.**
   In GitHub Actions or similar, prefer environment variables:
   ```bash
   export AWS_ACCESS_KEY_ID=${{ secrets.AWS_ACCESS_KEY_ID }}
   export AWS_SECRET_ACCESS_KEY=${{ secrets.AWS_SECRET_ACCESS_KEY }}
   export AWS_DEFAULT_REGION=us-east-1
   ```

5. **Leverage `--dry-run` for EC2 to validate permissions without side effects.**
   ```bash
   aws ec2 run-instances --dry-run --image-id ami-xxx --instance-type t3.micro ...
   # Returns error code 412 if allowed, permission error if not
   ```

6. **Use `aws s3 sync --dryrun` before large syncs to preview changes.**
   This prevents accidental deletions especially when `--delete` is involved.

7. **Chain `aws cloudformation wait` after deployments to block until completion.**
   ```bash
   aws cloudformation deploy --stack-name my-stack ...
   aws cloudformation wait stack-update-complete --stack-name my-stack
   ```
   Available waiters: `stack-create-complete`, `stack-update-complete`, `stack-delete-complete`.

8. **Store sensitive values in Secrets Manager, non-sensitive config in SSM Parameter Store.**
   Use hierarchical paths like `/app/env/key` for easy bulk retrieval via `get-parameters-by-path`. This pattern works well with ECS task definitions and Lambda environment injection.

9. **Tag all resources at creation time with `--tags` or `--tag-specifications`.**
   Tags enable cost allocation, automated cleanup scripts, and IAM condition-based access control. A minimal tagging standard might be `Environment`, `Team`, `Project`, and `ManagedBy`.

10. **Use `aws logs tail --follow` instead of clicking through the console during debugging.**
    It supports `--filter-pattern` so you can stream only `ERROR` or `WARN` entries in real time, making Lambda and ECS debugging significantly faster.
