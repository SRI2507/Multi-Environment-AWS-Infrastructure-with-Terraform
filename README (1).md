# Multi-Environment AWS Infrastructure with Terraform

This project provisions AWS infrastructure (EC2 instance and S3 bucket for static website hosting) using **Terraform**, with environment-specific configurations managed through **Terraform Workspaces** (`dev`, `stage`, `prod`).

## Overview

- Uses Terraform Workspaces to isolate state files per environment (dev, stage, prod)
- Uses the `lookup()` function to dynamically assign EC2 instance types based on the active workspace
- Uses separate `.tfvars` files per environment for clean, environment-specific variable management
- Deploys an EC2 instance and an S3 bucket configured for static website hosting

## Prerequisites

- Terraform installed (v1.x recommended)
- AWS CLI configured with valid credentials (`aws configure`)
- An AWS account with permissions to create EC2 and S3 resources

## Project Structure

```
.
├── main.tf                # Root module - provider, variables, module calls
├── module/
│   └── ec2_instance/       # EC2 instance module
├── dev.tfvars
├── stage.tfvars
├── prod.tfvars
└── README.md
```

## Terraform Workspaces

Terraform workspaces allow you to maintain **separate state files** for each environment (dev, stage, prod) using a single set of configuration files.

### 1. Create a new workspace

```bash
terraform workspace new dev
```

This creates a `dev` workspace. Once created, Terraform automatically generates a `terraform.tfstate.d` directory to store workspace-specific state files.

```bash
ls terraform.tfstate.d/
# dev/  prod/  stage/
```

Each subdirectory under `terraform.tfstate.d` holds the isolated `terraform.tfstate` file for that environment, so resources in `dev` never conflict with resources in `prod`.

### 2. Switch between workspaces

```bash
terraform workspace select dev
```

Once inside a workspace, initialize and apply as usual:

```bash
terraform init
terraform apply -var-file=dev.tfvars
```

The resulting state file (containing details of the created resources, such as instance IDs) is stored under `terraform.tfstate.d/dev/`.

### 3. Environment-specific variable files

Each environment has its own `.tfvars` file, allowing different configurations (AMI, instance type, region-specific values, etc.) per environment:

```bash
terraform apply -var-file=dev.tfvars
terraform apply -var-file=stage.tfvars
terraform apply -var-file=prod.tfvars
```

## Dynamic Instance Sizing with `lookup()`

Instead of duplicating instance-type logic across environments, this project uses Terraform's `lookup()` function to automatically resolve the correct instance type based on the **active workspace name** (`terraform.workspace`):

```hcl
provider "aws" {
  region = "us-east-1"
}

variable "ami" {
  description = "AMI ID used to create the EC2 instance"
}

variable "instance_type" {
  description = "Map of instance types per environment/workspace"
  type = map(string)
  default = {
    "dev"   = "t3.micro"
    "prod"  = "t3.micro"
    "stage" = "t3.xlarge"
  }
}

module "ec2_instance" {
  source        = "./module/ec2_instance"
  ami           = var.ami
  instance_type = lookup(var.instance_type, terraform.workspace, "t3.micro")
}
```

**How it works:**
- `lookup(map, key, default)` checks the `instance_type` map for a key matching the current workspace name (`terraform.workspace`).
- If a match is found (e.g., running `terraform apply` inside the `prod` workspace), it uses the corresponding value (`t3.micro` for prod).
- If no match is found for the current workspace name, it falls back to the default value (`t3.micro`).
- This removes the need for separate hardcoded instance-type variables per environment and keeps the configuration DRY (Don't Repeat Yourself).

## S3 Static Website Hosting

The project also provisions an S3 bucket configured for static website hosting, allowing deployment of a simple website alongside the EC2 infrastructure.

## Notes

- State files are environment-isolated via workspaces, reducing the risk of accidentally modifying the wrong environment's resources.
- Recommended: use a remote backend (e.g., S3 + DynamoDB for state locking) instead of local state for team collaboration and production use.
