# AWS CLI Cheatsheet

> A comprehensive reference for the AWS CLI — from IAM and compute to storage, networking, and containers.

---

## Table of Contents

- [Setup & Configuration](#setup--configuration)
- [IAM](#iam)
- [EC2](#ec2)
- [S3](#s3)
- [VPC & Networking](#vpc--networking)
- [RDS](#rds)
- [ECS & ECR](#ecs--ecr)
- [EKS](#eks)
- [Lambda](#lambda)
- [CloudFormation](#cloudformation)
- [CloudWatch](#cloudwatch)
- [Route 53](#route-53)
- [Secrets Manager & SSM Parameter Store](#secrets-manager--ssm-parameter-store)
- [Advanced Scenarios](#advanced-scenarios)

---

## Setup & Configuration

```bash
# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip && sudo ./aws/install

# Check version
aws --version

# Configure default profile
aws configure
# AWS Access Key ID:     AKIA...
# AWS Secret Access Key: ****
# Default region:        us-east-1
# Default output format: json

# Configure a named profile
aws configure --profile production

# List all profiles
aws configure list-profiles

# Use a named profile for a command
aws s3 ls --profile production

# Set profile for session
export AWS_PROFILE=production

# View current config
aws configure list

# Get current caller identity (who am I?)
aws sts get-caller-identity
```

---

## IAM

```bash
# List users
aws iam list-users

# Create a user
aws iam create-user --user-name jane

# Create access keys for a user
aws iam create-access-key --user-name jane

# List access keys
aws iam list-access-keys --user-name jane

# Delete access key
aws iam delete-access-key --user-name jane --access-key-id AKIA...

# Attach a managed policy to user
aws iam attach-user-policy \
  --user-name jane \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Detach a managed policy
aws iam detach-user-policy \
  --user-name jane \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# List attached user policies
aws iam list-attached-user-policies --user-name jane

# Create a group and add user
aws iam create-group --group-name Developers
aws iam add-user-to-group --user-name jane --group-name Developers

# Create a role with trust policy
aws iam create-role \
  --role-name MyLambdaRole \
  --assume-role-policy-document file://trust-policy.json

# Attach policy to role
aws iam attach-role-policy \
  --role-name MyLambdaRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Get role ARN
aws iam get-role --role-name MyLambdaRole --query 'Role.Arn' --output text

# Assume a role (get temporary credentials)
aws sts assume-role \
  --role-arn arn:aws:iam::123456789:role/MyRole \
  --role-session-name MySession

# List all roles
aws iam list-roles --query 'Roles[*].RoleName' --output table
```

---

## EC2

### Instances

```bash
# List running instances
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].[InstanceId,InstanceType,PublicIpAddress,Tags[?Key==`Name`].Value|[0]]' \
  --output table

# Launch an instance
aws ec2 run-instances \
  --image-id ami-0c02fb55956c7d316 \
  --instance-type t3.micro \
  --key-name my-keypair \
  --security-group-ids sg-0abc123 \
  --subnet-id subnet-0abc123 \
  --count 1 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=web-server}]'

# Stop / start / terminate
aws ec2 stop-instances --instance-ids i-0abc123
aws ec2 start-instances --instance-ids i-0abc123
aws ec2 terminate-instances --instance-ids i-0abc123

# Reboot
aws ec2 reboot-instances --instance-ids i-0abc123

# Get instance public IP
aws ec2 describe-instances \
  --instance-ids i-0abc123 \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text
```

### Key Pairs & Security Groups

```bash
# Create and save a key pair
aws ec2 create-key-pair \
  --key-name my-keypair \
  --query 'KeyMaterial' \
  --output text > my-keypair.pem
chmod 400 my-keypair.pem

# List key pairs
aws ec2 describe-key-pairs

# Create security group
aws ec2 create-security-group \
  --group-name web-sg \
  --description "Web server SG" \
  --vpc-id vpc-0abc123

# Add inbound rule
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc123 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Add SSH rule
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc123 \
  --protocol tcp \
  --port 22 \
  --cidr MY.IP.ADD.RESS/32

# List security group rules
aws ec2 describe-security-groups --group-ids sg-0abc123
```

### AMIs & Snapshots

```bash
# List your AMIs
aws ec2 describe-images --owners self

# Create AMI from instance
aws ec2 create-image \
  --instance-id i-0abc123 \
  --name "my-app-ami-$(date +%Y%m%d)" \
  --no-reboot

# Create snapshot of a volume
aws ec2 create-snapshot \
  --volume-id vol-0abc123 \
  --description "backup $(date +%Y-%m-%d)"

# List snapshots
aws ec2 describe-snapshots --owner-ids self
```

### Auto Scaling

```bash
# Create launch template
aws ec2 create-launch-template \
  --launch-template-name my-lt \
  --launch-template-data '{"ImageId":"ami-0abc123","InstanceType":"t3.micro"}'

# Create Auto Scaling group
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name my-asg \
  --launch-template LaunchTemplateName=my-lt,Version='$Latest' \
  --min-size 2 --max-size 10 --desired-capacity 3 \
  --vpc-zone-identifier "subnet-0abc,subnet-1def"

# Update desired capacity
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name my-asg \
  --desired-capacity 5

# Describe ASG
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names my-asg
```

---

## S3

```bash
# List buckets
aws s3 ls

# List objects in bucket
aws s3 ls s3://my-bucket/
aws s3 ls s3://my-bucket/prefix/ --recursive

# Create bucket
aws s3 mb s3://my-unique-bucket --region us-east-1

# Upload a file
aws s3 cp file.txt s3://my-bucket/
aws s3 cp file.txt s3://my-bucket/folder/file.txt

# Upload a directory
aws s3 cp ./dist s3://my-bucket/app/ --recursive

# Download a file
aws s3 cp s3://my-bucket/file.txt ./file.txt

# Download a directory
aws s3 cp s3://my-bucket/app/ ./app --recursive

# Sync (only upload changed files — like rsync)
aws s3 sync ./dist s3://my-bucket/app/
aws s3 sync s3://my-bucket/app/ ./app

# Sync and delete files removed locally
aws s3 sync ./dist s3://my-bucket/app/ --delete

# Delete a file
aws s3 rm s3://my-bucket/file.txt

# Delete a folder (recursive)
aws s3 rm s3://my-bucket/folder/ --recursive

# Delete a bucket (must be empty)
aws s3 rb s3://my-bucket

# Force delete bucket with contents
aws s3 rb s3://my-bucket --force

# Generate a pre-signed URL (valid 1 hour)
aws s3 presign s3://my-bucket/private.pdf --expires-in 3600

# Enable static website hosting
aws s3 website s3://my-bucket \
  --index-document index.html \
  --error-document error.html

# Set bucket policy
aws s3api put-bucket-policy \
  --bucket my-bucket \
  --policy file://bucket-policy.json

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Enabled
```

---

## VPC & Networking

```bash
# List VPCs
aws ec2 describe-vpcs

# Create VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16

# Create subnet
aws ec2 create-subnet \
  --vpc-id vpc-0abc123 \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a

# Create internet gateway and attach
aws ec2 create-internet-gateway
aws ec2 attach-internet-gateway \
  --internet-gateway-id igw-0abc123 \
  --vpc-id vpc-0abc123

# Create route table
aws ec2 create-route-table --vpc-id vpc-0abc123

# Add route to internet
aws ec2 create-route \
  --route-table-id rtb-0abc123 \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-0abc123

# Associate route table with subnet
aws ec2 associate-route-table \
  --route-table-id rtb-0abc123 \
  --subnet-id subnet-0abc123

# Allocate & associate Elastic IP
aws ec2 allocate-address --domain vpc
aws ec2 associate-address \
  --instance-id i-0abc123 \
  --allocation-id eipalloc-0abc123

# Create NAT Gateway (for private subnets)
aws ec2 create-nat-gateway \
  --subnet-id subnet-0abc123 \
  --allocation-id eipalloc-0abc123
```

---

## RDS

```bash
# List DB instances
aws rds describe-db-instances \
  --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus,Endpoint.Address]' \
  --output table

# Create a PostgreSQL instance
aws rds create-db-instance \
  --db-instance-identifier mydb \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version 16.1 \
  --master-username admin \
  --master-user-password secret123 \
  --allocated-storage 20 \
  --no-publicly-accessible \
  --vpc-security-group-ids sg-0abc123 \
  --db-subnet-group-name my-subnet-group

# Create a read replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier mydb-replica \
  --source-db-instance-identifier mydb

# Create a snapshot
aws rds create-db-snapshot \
  --db-instance-identifier mydb \
  --db-snapshot-identifier mydb-snapshot-$(date +%Y%m%d)

# Restore from snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier mydb-restored \
  --db-snapshot-identifier mydb-snapshot-20240101

# Start / stop RDS instance
aws rds stop-db-instance --db-instance-identifier mydb
aws rds start-db-instance --db-instance-identifier mydb

# Delete RDS instance
aws rds delete-db-instance \
  --db-instance-identifier mydb \
  --skip-final-snapshot
```

---

## ECS & ECR

### ECR

```bash
# Create repository
aws ecr create-repository --repository-name myapp

# Login to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

# Tag and push image
docker tag myapp:latest 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest

# List images in a repository
aws ecr list-images --repository-name myapp

# Delete image
aws ecr batch-delete-image \
  --repository-name myapp \
  --image-ids imageTag=old-tag
```

### ECS

```bash
# List clusters
aws ecs list-clusters

# Create cluster
aws ecs create-cluster --cluster-name production

# List services in cluster
aws ecs list-services --cluster production

# Describe service
aws ecs describe-services \
  --cluster production \
  --services myapp-service

# Update service (deploy new task definition)
aws ecs update-service \
  --cluster production \
  --service myapp-service \
  --task-definition myapp:5 \
  --force-new-deployment

# Scale service
aws ecs update-service \
  --cluster production \
  --service myapp-service \
  --desired-count 5

# List running tasks
aws ecs list-tasks --cluster production --service-name myapp-service

# Stop a task
aws ecs stop-task --cluster production --task <task-arn>

# Run a one-off task
aws ecs run-task \
  --cluster production \
  --task-definition myapp-migrate \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-0abc],securityGroups=[sg-0abc],assignPublicIp=ENABLED}"
```

---

## EKS

```bash
# Create EKS cluster (uses eksctl)
eksctl create cluster \
  --name my-cluster \
  --region us-east-1 \
  --nodegroup-name workers \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 5 \
  --managed

# Update kubeconfig for kubectl
aws eks update-kubeconfig --name my-cluster --region us-east-1

# List clusters
aws eks list-clusters

# Describe cluster
aws eks describe-cluster --name my-cluster

# Add managed node group
eksctl create nodegroup \
  --cluster my-cluster \
  --name gpu-workers \
  --node-type p3.2xlarge \
  --nodes 2

# Delete cluster
eksctl delete cluster --name my-cluster
```

---

## Lambda

```bash
# List functions
aws lambda list-functions \
  --query 'Functions[*].[FunctionName,Runtime,LastModified]' \
  --output table

# Create function
aws lambda create-function \
  --function-name my-function \
  --runtime nodejs20.x \
  --role arn:aws:iam::123456789:role/MyLambdaRole \
  --handler index.handler \
  --zip-file fileb://function.zip

# Update function code
zip -r function.zip .
aws lambda update-function-code \
  --function-name my-function \
  --zip-file fileb://function.zip

# Invoke function
aws lambda invoke \
  --function-name my-function \
  --payload '{"key": "value"}' \
  --cli-binary-format raw-in-base64-out \
  output.json && cat output.json

# Update environment variables
aws lambda update-function-configuration \
  --function-name my-function \
  --environment Variables="{NODE_ENV=production,DB_URL=postgres://...}"

# Set concurrency
aws lambda put-function-concurrency \
  --function-name my-function \
  --reserved-concurrent-executions 50

# View logs
aws logs tail /aws/lambda/my-function --follow

# Delete function
aws lambda delete-function --function-name my-function
```

---

## CloudFormation

```bash
# Validate template
aws cloudformation validate-template --template-body file://stack.yaml

# Create stack
aws cloudformation create-stack \
  --stack-name my-stack \
  --template-body file://stack.yaml \
  --parameters ParameterKey=Env,ParameterValue=production \
  --capabilities CAPABILITY_NAMED_IAM

# Update stack
aws cloudformation update-stack \
  --stack-name my-stack \
  --template-body file://stack.yaml \
  --capabilities CAPABILITY_NAMED_IAM

# Describe stack
aws cloudformation describe-stacks --stack-name my-stack

# Watch stack events
aws cloudformation describe-stack-events --stack-name my-stack

# List stack resources
aws cloudformation list-stack-resources --stack-name my-stack

# Get stack outputs
aws cloudformation describe-stacks \
  --stack-name my-stack \
  --query 'Stacks[0].Outputs'

# Delete stack
aws cloudformation delete-stack --stack-name my-stack

# Create change set (preview changes)
aws cloudformation create-change-set \
  --stack-name my-stack \
  --change-set-name my-changes \
  --template-body file://stack.yaml

# Execute change set
aws cloudformation execute-change-set \
  --stack-name my-stack \
  --change-set-name my-changes
```

---

## CloudWatch

```bash
# List log groups
aws logs describe-log-groups

# Create log group
aws logs create-log-group --log-group-name /myapp/production

# Tail logs in real time
aws logs tail /myapp/production --follow

# Filter log events
aws logs filter-log-events \
  --log-group-name /myapp/production \
  --filter-pattern "ERROR" \
  --start-time $(date -d '1 hour ago' +%s000)

# Get metric statistics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0abc123 \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-01T01:00:00Z \
  --period 300 \
  --statistics Average

# Create an alarm
aws cloudwatch put-metric-alarm \
  --alarm-name high-cpu \
  --alarm-description "CPU > 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:123456789:alerts

# List alarms
aws cloudwatch describe-alarms --alarm-name-prefix high-
```

---

## Route 53

```bash
# List hosted zones
aws route53 list-hosted-zones

# Create hosted zone
aws route53 create-hosted-zone \
  --name example.com \
  --caller-reference "$(date +%s)"

# List records in a zone
aws route53 list-resource-record-sets \
  --hosted-zone-id Z123ABC

# Create/update a record (upsert)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123ABC \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "1.2.3.4"}]
      }
    }]
  }'

# Check DNS propagation
aws route53 test-dns-answer \
  --hosted-zone-id Z123ABC \
  --record-name api.example.com \
  --record-type A
```

---

## Secrets Manager & SSM Parameter Store

### Secrets Manager

```bash
# Create a secret
aws secretsmanager create-secret \
  --name /production/db/password \
  --secret-string "s3cr3tP@ssw0rd"

# Create JSON secret
aws secretsmanager create-secret \
  --name /production/app/credentials \
  --secret-string '{"username":"admin","password":"s3cr3t"}'

# Get secret value
aws secretsmanager get-secret-value \
  --secret-id /production/db/password \
  --query SecretString --output text

# Update secret
aws secretsmanager update-secret \
  --secret-id /production/db/password \
  --secret-string "newP@ssw0rd"

# List secrets
aws secretsmanager list-secrets
```

### SSM Parameter Store

```bash
# Put a parameter
aws ssm put-parameter \
  --name /production/app/db_url \
  --value "postgres://user:pass@host:5432/db" \
  --type SecureString

# Get a parameter
aws ssm get-parameter \
  --name /production/app/db_url \
  --with-decryption \
  --query Parameter.Value --output text

# Get all parameters by path
aws ssm get-parameters-by-path \
  --path /production/app/ \
  --with-decryption

# Delete parameter
aws ssm delete-parameter --name /production/app/db_url
```

---

## Advanced Scenarios

### Scenario 1 — Deploy a static site to S3 + CloudFront

```bash
# Build and sync
npm run build
aws s3 sync ./dist s3://my-site-bucket --delete

# Invalidate CloudFront cache
aws cloudfront create-invalidation \
  --distribution-id E1ABCDEF \
  --paths "/*"
```

### Scenario 2 — Query EC2 instances across all regions

```bash
for region in $(aws ec2 describe-regions --query 'Regions[*].RegionName' --output text); do
  echo "=== $region ==="
  aws ec2 describe-instances \
    --region $region \
    --filters "Name=instance-state-name,Values=running" \
    --query 'Reservations[*].Instances[*].[InstanceId,InstanceType]' \
    --output text
done
```

### Scenario 3 — Rotate access keys safely

```bash
# Create new key
aws iam create-access-key --user-name jane

# Update application to use new key, then:
# Deactivate old key first (safer than immediate delete)
aws iam update-access-key \
  --user-name jane \
  --access-key-id AKIA_OLD \
  --status Inactive

# Confirm nothing is broken, then delete
aws iam delete-access-key \
  --user-name jane \
  --access-key-id AKIA_OLD
```

### Scenario 4 — Find all unattached EBS volumes (cost savings)

```bash
aws ec2 describe-volumes \
  --filters "Name=status,Values=available" \
  --query 'Volumes[*].[VolumeId,Size,CreateTime]' \
  --output table
```

### Scenario 5 — Cross-account S3 copy using profiles

```bash
aws s3 cp s3://source-bucket/file.txt s3://dest-bucket/file.txt \
  --source-region us-east-1 \
  --region eu-west-1 \
  --profile source-account
```

---

## Quick Reference Card

| Task                          | Command                                                   |
|-------------------------------|-----------------------------------------------------------|
| Who am I?                     | `aws sts get-caller-identity`                             |
| List running EC2              | `aws ec2 describe-instances --filters "Name=instance-state-name,Values=running"` |
| S3 sync                       | `aws s3 sync ./dist s3://bucket --delete`                 |
| ECR login                     | `aws ecr get-login-password \| docker login --username AWS ...` |
| Tail Lambda logs              | `aws logs tail /aws/lambda/fn-name --follow`              |
| Get secret                    | `aws secretsmanager get-secret-value --secret-id /path`   |
| Update ECS service            | `aws ecs update-service --cluster c --service s --force-new-deployment` |
| Invoke Lambda                 | `aws lambda invoke --function-name fn output.json`        |
| Validate CF template          | `aws cloudformation validate-template --template-body file://stack.yaml` |

---

> Always apply least privilege: grant only the permissions your workload actually needs.
