# Infrastructure

All Terraform configurations for the investment order service.

## Structure

```
terraform/
├── compute/        # EC2, ASG, ALB
├── database/       # RDS, RDS Proxy, Read Replicas
├── cache/          # ElastiCache Redis
├── network/        # VPC, subnets, security groups, NAT
├── observability/  # CloudWatch dashboards and alarms
└── security/       # WAF, KMS keys, IAM roles
```

## Usage

```bash
cd infrastructure/terraform
terraform init
terraform plan  -var-file="envs/production.tfvars"
terraform apply -var-file="envs/production.tfvars"
```

## State Backend

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

> Never commit `production.tfvars` — use CI/CD secrets instead.
> Never run `terraform apply` on production without a peer-reviewed plan.
