# Terraform Cheatsheet

## Installation
```bash
# Linux
wget https://releases.hashicorp.com/terraform/1.7.0/terraform_1.7.0_linux_amd64.zip
unzip terraform_1.7.0_linux_amd64.zip && mv terraform /usr/local/bin/

# macOS
brew install terraform

# Verify
terraform version
```

---

## Core Workflow
```bash
terraform init          # Initialize working directory, download providers
terraform plan          # Preview changes (dry run)
terraform apply         # Apply changes
terraform apply -auto-approve   # Apply without confirmation prompt
terraform destroy       # Destroy all managed infrastructure
terraform destroy -auto-approve
```

---

## State Management
```bash
terraform show                        # Show current state
terraform state list                  # List all resources in state
terraform state show aws_instance.web # Show specific resource
terraform state mv old_name new_name  # Rename resource in state
terraform state rm aws_instance.web   # Remove resource from state (won't destroy)
terraform refresh                     # Sync state with real infrastructure
terraform output                      # Show output values
terraform output instance_ip          # Show specific output
```

---

## Remote State (S3 Backend)
```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-tf-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tf-state-lock"
    encrypt        = true
  }
}
```
```bash
terraform init -reconfigure   # Re-initialize with new backend
terraform init -migrate-state # Migrate existing state to new backend
```

---

## Workspaces
```bash
terraform workspace list        # List workspaces
terraform workspace new staging # Create new workspace
terraform workspace select prod # Switch workspace
terraform workspace show        # Show current workspace
terraform workspace delete dev  # Delete workspace
```

---

## Variables
```hcl
# variables.tf
variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t3.micro"
}

variable "tags" {
  type = map(string)
  default = {
    Environment = "dev"
    Team        = "devops"
  }
}
```
```bash
# Pass variables
terraform apply -var="instance_type=t3.small"
terraform apply -var-file="prod.tfvars"

# terraform.tfvars (auto-loaded)
instance_type = "t3.medium"
region        = "us-west-2"
```

---

## Outputs
```hcl
# outputs.tf
output "instance_ip" {
  value       = aws_instance.web.public_ip
  description = "Public IP of the web server"
}

output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true
}
```

---

## Providers
```hcl
# versions.tf
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region  = var.region
  profile = "my-profile"
}
```

---

## Common Resources

### AWS EC2
```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.instance_type

  tags = {
    Name = "web-server"
  }
}
```

### AWS VPC
```hcl
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true

  tags = { Name = "main-vpc" }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true
}
```

### AWS S3
```hcl
resource "aws_s3_bucket" "data" {
  bucket = "my-unique-bucket-name"
}

resource "aws_s3_bucket_versioning" "data" {
  bucket = aws_s3_bucket.data.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

### GCP Compute
```hcl
resource "google_compute_instance" "vm" {
  name         = "web-server"
  machine_type = "e2-medium"
  zone         = "us-central1-a"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  network_interface {
    network = "default"
    access_config {}
  }
}
```

### Azure VM
```hcl
resource "azurerm_linux_virtual_machine" "web" {
  name                = "web-vm"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  size                = "Standard_B1s"
  admin_username      = "adminuser"

  network_interface_ids = [azurerm_network_interface.main.id]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "UbuntuServer"
    sku       = "18.04-LTS"
    version   = "latest"
  }
}
```

---

## Meta-Arguments
```hcl
# count — create multiple resources
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  tags = { Name = "web-${count.index}" }
}

# for_each — create from map/set
resource "aws_iam_user" "users" {
  for_each = toset(["alice", "bob", "carol"])
  name     = each.value
}

# depends_on — explicit dependency
resource "aws_instance" "app" {
  depends_on = [aws_db_instance.main]
}

# lifecycle — control resource behavior
resource "aws_instance" "web" {
  lifecycle {
    create_before_destroy = true
    prevent_destroy       = true
    ignore_changes        = [ami, tags]
  }
}
```

---

## Data Sources
```hcl
# Fetch existing resource info
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-22.04-amd64-server-*"]
  }
}

resource "aws_instance" "web" {
  ami = data.aws_ami.ubuntu.id
}
```

---

## Locals
```hcl
locals {
  env    = terraform.workspace
  prefix = "${var.project}-${local.env}"
  common_tags = {
    Project     = var.project
    Environment = local.env
    ManagedBy   = "Terraform"
  }
}

resource "aws_s3_bucket" "data" {
  bucket = "${local.prefix}-data"
  tags   = local.common_tags
}
```

---

## Modules
```hcl
# Call a module
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
}

# Reference module output
resource "aws_instance" "app" {
  subnet_id = module.vpc.private_subnets[0]
}
```
```bash
terraform init          # Downloads module source
terraform get -update   # Update modules
```

---

## Import Existing Resources
```bash
terraform import aws_instance.web i-1234567890abcdef0
terraform import aws_s3_bucket.data my-existing-bucket
```

---

## Formatting & Validation
```bash
terraform fmt               # Format code
terraform fmt -recursive    # Format all files recursively
terraform validate          # Validate configuration syntax
terraform fmt -check        # Check formatting without modifying (CI use)
```

---

## Useful Flags
```bash
terraform plan -out=tfplan          # Save plan to file
terraform apply tfplan              # Apply saved plan
terraform plan -target=aws_instance.web   # Plan only specific resource
terraform apply -target=aws_instance.web  # Apply only specific resource
terraform plan -var-file=prod.tfvars
terraform apply -lock=false         # Skip state locking (use with caution)
terraform apply -parallelism=20     # Control concurrent operations (default 10)
```

---

## Debugging
```bash
TF_LOG=DEBUG terraform apply     # Verbose logging
TF_LOG=INFO terraform plan
TF_LOG_PATH=./terraform.log terraform apply
```

---

## Common Expressions
```hcl
# Conditional
instance_type = var.env == "prod" ? "t3.large" : "t3.micro"

# String interpolation
name = "${var.project}-${var.env}-server"

# Functions
upper("hello")               # "HELLO"
lower("HELLO")               # "hello"
length(var.list)             # count elements
join(",", ["a","b","c"])     # "a,b,c"
split(",", "a,b,c")          # ["a","b","c"]
merge(map1, map2)            # merge maps
toset(["a","b","a"])         # {"a","b"}
lookup(var.map, "key", "default")
file("path/to/file.txt")     # read file contents
base64encode("hello")
jsonencode({ key = "value" })
```
