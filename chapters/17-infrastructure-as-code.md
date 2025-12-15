# Chapter 17: Infrastructure as Code

> **First Principles Question**: Why should infrastructure be defined in code? What problems does this solve, and how does treating infrastructure like software change the way we build and operate systems?

---

## Chapter Overview

Infrastructure as Code (IaC) represents a fundamental shift in how we manage infrastructure. Instead of clicking through consoles or running ad-hoc commands, we describe our desired infrastructure in code files that can be versioned, reviewed, tested, and automated.

**What readers will understand after this chapter:**
- Why Infrastructure as Code exists and what problems it solves
- The core concepts: declarative vs. imperative, state management
- AWS CloudFormation fundamentals
- Terraform: the multi-cloud IaC tool
- Best practices for managing infrastructure as code
- How IaC fits into the broader DevOps ecosystem

---

## Section 1: The Problem with Manual Infrastructure

### 1.1 The Console Click Problem

**Scenario: Setting Up a New Environment**
```
Manual Process:
1. Log into AWS Console
2. Create VPC (click, click, click)
3. Create subnets (click, click, click)
4. Create security groups (click, click, click)
5. Launch EC2 instances (click, click, click)
6. Create RDS database (click, click, click)
7. Configure everything to connect (click, click, click)

Time: 2-4 hours
Repeatability: Low
Documentation: None (unless you wrote it down)
```

**The Problems:**
```
├── Time-consuming
├── Error-prone (typos, missed steps)
├── Not repeatable (will you remember in 6 months?)
├── Not documented (how was this set up?)
├── Not reviewable (can't code review a click)
├── Not testable (hope it works!)
└── Drift (manual changes accumulate)
```

### 1.2 The Snowflake Server Problem

**Snowflake Servers:**
Each server is unique, with manual tweaks over time:
```
Server A:                Server B:                 Server C:
├── Version 2.3.1        ├── Version 2.3.0         ├── Version 2.3.1
├── Config change X      ├── Config change X       ├── Config change Y
├── Hotfix patch         ├── Missing patch         ├── Hotfix patch
└── Manual file edit     └── Different config      └── Different user
```

**The Consequence:**
"It works on Server A but not Server B. Why?"

### 1.3 The Documentation Lie

```
Documentation says:        Reality:
┌────────────────────┐    ┌────────────────────┐
│ Instance type:     │    │ Instance type:     │
│   t3.medium        │    │   t3.large         │ ← Changed last month
│                    │    │                    │
│ Disk size: 100 GB  │    │ Disk size: 200 GB  │ ← Expanded for logs
│                    │    │                    │
│ Security group:    │    │ Security group:    │
│   web-server       │    │   web-server-v2    │ ← "Temporary" fix
└────────────────────┘    └────────────────────┘

Documentation becomes stale immediately.
```

---

## Section 2: Infrastructure as Code Fundamentals

### 2.1 What Is Infrastructure as Code?

**Definition:**
Managing and provisioning infrastructure through machine-readable definition files, rather than manual processes.

**The Core Idea:**
```
Traditional:                    Infrastructure as Code:
Admin → Console → Infrastructure   Code → Tool → Infrastructure
(manual, variable)                 (automated, consistent)
```

### 2.2 Benefits of IaC

**1. Version Control:**
```
Git History:
├── Initial VPC setup (commit abc123)
├── Added staging environment (commit def456)
├── Increased instance size (commit ghi789)
│   └── Can see WHO changed it, WHEN, and WHY
├── Rolled back instance size (revert ghi789)
│   └── Easy to undo changes
└── Current state (HEAD)
```

**2. Repeatability:**
```
Same code = Same infrastructure

Dev Environment:     Staging Environment:    Production:
┌────────────────┐   ┌────────────────┐     ┌────────────────┐
│ From template  │   │ From template  │     │ From template  │
│   (small)      │   │   (medium)     │     │   (large)      │
└────────────────┘   └────────────────┘     └────────────────┘

Same structure, different parameters
```

**3. Code Review:**
```
Pull Request #42: "Add Redis cache cluster"

+  resource "aws_elasticache_cluster" "cache" {
+    cluster_id           = "app-cache"
+    engine               = "redis"
+    node_type            = "cache.t3.medium"
+    num_cache_nodes      = 2
+    port                 = 6379
+  }

Reviewer: "Should we use cache.t3.small for dev?"
Author: "Good point, let me parameterize that..."
```

**4. Testing:**
```
Before applying to production:
├── Lint the code (syntax errors)
├── Validate (will it work?)
├── Plan (what will change?)
├── Apply to dev (test it)
├── Apply to staging (test more)
└── Apply to production (confidence!)
```

**5. Disaster Recovery:**
```
Disaster: Region goes down

Traditional:                    IaC:
├── Panic                       ├── Run terraform apply
├── Try to remember setup       │   in new region
├── Click through console       ├── Wait 15 minutes
├── Miss configurations         └── Done
├── Debug for hours
└── Maybe recover
```

### 2.3 Declarative vs. Imperative

**Imperative (How):**
```bash
# Script telling HOW to create infrastructure
aws ec2 run-instances --image-id ami-123 --instance-type t3.micro
aws ec2 wait instance-running --instance-ids $INSTANCE_ID
aws ec2 create-tags --resources $INSTANCE_ID --tags Key=Name,Value=web
aws ec2 allocate-address
aws ec2 associate-address --instance-id $INSTANCE_ID --allocation-id $ALLOC_ID
```

**Declarative (What):**
```hcl
# Definition of WHAT you want
resource "aws_instance" "web" {
  ami           = "ami-123"
  instance_type = "t3.micro"

  tags = {
    Name = "web"
  }
}

resource "aws_eip" "web" {
  instance = aws_instance.web.id
}
```

**Why Declarative Wins:**
```
Declarative advantages:
├── Describe desired end state
├── Tool figures out how to get there
├── Handles ordering automatically
├── Idempotent (run multiple times, same result)
└── Easier to understand
```

### 2.4 State Management

**The State Problem:**
How does the tool know what exists vs. what you want?

**State File:**
```
Your Code (desired):        State File (actual):        AWS (reality):
┌────────────────────┐     ┌────────────────────┐     ┌────────────────────┐
│ 3 EC2 instances    │     │ 2 EC2 instances    │     │ 2 EC2 instances    │
│ 1 RDS database     │     │ 1 RDS database     │     │ 1 RDS database     │
└────────────────────┘     └────────────────────┘     └────────────────────┘
         │                          │
         └──────────┬───────────────┘
                    │
              Tool compares:
              "Need to create 1 more EC2 instance"
```

**State Storage:**
```
Local State:                    Remote State:
├── Stored on your machine      ├── Stored in S3, Cloud Storage, etc.
├── Not shared                  ├── Shared across team
├── Easy to lose                ├── Backed up
└── Single user only            └── Supports collaboration
```

---

## Section 3: AWS CloudFormation

### 3.1 What Is CloudFormation?

**Definition:**
AWS's native Infrastructure as Code service. Define AWS resources in JSON or YAML templates.

**Mental Model:**
"A recipe for AWS infrastructure."

### 3.2 Template Structure

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'My application infrastructure'

Parameters:
  Environment:
    Type: String
    AllowedValues:
      - dev
      - staging
      - prod

Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-vpc'

  MyInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t3.micro
      ImageId: ami-0123456789abcdef0
      SubnetId: !Ref MySubnet
      Tags:
        - Key: Environment
          Value: !Ref Environment

Outputs:
  InstanceId:
    Description: 'The instance ID'
    Value: !Ref MyInstance
    Export:
      Name: !Sub '${Environment}-instance-id'
```

### 3.3 Key Concepts

**Resources:**
```yaml
Resources:
  LogicalName:           # Your reference name
    Type: AWS::Service::Resource
    Properties:
      Property1: Value1
      Property2: Value2
```

**Parameters:**
```yaml
Parameters:
  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues:
      - t3.micro
      - t3.small
      - t3.medium
    Description: 'EC2 instance type'
```

**Intrinsic Functions:**
```yaml
# Reference another resource
!Ref MyVPC

# Get an attribute
!GetAtt MyInstance.PrivateIp

# Substitute variables
!Sub 'arn:aws:s3:::${BucketName}/*'

# Conditional
!If [IsProd, m5.large, t3.micro]

# Join strings
!Join ['-', [!Ref Environment, 'app', 'bucket']]
```

**Outputs:**
```yaml
Outputs:
  VPCId:
    Description: 'VPC ID'
    Value: !Ref MyVPC
    Export:
      Name: !Sub '${AWS::StackName}-VPCId'
```

### 3.4 Stacks and Nested Stacks

**Stack:**
A single CloudFormation template deployed as a unit.

**Nested Stacks:**
```yaml
# Parent template
Resources:
  NetworkStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/bucket/network.yaml
      Parameters:
        Environment: !Ref Environment

  AppStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/bucket/app.yaml
      Parameters:
        VPCId: !GetAtt NetworkStack.Outputs.VPCId
```

**Stack Organization:**
```
Production Environment:
├── network-stack
│   ├── VPC
│   ├── Subnets
│   └── Security Groups
├── database-stack
│   ├── RDS Instance
│   └── Parameter Groups
└── application-stack
    ├── ECS Cluster
    ├── Services
    └── Load Balancer
```

### 3.5 CloudFormation Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│                   CloudFormation Workflow                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                    Write/Update Template
                              │
                              ▼
              ┌───────────────────────────────┐
              │     Validate Template          │
              │   aws cloudformation validate  │
              └───────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │     Create Change Set          │
              │   (Preview what will change)   │
              └───────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │     Review Changes             │
              │   (Human approval)             │
              └───────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │     Execute Change Set         │
              │   (Apply changes)              │
              └───────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │     Monitor Stack Events       │
              │   (Watch progress/errors)      │
              └───────────────────────────────┘
```

### 3.6 CloudFormation Limitations

```
Limitations:
├── AWS-only (can't manage other clouds)
├── Verbose (lots of YAML)
├── Limited programming constructs
├── Drift detection is separate feature
└── State managed by AWS (less control)
```

---

## Section 4: Terraform

### 4.1 What Is Terraform?

**Definition:**
HashiCorp's open-source IaC tool. Multi-cloud, uses HCL (HashiCorp Configuration Language).

**Why Terraform:**
```
CloudFormation:              Terraform:
├── AWS-only                 ├── Multi-cloud
├── YAML/JSON                ├── HCL (more readable)
├── State in AWS             ├── State you control
├── Limited logic            ├── More programming features
└── AWS maintains            └── HashiCorp + community
```

### 4.2 Terraform Basics

**Provider Configuration:**
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
  region = "us-east-1"
}
```

**Resource Definition:**
```hcl
resource "aws_instance" "web" {
  ami           = "ami-0123456789abcdef0"
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.main.id

  tags = {
    Name        = "web-server"
    Environment = var.environment
  }
}
```

**Variables:**
```hcl
# variables.tf
variable "environment" {
  type        = string
  description = "Environment name"
  default     = "dev"
}

variable "instance_types" {
  type = map(string)
  default = {
    dev     = "t3.micro"
    staging = "t3.small"
    prod    = "t3.medium"
  }
}

# Usage
resource "aws_instance" "web" {
  instance_type = var.instance_types[var.environment]
}
```

**Outputs:**
```hcl
output "instance_ip" {
  value       = aws_instance.web.public_ip
  description = "The public IP of the web server"
}

output "instance_id" {
  value = aws_instance.web.id
}
```

### 4.3 Terraform Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│                     Terraform Workflow                           │
└─────────────────────────────────────────────────────────────────┘

    terraform init          terraform plan          terraform apply
         │                       │                        │
         ▼                       ▼                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ Initialize      │    │ Compare desired │    │ Execute the     │
│ - Download      │    │ vs actual state │    │ planned changes │
│   providers     │    │                 │    │                 │
│ - Set up backend│    │ Show what will  │    │ Update state    │
│ - Initialize    │    │ change          │    │ file            │
│   modules       │    │                 │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘

                       terraform destroy
                              │
                              ▼
                    ┌─────────────────┐
                    │ Remove all      │
                    │ managed         │
                    │ resources       │
                    └─────────────────┘
```

### 4.4 State Management

**Remote State (Recommended):**
```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"  # For state locking
  }
}
```

**State Locking:**
```
Without locking:               With locking:
┌─────────┐   ┌─────────┐     ┌─────────┐   ┌─────────┐
│ Alice   │   │  Bob    │     │ Alice   │   │  Bob    │
│ apply   │   │ apply   │     │ apply   │   │ apply   │
└────┬────┘   └────┬────┘     └────┬────┘   └────┬────┘
     │             │               │             │
     ▼             ▼               ▼             ▼
┌────────────────────┐         ┌─────────────┐ ┌─────────────┐
│    State File      │         │Lock acquired│ │ Lock denied │
│ (corrupted!)       │         │ Apply runs  │ │ Wait...     │
└────────────────────┘         └─────────────┘ └─────────────┘
```

### 4.5 Modules

**What Are Modules:**
Reusable infrastructure components.

**Module Structure:**
```
modules/
└── vpc/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    └── README.md
```

**Module Definition:**
```hcl
# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block = var.cidr_block

  tags = {
    Name = var.name
  }
}

resource "aws_subnet" "public" {
  count             = length(var.public_subnets)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnets[count.index]
  availability_zone = var.azs[count.index]
}

# modules/vpc/variables.tf
variable "cidr_block" {
  type = string
}

variable "name" {
  type = string
}

variable "public_subnets" {
  type = list(string)
}

variable "azs" {
  type = list(string)
}

# modules/vpc/outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}
```

**Using Modules:**
```hcl
module "vpc" {
  source = "./modules/vpc"

  name           = "production"
  cidr_block     = "10.0.0.0/16"
  public_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  azs            = ["us-east-1a", "us-east-1b"]
}

module "vpc_staging" {
  source = "./modules/vpc"

  name           = "staging"
  cidr_block     = "10.1.0.0/16"
  public_subnets = ["10.1.1.0/24", "10.1.2.0/24"]
  azs            = ["us-east-1a", "us-east-1b"]
}

# Reference module outputs
resource "aws_instance" "web" {
  subnet_id = module.vpc.public_subnet_ids[0]
}
```

### 4.6 Data Sources

**Reading Existing Resources:**
```hcl
# Look up existing VPC
data "aws_vpc" "existing" {
  filter {
    name   = "tag:Name"
    values = ["production-vpc"]
  }
}

# Look up latest AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Use the data
resource "aws_instance" "web" {
  ami       = data.aws_ami.amazon_linux.id
  vpc_id    = data.aws_vpc.existing.id
}
```

### 4.7 Terraform vs CloudFormation

```
┌─────────────────────┬────────────────────┬─────────────────────┐
│ Aspect              │ CloudFormation     │ Terraform           │
├─────────────────────┼────────────────────┼─────────────────────┤
│ Cloud Support       │ AWS only           │ Multi-cloud         │
│ Language            │ YAML/JSON          │ HCL                 │
│ State Management    │ AWS-managed        │ You manage          │
│ Ecosystem           │ AWS only           │ Large provider list │
│ Learning Curve      │ Moderate           │ Moderate            │
│ Pricing             │ Free               │ Free (OSS)          │
│ Drift Detection     │ Yes (separate)     │ terraform plan      │
│ Modularity          │ Nested stacks      │ Modules             │
│ Programming Logic   │ Limited            │ Rich                │
└─────────────────────┴────────────────────┴─────────────────────┘
```

---

## Section 5: IaC Best Practices

### 5.1 Project Structure

**Recommended Layout:**
```
infrastructure/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── staging/
│   │   └── ...
│   └── prod/
│       └── ...
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── ecs-cluster/
│   │   └── ...
│   └── rds/
│       └── ...
└── README.md
```

### 5.2 Environment Separation

**Using Workspaces (Simple):**
```bash
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

terraform workspace select prod
terraform apply
```

**Using Separate State Files (Recommended for Production):**
```hcl
# environments/prod/backend.tf
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"  # Different key per env
    region = "us-east-1"
  }
}
```

### 5.3 Secrets Management

**Never Do This:**
```hcl
# BAD: Secrets in code
resource "aws_db_instance" "db" {
  password = "super-secret-password"  # Will be in state file!
}
```

**Better Approaches:**
```hcl
# Option 1: Variables (still needs secure input)
variable "db_password" {
  type      = string
  sensitive = true
}

resource "aws_db_instance" "db" {
  password = var.db_password
}

# Option 2: AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/database/password"
}

resource "aws_db_instance" "db" {
  password = data.aws_secretsmanager_secret_version.db.secret_string
}

# Option 3: Let AWS generate it
resource "random_password" "db" {
  length  = 32
  special = true
}

resource "aws_secretsmanager_secret_version" "db" {
  secret_id     = aws_secretsmanager_secret.db.id
  secret_string = random_password.db.result
}
```

### 5.4 Tagging Strategy

**Consistent Tags:**
```hcl
locals {
  common_tags = {
    Project     = "my-app"
    Environment = var.environment
    ManagedBy   = "terraform"
    Owner       = "platform-team"
    CostCenter  = "engineering"
  }
}

resource "aws_instance" "web" {
  # ...
  tags = merge(local.common_tags, {
    Name = "web-server"
    Role = "web"
  })
}
```

### 5.5 Code Review Checklist

```
Terraform PR Review Checklist:
├── terraform plan output included?
├── No secrets in code?
├── State file protected?
├── Resources tagged properly?
├── Least privilege IAM?
├── Encryption enabled where applicable?
├── Modules used for reusability?
├── Variables have descriptions?
├── Outputs documented?
└── README updated?
```

### 5.6 Testing Infrastructure Code

**Validation:**
```bash
# Syntax check
terraform validate

# Format check
terraform fmt -check

# Plan (without applying)
terraform plan
```

**Policy as Code (OPA/Sentinel):**
```rego
# policy.rego - Require encryption on S3 buckets
deny[msg] {
  resource := input.resource.aws_s3_bucket[name]
  not resource.server_side_encryption_configuration
  msg := sprintf("S3 bucket '%s' must have encryption enabled", [name])
}
```

**Integration Testing:**
```
Testing pyramid for IaC:
┌─────────────────────────────────────────┐
│          End-to-End Tests               │  ← Slow, expensive
│    (Deploy and verify full stack)       │
├─────────────────────────────────────────┤
│        Integration Tests                │
│  (Deploy modules, verify behavior)      │
├─────────────────────────────────────────┤
│          Unit Tests                     │
│    (Validate, lint, policy checks)      │  ← Fast, cheap
└─────────────────────────────────────────┘
```

---

## Section 6: Managing Drift

### 6.1 What Is Drift?

**Definition:**
When actual infrastructure differs from what's defined in code.

**How It Happens:**
```
Causes of Drift:
├── Manual console changes ("just this once")
├── Direct CLI commands
├── Automated processes outside IaC
├── Another team's changes
└── AWS making changes (deprecations, etc.)
```

### 6.2 Detecting Drift

**Terraform:**
```bash
# Shows differences between state and actual
terraform plan

# Example output:
# ~ aws_instance.web
#     instance_type: "t3.micro" => "t3.small"  # Changed manually!
```

**CloudFormation:**
```bash
# Detect drift
aws cloudformation detect-stack-drift --stack-name my-stack
aws cloudformation describe-stack-resource-drifts --stack-name my-stack
```

### 6.3 Preventing Drift

**1. Lock Down Manual Access:**
```
├── Restrict console/CLI access to production
├── Use IAM policies to limit who can change what
└── Require IaC changes go through PR process
```

**2. Regular Drift Checks:**
```bash
# Run in CI on a schedule
terraform plan -detailed-exitcode
# Exit code 2 = changes detected
```

**3. Automated Remediation:**
```bash
# Re-apply to fix drift (be careful!)
terraform apply
```

### 6.4 Importing Existing Resources

**When You Need to Import:**
```
Scenarios:
├── Adopting IaC for existing infrastructure
├── Resource created manually that should be managed
└── Migrating from another IaC tool
```

**Terraform Import:**
```bash
# 1. Write the resource configuration
resource "aws_instance" "existing" {
  # Will be populated after import
}

# 2. Import the resource
terraform import aws_instance.existing i-1234567890abcdef0

# 3. Run plan to see what's different
terraform plan

# 4. Update config to match reality (or apply to change reality)
```

---

## Section 7: IaC in CI/CD

### 7.1 Infrastructure Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                  Infrastructure CI/CD Pipeline                   │
└─────────────────────────────────────────────────────────────────┘

  Push to Branch        Pull Request           Merge to Main
       │                     │                      │
       ▼                     ▼                      ▼
┌─────────────┐      ┌─────────────────┐    ┌─────────────────┐
│ terraform   │      │ terraform plan  │    │ terraform apply │
│ validate    │      │ (post as PR     │    │ (auto-apply or  │
│ terraform   │      │  comment)       │    │  manual approve)│
│ fmt -check  │      │                 │    │                 │
│ tflint      │      │ Human review    │    │                 │
└─────────────┘      └─────────────────┘    └─────────────────┘
```

### 7.2 Example GitHub Actions Workflow

```yaml
name: Terraform

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  terraform:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.6.0

      - name: Terraform Init
        run: terraform init

      - name: Terraform Format Check
        run: terraform fmt -check

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        id: plan
        run: terraform plan -no-color
        continue-on-error: true

      - name: Post Plan to PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const output = `#### Terraform Plan
            \`\`\`
            ${{ steps.plan.outputs.stdout }}
            \`\`\``;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve
```

### 7.3 GitOps for Infrastructure

**GitOps Principle:**
Git is the source of truth. Changes only happen through Git.

```
┌─────────────────────────────────────────────────────────────────┐
│                      GitOps Flow                                 │
└─────────────────────────────────────────────────────────────────┘

Developer                  Git                    Infrastructure
┌─────────┐           ┌─────────┐              ┌─────────────────┐
│ Change  │──push────►│ Main    │──trigger────►│ CI/CD runs      │
│ .tf     │           │ Branch  │              │ terraform apply │
│ files   │           │         │              │                 │
└─────────┘           └─────────┘              └─────────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │ Full audit  │
                    │ trail in    │
                    │ Git history │
                    └─────────────┘
```

---

## Section 8: Real-World Patterns

### 8.1 Blue-Green Infrastructure

```hcl
variable "active_color" {
  type    = string
  default = "blue"
}

module "blue" {
  source = "./modules/environment"
  name   = "blue"
  # ...
}

module "green" {
  source = "./modules/environment"
  name   = "green"
  # ...
}

# Switch traffic by changing the variable
resource "aws_route53_record" "app" {
  name    = "app.example.com"
  type    = "A"

  alias {
    name = var.active_color == "blue" ?
           module.blue.lb_dns :
           module.green.lb_dns
    # ...
  }
}
```

### 8.2 Feature Flags in Infrastructure

```hcl
variable "enable_redis_cache" {
  type    = bool
  default = false
}

resource "aws_elasticache_cluster" "cache" {
  count = var.enable_redis_cache ? 1 : 0

  cluster_id      = "app-cache"
  engine          = "redis"
  node_type       = "cache.t3.micro"
  num_cache_nodes = 1
}

# Reference conditionally created resource
output "cache_endpoint" {
  value = var.enable_redis_cache ?
          aws_elasticache_cluster.cache[0].cache_nodes[0].address :
          null
}
```

### 8.3 Multi-Region Deployment

```hcl
provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu_west"
  region = "eu-west-1"
}

module "us_east" {
  source = "./modules/regional-app"
  providers = {
    aws = aws.us_east
  }

  region = "us-east-1"
}

module "eu_west" {
  source = "./modules/regional-app"
  providers = {
    aws = aws.eu_west
  }

  region = "eu-west-1"
}
```

---

## Summary: Infrastructure as Code

**Why IaC Matters:**
- Version control for infrastructure
- Repeatability and consistency
- Code review and collaboration
- Automated testing and deployment
- Disaster recovery

**Key Concepts:**
```
├── Declarative: Describe what, not how
├── State: Track what exists vs what's desired
├── Idempotency: Same input = same output
├── Modularity: Reusable components
└── Immutability: Replace, don't modify
```

**Tools:**
```
CloudFormation: AWS-native, tightly integrated
Terraform: Multi-cloud, larger ecosystem
Both: Mature, production-ready choices
```

**Best Practices:**
```
├── Use remote state with locking
├── Structure code by environment
├── Never commit secrets
├── Tag everything consistently
├── Review plans before applying
├── Test in lower environments first
└── Detect and prevent drift
```

---

## What's Next?

You now understand how to define and manage infrastructure as code. Chapter 18 explores cloud architecture patterns—the proven designs for building resilient, scalable applications in the cloud.
