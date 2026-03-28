# Terraform

Terraform is an open-source Infrastructure as Code (IaC) tool by HashiCorp that lets you define, provision, and manage cloud and on-premises infrastructure in a declarative, human-readable configuration language (HCL — HashiCorp Configuration Language). Terraform uses a **desired-state model**: you describe the infrastructure you want, and Terraform figures out the API calls needed to achieve it. It supports hundreds of providers (AWS, Azure, GCP, Kubernetes, GitHub, etc.) through a plugin architecture, enabling consistent workflows across any infrastructure platform.

---

## Table of Contents

1. [Installation](#installation)
2. [Core Workflow](#core-workflow)
3. [State Management](#state-management)
4. [Remote State Backends](#remote-state-backends)
5. [Workspaces](#workspaces)
6. [Variables & Outputs](#variables--outputs)
7. [Data Sources](#data-sources)
8. [Modules](#modules)
9. [Providers](#providers)
10. [Resource Lifecycle](#resource-lifecycle)
11. [Conditionals & Loops](#conditionals--loops)
12. [Functions & Expressions](#functions--expressions)
13. [Import & Move](#import--move)
14. [terraform fmt / validate / taint / untaint](#terraform-fmt--validate--taint--untaint)
15. [Terraform Cloud / Enterprise](#terraform-cloud--enterprise)
16. [Common Patterns & Tips](#common-patterns--tips)

---

## Installation

```bash
# macOS (Homebrew)
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Ubuntu / Debian
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# RHEL / CentOS / Fedora
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo yum install terraform

# Windows (Chocolatey)
choco install terraform

# Manual binary install (any platform)
TERRAFORM_VERSION=1.7.0
curl -LO "https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_amd64.zip"
unzip terraform_${TERRAFORM_VERSION}_linux_amd64.zip
sudo mv terraform /usr/local/bin/

# Version manager (tfenv) – manage multiple Terraform versions
brew install tfenv                       # macOS
tfenv install 1.7.0
tfenv use 1.7.0

# Verify installation
terraform version
# Terraform v1.7.0
```

---

## Core Workflow

The fundamental Terraform workflow is four commands: `init` → `plan` → `apply` → `destroy`.

### terraform init

Downloads provider plugins, sets up the backend, and installs modules.

```bash
terraform init

# Re-initialise after adding a new provider or backend
terraform init -upgrade             # upgrade provider versions

# Use a specific backend config file
terraform init -backend-config=backend.hcl

# Skip downloading modules (useful in CI when modules are pre-cached)
terraform init -get=false

# Migrate state to a new backend
terraform init -migrate-state
```

### terraform plan

Creates an execution plan: compares current state with the desired configuration and lists the changes that will be made. **No changes are applied.**

```bash
terraform plan

# Save the plan to a file (for use with apply)
terraform plan -out=tfplan

# Target a specific resource
terraform plan -target=aws_instance.web

# Pass variable values
terraform plan -var="region=us-west-2" -var="instance_count=3"

# Use a variables file
terraform plan -var-file="prod.tfvars"

# Plan a destroy (shows what would be deleted)
terraform plan -destroy

# Refresh state before planning (default: true)
terraform plan -refresh=false        # skip state refresh for speed
```

### terraform apply

Executes the changes in the plan to reach the desired state.

```bash
# Interactive apply (shows plan then prompts for confirmation)
terraform apply

# Apply without interactive confirmation
terraform apply -auto-approve

# Apply a saved plan (no re-planning, skips confirmation)
terraform apply tfplan

# Target a specific resource
terraform apply -target=aws_instance.web -auto-approve

# Pass variables
terraform apply -var="environment=production" -auto-approve

# Parallelism (default: 10 concurrent operations)
terraform apply -parallelism=20
```

### terraform destroy

Destroys all resources managed by the current configuration.

```bash
# Destroy all resources (prompts for confirmation)
terraform destroy

# Destroy without confirmation
terraform destroy -auto-approve

# Destroy a specific resource
terraform destroy -target=aws_instance.old_server -auto-approve

# Preview what will be destroyed (same as plan -destroy)
terraform plan -destroy
```

---

## State Management

Terraform tracks the current state of your infrastructure in a **state file** (`terraform.tfstate`). This file is the source of truth that Terraform compares against desired configuration.

```bash
# List all resources in the state
terraform state list

# Show details of a specific resource
terraform state show aws_instance.web
terraform state show 'aws_s3_bucket.my_bucket["production"]'

# Move a resource to a new address (rename without destroying)
terraform state mv aws_instance.web aws_instance.app_server

# Move a resource into a module
terraform state mv aws_instance.web module.compute.aws_instance.web

# Remove a resource from state (Terraform forgets it, but it still exists)
terraform state rm aws_instance.orphan

# Pull the current state to stdout (useful for inspection or backup)
terraform state pull

# Push a local state file to the remote backend (use with care)
terraform state push terraform.tfstate

# Refresh state (update state file to match real-world infrastructure)
terraform refresh                    # deprecated; use apply -refresh-only
terraform apply -refresh-only        # preferred modern approach
```

### Inspecting State

```bash
# Show all outputs
terraform output

# Show a specific output
terraform output vpc_id

# Show output as JSON (useful for scripting)
terraform output -json

# Show full state as JSON
terraform show -json | jq .
```

---

## Remote State Backends

By default, Terraform stores state locally in `terraform.tfstate`. Remote backends store state in a shared, durable location with optional locking.

### S3 Backend (AWS)

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/vpc/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"   # enables state locking
  }
}
```

```bash
# Create the DynamoDB lock table (one-time setup)
aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

### Azure Blob Storage Backend

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "tfstatemyorg"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```

### GCS Backend (GCP)

```hcl
terraform {
  backend "gcs" {
    bucket = "my-terraform-state-bucket"
    prefix = "prod/infra"
  }
}
```

### Terraform Cloud Backend

```hcl
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "production"
    }
  }
}
```

### Partial Backend Configuration (for CI/CD)

```hcl
# main.tf – placeholder with no values
terraform {
  backend "s3" {}
}
```

```bash
# Supply values at init time (safe for CI pipelines)
terraform init \
  -backend-config="bucket=my-tf-state" \
  -backend-config="key=prod/terraform.tfstate" \
  -backend-config="region=us-east-1"

# Or use a backend config file
terraform init -backend-config=backend.hcl
```

---

## Workspaces

Workspaces provide isolated state files within a single backend, allowing you to reuse the same configuration for multiple environments.

```bash
# List workspaces (default workspace always exists)
terraform workspace list

# Create a new workspace
terraform workspace new staging
terraform workspace new production

# Switch to a workspace
terraform workspace select staging

# Show current workspace
terraform workspace show

# Delete a workspace (must switch away first; cannot delete default)
terraform workspace delete staging
```

### Using Workspaces in Configuration

```hcl
locals {
  environment = terraform.workspace     # "default", "staging", "production"

  instance_type = {
    default    = "t3.micro"
    staging    = "t3.small"
    production = "t3.large"
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = local.instance_type[terraform.workspace]

  tags = {
    Environment = terraform.workspace
  }
}
```

> **Note:** For complex multi-environment setups, separate directories or root modules per environment are often more maintainable than workspaces.

---

## Variables & Outputs

### Input Variables

```hcl
# variables.tf
variable "region" {
  description = "AWS region to deploy to"
  type        = string
  default     = "us-east-1"
}

variable "instance_count" {
  description = "Number of EC2 instances"
  type        = number
  default     = 1

  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 10
    error_message = "instance_count must be between 1 and 10."
  }
}

variable "allowed_cidr_blocks" {
  description = "CIDR blocks allowed to access the load balancer"
  type        = list(string)
  default     = ["0.0.0.0/0"]
}

variable "tags" {
  description = "Common tags to apply to all resources"
  type        = map(string)
  default     = {}
}

variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true        # prevents value from appearing in logs/output
}
```

**Variable precedence (highest wins):**
1. `-var` and `-var-file` command line flags
2. `*.auto.tfvars` and `*.auto.tfvars.json` files (alphabetical)
3. `terraform.tfvars.json`
4. `terraform.tfvars`
5. Environment variables (`TF_VAR_<name>`)
6. Default values in `variable` block

```bash
# Supply variables at runtime
terraform apply -var="region=eu-west-1" -var="instance_count=3"

# Supply via tfvars file
terraform apply -var-file="prod.tfvars"

# Supply via environment variable
export TF_VAR_db_password="supersecret"
terraform apply
```

```hcl
# prod.tfvars
region         = "eu-west-1"
instance_count = 3
tags = {
  Environment = "production"
  Team        = "platform"
}
```

### Output Values

```hcl
# outputs.tf
output "vpc_id" {
  description = "The ID of the created VPC"
  value       = aws_vpc.main.id
}

output "load_balancer_dns" {
  description = "DNS name of the load balancer"
  value       = aws_lb.main.dns_name
}

output "db_connection_string" {
  description = "Database connection string"
  value       = "postgresql://${aws_db_instance.main.endpoint}/appdb"
  sensitive   = true            # masks value in CLI output
}
```

```bash
# Read outputs after apply
terraform output
terraform output vpc_id
terraform output -json                 # machine-readable JSON
terraform output -raw load_balancer_dns  # unquoted raw string
```

---

## Data Sources

Data sources let you fetch existing infrastructure information or external data for use in your configuration — without managing those resources.

```hcl
# Fetch the latest Ubuntu 22.04 AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Use the data source
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
}

# Fetch an existing VPC by tag
data "aws_vpc" "existing" {
  filter {
    name   = "tag:Name"
    values = ["production-vpc"]
  }
}

# Fetch all subnets in the VPC
data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.existing.id]
  }
  filter {
    name   = "tag:Tier"
    values = ["Private"]
  }
}

# Read a local file
data "local_file" "config" {
  filename = "${path.module}/config.json"
}

# Read a secret from AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db_creds" {
  secret_id = "prod/myapp/db"
}

locals {
  db_creds = jsondecode(data.aws_secretsmanager_secret_version.db_creds.secret_string)
}
```

---

## Modules

Modules are self-contained, reusable packages of Terraform configuration. Every Terraform configuration is a module (the **root module**). Calling other modules creates a tree of modules.

### Calling a Module

```hcl
# Root module calling a child module
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"  # Terraform Registry
  version = "~> 5.0"                          # pin version

  name             = "production-vpc"
  cidr             = "10.0.0.0/16"
  azs              = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets  = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets   = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  enable_nat_gateway = true
  single_nat_gateway = false

  tags = var.common_tags
}

# Use module outputs
resource "aws_instance" "web" {
  subnet_id = module.vpc.private_subnets[0]
}

# Local module
module "security_groups" {
  source  = "./modules/security-groups"
  vpc_id  = module.vpc.vpc_id
  env     = var.environment
}

# GitHub module source
module "ecs_service" {
  source = "git::https://github.com/myorg/terraform-modules.git//ecs-service?ref=v1.2.0"
  name   = "myapp"
}
```

### Creating a Module

```
modules/
└── ec2-instance/
    ├── main.tf       # Resources
    ├── variables.tf  # Input variables
    ├── outputs.tf    # Output values
    └── README.md
```

```hcl
# modules/ec2-instance/main.tf
resource "aws_instance" "this" {
  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id

  tags = merge(var.tags, {
    Name = var.name
  })
}

# modules/ec2-instance/variables.tf
variable "name"          { type = string }
variable "ami_id"        { type = string }
variable "instance_type" { type = string; default = "t3.micro" }
variable "subnet_id"     { type = string }
variable "tags"          { type = map(string); default = {} }

# modules/ec2-instance/outputs.tf
output "instance_id"         { value = aws_instance.this.id }
output "private_ip"          { value = aws_instance.this.private_ip }
output "public_ip"           { value = aws_instance.this.public_ip }
```

---

## Providers

Providers are plugins that implement resource types for specific platforms (AWS, Azure, GCP, Kubernetes, GitHub, etc.).

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = ">= 2.23"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.12"
    }
  }
}

# Configure provider
provider "aws" {
  region  = var.region
  profile = "production"                  # AWS CLI profile

  default_tags {
    tags = {
      ManagedBy   = "Terraform"
      Repository  = "github.com/myorg/infra"
    }
  }
}

# Multiple provider configurations (aliases)
provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu_west"
  region = "eu-west-1"
}

# Use an aliased provider
resource "aws_instance" "eu_server" {
  provider      = aws.eu_west
  ami           = "ami-0abc123"
  instance_type = "t3.micro"
}

# Cross-account provider using AssumeRole
provider "aws" {
  alias  = "prod_account"
  region = "us-east-1"

  assume_role {
    role_arn     = "arn:aws:iam::123456789012:role/TerraformDeployRole"
    session_name = "TerraformSession"
  }
}
```

---

## Resource Lifecycle

The `lifecycle` block controls how Terraform creates, updates, and deletes resources.

### create_before_destroy

When a resource needs to be replaced, Terraform normally destroys the old one first. `create_before_destroy` inverts this — creating the replacement before destroying the original. Useful for zero-downtime deployments.

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  lifecycle {
    create_before_destroy = true
  }
}
```

### prevent_destroy

Prevents accidental deletion. `terraform destroy` or any plan that would delete the resource will fail with an error.

```hcl
resource "aws_rds_cluster" "production" {
  cluster_identifier = "prod-aurora-cluster"
  engine             = "aurora-postgresql"

  lifecycle {
    prevent_destroy = true
  }
}
```

### ignore_changes

Tells Terraform to ignore changes to specific attributes — useful when an external process modifies a resource after Terraform creates it (e.g., an autoscaler changing instance counts).

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  lifecycle {
    ignore_changes = [
      ami,                 # ignore AMI updates (managed by patching process)
      user_data,           # ignore user_data changes
    ]
  }
}

# Ignore all changes (use very sparingly)
resource "aws_autoscaling_group" "app" {
  lifecycle {
    ignore_changes = all
  }
}
```

### replace_triggered_by

Force resource replacement when a referenced resource or attribute changes.

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  lifecycle {
    replace_triggered_by = [
      aws_security_group.web.id     # replace instance if SG changes
    ]
  }
}
```

---

## Conditionals & Loops

### count — Create N Copies of a Resource

```hcl
# Create a variable number of instances
resource "aws_instance" "web" {
  count         = var.instance_count
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  tags = {
    Name = "web-${count.index + 1}"
  }
}

# Conditional resource creation (count = 0 or 1)
resource "aws_eip" "web" {
  count    = var.create_eip ? 1 : 0
  instance = aws_instance.web[0].id
}

# Reference a counted resource
output "web_ips" {
  value = aws_instance.web[*].private_ip   # splat expression
}
```

### for_each — Create Resources from a Map or Set

Preferred over `count` when resources are addressed by a meaningful key rather than an index.

```hcl
# Create from a map
variable "buckets" {
  default = {
    logs    = "private"
    assets  = "public-read"
    backups = "private"
  }
}

resource "aws_s3_bucket" "this" {
  for_each = var.buckets
  bucket   = "${var.project}-${each.key}"

  tags = {
    Name = each.key
    ACL  = each.value
  }
}

# Create from a set of strings
resource "aws_iam_user" "this" {
  for_each = toset(["alice", "bob", "charlie"])
  name     = each.key
}

# Nested for_each using for expression
locals {
  user_role_pairs = {
    for pair in setproduct(["alice", "bob"], ["admin", "viewer"]) :
    "${pair[0]}_${pair[1]}" => {
      user = pair[0]
      role = pair[1]
    }
  }
}

resource "aws_iam_user_policy_attachment" "this" {
  for_each   = local.user_role_pairs
  user       = each.value.user
  policy_arn = aws_iam_policy.roles[each.value.role].arn
}
```

### dynamic Blocks — Conditionally Repeat Nested Blocks

```hcl
# Dynamically generate ingress rules
variable "ingress_rules" {
  default = [
    { port = 80,  cidr = "0.0.0.0/0",    description = "HTTP" },
    { port = 443, cidr = "0.0.0.0/0",    description = "HTTPS" },
    { port = 22,  cidr = "10.0.0.0/8",   description = "SSH" },
  ]
}

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = module.vpc.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = [ingress.value.cidr]
      description = ingress.value.description
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## Functions & Expressions

Terraform provides a rich set of built-in functions for string manipulation, collections, encoding, and more.

```hcl
locals {
  # String functions
  upper_name    = upper("hello")                    # "HELLO"
  lower_name    = lower("HELLO")                    # "hello"
  trimmed       = trimspace("  hello  ")            # "hello"
  joined        = join(", ", ["a", "b", "c"])       # "a, b, c"
  split_list    = split(",", "a,b,c")               # ["a", "b", "c"]
  replaced      = replace("hello world", "world", "terraform")

  # Number functions
  max_val       = max(5, 12, 9)                     # 12
  min_val       = min(5, 12, 9)                     # 5
  ceil_val      = ceil(1.2)                          # 2
  floor_val     = floor(1.9)                         # 1

  # Collection functions
  list_length   = length(["a", "b", "c"])           # 3
  map_keys      = keys({ a = 1, b = 2 })            # ["a", "b"]
  map_values    = values({ a = 1, b = 2 })          # [1, 2]
  merged_map    = merge({ a = 1 }, { b = 2 })       # { a = 1, b = 2 }
  flat_list     = flatten([["a","b"],["c"]])         # ["a", "b", "c"]
  unique_list   = toset(["a", "b", "a"])             # {"a", "b"}

  # Type conversion
  to_string     = tostring(42)                      # "42"
  to_number     = tonumber("42")                    # 42
  to_list       = tolist(toset(["a","b"]))

  # Encoding
  base64_enc    = base64encode("hello")
  base64_dec    = base64decode("aGVsbG8=")
  json_enc      = jsonencode({ key = "value" })
  json_dec      = jsondecode("{\"key\":\"value\"}")

  # Filesystem functions
  file_content  = file("${path.module}/script.sh")
  template_out  = templatefile("${path.module}/user_data.sh.tpl", {
    db_host = aws_db_instance.main.endpoint
    app_env = var.environment
  })

  # Conditional expression
  instance_type = var.environment == "production" ? "t3.large" : "t3.micro"

  # For expressions (transform collections)
  upper_names   = [for name in var.names : upper(name)]
  filtered      = [for n in var.names : n if length(n) > 3]
  name_map      = { for n in var.names : n => upper(n) }

  # Null and default handling
  with_default  = coalesce(var.optional_value, "default")
  try_value     = try(var.complex_obj.nested.value, "fallback")
}
```

---

## Import & Move

### terraform import

Brings existing infrastructure under Terraform management without recreating it.

```bash
# Import syntax: terraform import <resource_address> <provider_id>
terraform import aws_instance.web i-1234567890abcdef0
terraform import aws_s3_bucket.logs my-existing-bucket-name
terraform import aws_security_group.web sg-0abc12345def67890

# Import into a module
terraform import 'module.vpc.aws_vpc.this' vpc-0abc123def456789

# Import with for_each (Terraform >= 1.5 supports import blocks)
```

### Import Blocks (Terraform >= 1.5)

```hcl
# import.tf – declarative import
import {
  to = aws_instance.web
  id = "i-1234567890abcdef0"
}

import {
  to = aws_s3_bucket.logs
  id = "my-existing-bucket"
}

# Generate configuration automatically (Terraform >= 1.5)
# terraform plan -generate-config-out=generated.tf
```

### terraform state mv (moved block)

When refactoring configuration (e.g., moving a resource into a module), use `moved` blocks to update state without destroying resources.

```hcl
# moved.tf
moved {
  from = aws_instance.web
  to   = module.compute.aws_instance.web
}

moved {
  from = aws_security_group.app
  to   = module.compute.aws_security_group.app
}
```

---

## terraform fmt / validate / taint / untaint

### terraform fmt

Automatically formats Terraform files to the canonical style.

```bash
# Format all .tf files in current directory recursively
terraform fmt -recursive

# Check formatting without modifying files (exit 1 if changes needed)
terraform fmt -check -recursive

# Show diff of changes that would be made
terraform fmt -diff

# Format a specific file
terraform fmt main.tf
```

### terraform validate

Validates the configuration syntax and internal consistency. Does **not** contact the provider API.

```bash
# Validate configuration in current directory
terraform validate

# Output validation result as JSON
terraform validate -json

# Common usage in CI (after init)
terraform init -backend=false && terraform validate
```

### terraform taint (deprecated in 1.0; use -replace)

Marks a resource for replacement on the next `apply`. Deprecated in favour of the `-replace` flag.

```bash
# Modern approach (Terraform >= 0.15.2)
terraform apply -replace=aws_instance.web

# Legacy taint (still works but deprecated)
terraform taint aws_instance.web
terraform untaint aws_instance.web   # remove the taint mark

# Force-replace a resource in a module
terraform apply -replace='module.compute.aws_instance.web'
```

### terraform graph

Generates a visual dependency graph of resources.

```bash
# Output in DOT format (pipe to graphviz)
terraform graph | dot -Tsvg > graph.svg
terraform graph | dot -Tpng > graph.png
```

---

## Terraform Cloud / Enterprise

Terraform Cloud (TFC) and Terraform Enterprise (TFE) provide remote state storage, team collaboration, policy as code (Sentinel/OPA), run histories, and SSO.

### Configuration

```hcl
# versions.tf
terraform {
  cloud {
    organization = "my-org"

    workspaces {
      name = "production-infra"          # single workspace
      # tags = ["production", "aws"]     # workspace by tags
    }
  }
}
```

```bash
# Log in to Terraform Cloud
terraform login                          # opens browser for API token

# Log out
terraform logout

# After configuring the cloud block, init will configure remote execution
terraform init
```

### Key Features

| Feature | Description |
|---|---|
| Remote State | State stored securely in TFC with locking |
| Remote Runs | `plan` and `apply` run on TFC infrastructure |
| VCS Integration | Auto-plan on pull requests |
| Private Registry | Host private modules and providers |
| Policy as Code | Sentinel/OPA policies to enforce guardrails |
| Team Permissions | Fine-grained access control per workspace |
| Audit Logs | Full history of all runs and changes |

### Workspace Variables in TFC

```bash
# Set a workspace variable via CLI
terraform workspace select production
# Or set via TFC UI / API under Workspace > Variables

# Variable types:
# Terraform variables – equivalent to -var flag
# Environment variables – available during runs (e.g., AWS_ACCESS_KEY_ID)
# Sensitive variables – stored encrypted, never shown in UI/logs
```

### API-Driven Runs

```bash
# Trigger a run via Terraform Cloud API
curl \
  --header "Authorization: Bearer $TFC_TOKEN" \
  --header "Content-Type: application/vnd.api+json" \
  --request POST \
  --data '{
    "data": {
      "type": "runs",
      "attributes": {
        "is-destroy": false,
        "message": "Triggered by CI pipeline"
      },
      "relationships": {
        "workspace": {
          "data": {
            "type": "workspaces",
            "id": "ws-XXXXXXXXXX"
          }
        }
      }
    }
  }' \
  https://app.terraform.io/api/v2/runs
```

---

## Common Patterns & Tips

1. **Always run `terraform plan` before `terraform apply`.**  
   Save the plan with `-out=tfplan` and apply the saved plan file. This ensures exactly what was reviewed is applied — no surprises from infrastructure drift between plan and apply.

2. **Use remote state with locking from day one.**  
   Local state files in version control cause merge conflicts and risk state corruption when two people run Terraform simultaneously. Set up S3 + DynamoDB (AWS), GCS (GCP), or Azure Blob with locking before any team uses Terraform.

3. **Structure code as root module + reusable modules.**  
   Keep `environments/prod/main.tf` and `environments/staging/main.tf` as thin root modules that call shared modules from `modules/`. This enforces consistency across environments while allowing per-environment customisation through variables.

4. **Pin all provider and module versions.**  
   Use `~>` (pessimistic constraint) to allow patch updates but prevent minor version jumps: `version = "~> 5.0"`. Commit the `.terraform.lock.hcl` file to version control to ensure all team members and CI use identical provider versions.

5. **Use `sensitive = true` for all secret variables and outputs.**  
   This prevents values from appearing in plan/apply output and in Terraform Cloud logs. Combine with a secrets manager data source to avoid hardcoding secrets in `.tfvars` files.

6. **Prefer `for_each` over `count` for most resource collections.**  
   `count`-based resources are referenced by index (`aws_instance.web[0]`), so inserting an element in the middle forces Terraform to replace all subsequent resources. `for_each` uses stable string keys, so additions/removals only affect the specific resource.

7. **Use `moved` blocks instead of `terraform state mv` when refactoring.**  
   Declarative `moved` blocks in `.tf` files are version-controlled, reviewable, and automatically applied during `plan`/`apply`. `terraform state mv` requires manual execution and is not reproducible.

8. **Implement `prevent_destroy` on critical stateful resources.**  
   Add `lifecycle { prevent_destroy = true }` to databases, object storage buckets, and any resource that holds data. This is a last line of defence against accidental `terraform destroy` or misconfigured replacement operations.

9. **Use `templatefile()` instead of `user_data = file()`.**  
   `templatefile("user_data.sh.tpl", { ... })` lets you pass Terraform variables into shell scripts and other user data, making them dynamic without string interpolation hacks.

10. **Keep Terraform workspaces simple; prefer directories for environments.**  
    Workspaces share the same code, making it easy to accidentally apply staging config to production. Separate directories per environment with their own state files provide stronger isolation, clearer review boundaries, and are easier to audit.
