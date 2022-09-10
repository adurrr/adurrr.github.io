+++
author = "Adur"
title = "Infrastructure as code with Terraform: a practical guide"
date = "2022-09-10"
description = "A hands-on guide to Terraform covering core concepts, practical examples, state management, and production best practices."
featured = true
tags = [
    "terraform",
    "iac",
    "infrastructure",
    "hcl",
    "automation"
]
categories = [
    "devops"
]
series = ["DevOps"]
thumbnail = "images/devops.png"
toc = true
+++

## Why infrastructure as code matters

Managing infrastructure manually through web consoles or ad-hoc scripts creates problems that pile up over time: inconsistent environments, undocumented changes, impossible rollbacks, and the classic "it works on my machine" extended to entire servers.

Infrastructure as Code (IaC) fixes this by treating infrastructure like application code: it's written, versioned, reviewed, tested, and applied through automated workflows. The benefits show up right away:

- **Reproducibility**: Spin up identical environments in minutes, not days.
- **Version control**: Every infrastructure change goes through a PR with code review.
- **Documentation by default**: The code *is* the documentation of what your infrastructure looks like.
- **Disaster recovery**: Rebuild everything from code if a region goes down.
- **Cost visibility**: Review infrastructure changes before they are applied (and before they start costing money).

## Terraform vs other tools

Several IaC tools exist. Here's how Terraform compares to the main alternatives:

| Feature | Terraform | Pulumi | CloudFormation | Ansible |
|---|---|---|---|---|
| **Language** | HCL (declarative) | Python, TypeScript, Go, etc. | JSON/YAML | YAML (procedural) |
| **Cloud support** | Multi-cloud | Multi-cloud | AWS only | Multi-cloud (via modules) |
| **State management** | Explicit state file | Managed by Pulumi service | Managed by AWS | Stateless |
| **Learning curve** | Moderate | Varies by language | Moderate | Low |
| **Ecosystem** | Huge provider ecosystem | Growing | AWS-only but deep | Huge role ecosystem |
| **Best for** | Multi-cloud infra | Teams that prefer general-purpose languages | AWS-only shops | Configuration management |

Terraform's sweet spot is multi-cloud infrastructure provisioning with a declarative approach. If you're on AWS only and want tight integration, CloudFormation is reasonable. If your team prefers writing Python over HCL, Pulumi deserves a look. But for most teams managing infrastructure across providers, Terraform is the pragmatic choice.

## Core concepts

### Providers

Providers are plugins that let Terraform interact with APIs — AWS, Azure, GCP, Kubernetes, GitHub, Cloudflare, and hundreds more.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "eu-west-1"
}
```

### Resources

Resources are the fundamental building blocks. Each resource block describes one infrastructure object.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name = "web-server"
  }
}
```

### State

Terraform maintains a **state file** that maps your configuration to real-world resources. This is how Terraform knows what exists, what needs to change, and what to destroy. The state file is critical. Losing it means Terraform loses track of your infrastructure.

### Modules

Modules are reusable packages of Terraform configuration. Think of them as functions: they take inputs (variables), create resources, and produce outputs.

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["eu-west-1a", "eu-west-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
}
```

## Practical example: VPC + EC2

Here's a complete example that provisions a VPC with a public subnet and an EC2 instance:

```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "eu-west-1"
}

# --- Networking ---

resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "main-vpc"
  }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "eu-west-1a"
  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet"
  }
}

resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "main-igw"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.gw.id
  }

  tags = {
    Name = "public-rt"
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# --- Security Group ---

resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Allow HTTP and SSH"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["YOUR_IP/32"]  # Restrict to your IP
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# --- EC2 Instance ---

resource "aws_instance" "web" {
  ami                    = "ami-0c55b159cbfafe1f0"
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]

  tags = {
    Name = "web-server"
  }
}

# --- Outputs ---

output "instance_public_ip" {
  value = aws_instance.web.public_ip
}

output "vpc_id" {
  value = aws_vpc.main.id
}
```

## The plan/apply workflow

Terraform follows a predictable workflow:

```bash
# 1. Initialize - download providers and modules
terraform init

# 2. Format - ensure consistent code style
terraform fmt

# 3. Validate - check syntax and configuration
terraform validate

# 4. Plan - preview what will change (critical step!)
terraform plan -out=tfplan

# 5. Apply - execute the plan
terraform apply tfplan

# 6. Destroy - tear down all resources (when needed)
terraform destroy
```

The `terraform plan` step is the most important. Never skip it. Always review the plan output before applying, especially in production. The plan shows you exactly what will be created, modified, or destroyed.

```bash
# Example plan output
Plan: 6 to add, 0 to change, 0 to destroy.
```

In CI/CD pipelines, save the plan to a file (`-out=tfplan`) and apply that exact plan. This prevents race conditions where infrastructure changes between the plan and apply steps.

## State management best practices

State management is where most Terraform problems originate. Follow these practices:

### Use a remote backend

Never store state locally or in Git. Use a remote backend with encryption and locking:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/networking/terraform.tfstate"
    region         = "eu-west-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

The DynamoDB table provides **state locking**. This prevents two people or pipelines from modifying the same infrastructure at the same time.

### Organize state by component

Don't put all your infrastructure in one state file. Split by component or team:

```
environments/
├── prod/
│   ├── networking/    # VPC, subnets, routes
│   ├── compute/       # EC2, ASGs, load balancers
│   ├── database/      # RDS instances
│   └── monitoring/    # CloudWatch, alerts
└── staging/
    ├── networking/
    ├── compute/
    └── database/
```

Smaller state files mean faster plans, smaller blast radius, and fewer teams competing for locks.

### Use `terraform_remote_state` sparingly

You can reference outputs from other state files, but use it carefully. Over-reliance on remote state creates tight coupling between components. Prefer passing values through variables or a parameter store.

## Tips for production use

1. **Pin provider versions.** Use `~>` constraints to allow patch updates but prevent breaking changes: `version = "~> 5.0"`.

2. **Use workspaces carefully.** Workspaces are useful for simple environment separation but get confusing at scale. Separate directories per environment is usually clearer.

3. **Implement a CI/CD pipeline for Terraform.** Run `terraform plan` on PRs and post the output as a PR comment. Run `terraform apply` only after merge and approval.

4. **Use `prevent_destroy` for critical resources.** This lifecycle rule stops accidental destruction of databases or persistent storage:

    ```hcl
    resource "aws_db_instance" "main" {
      # ...
      lifecycle {
        prevent_destroy = true
      }
    }
    ```

5. **Tag everything.** Use a `default_tags` block in the provider to ensure every resource gets standard tags (environment, team, project).

6. **Use `tflint` and `checkov`.** Lint your Terraform code and scan for security misconfigurations before applying.

    ```bash
    tflint --init
    tflint
    checkov -d .
    ```

7. **Import existing resources.** If you have manually created infrastructure, use `terraform import` to bring it under management instead of recreating it.

8. **Review the plan diff carefully.** A resource showing "destroy and recreate" might cause downtime. Understand which changes are in-place versus destructive.

Terraform is one of those tools that rewards discipline. The more consistently you follow these practices, the more confidently your team manages infrastructure at scale.
