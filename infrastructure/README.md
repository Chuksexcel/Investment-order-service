# Investment Order Service — Infrastructure

This directory contains all Terraform configurations for the investment order service.

## Directory Structure

```
infrastructure/
├── terraform/
│   ├── compute/         # EC2, ASG, ALB
│   ├── database/        # RDS, RDS Proxy, Read Replicas
│   ├── cache/           # ElastiCache Redis
│   ├── network/         # VPC, subnets, security groups, NAT
│   ├── observability/   # CloudWatch, alarms, dashboards
│   └── security/        # WAF, KMS keys, IAM roles
└── README.md
```

## Prerequisites

- Terraform >= 1.5
- AWS CLI configured with appropriate credentials
- S3 bucket for Terraform state (configure in `backend.tf`)

## Usage

```bash
cd infrastructure/terraform
terraform init
terraform plan -var-file="envs/production.tfvars"
terraform apply -var-file="envs/production.tfvars"
```

## State Management

Remote state is stored in S3 with DynamoDB locking:

```hcl
terraform {
  backend "s3" {
    bucket         = "investment-orders-tfstate"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "investment-orders-tfstate-lock"
    encrypt        = true
  }
}
```

## Environments

- `envs/staging.tfvars` — staging environment variables
- `envs/production.tfvars` — production environment variables

**Never commit `production.tfvars` to version control.** Use CI/CD secrets.
