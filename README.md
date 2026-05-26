# Investment Order Service

Critical microservice handling all investment buy/sell/cancel operations.

## Architecture

See [ARCHITECTURE.md](./ARCHITECTURE.md) for the full deployment architecture including:

- SLO definitions and error budget policy
- Multi-AZ EC2 Auto Scaling Group with blue/green deployment
- RDS PostgreSQL Multi-AZ + RDS Proxy + 2× Read Replicas
- ElastiCache Redis cluster (3-node) with automatic failover
- Circuit breaker, idempotency pattern, retry logic
- 35-day RDS backup retention + cross-region DR copy
- WAF + VPC isolation + Secrets Manager security hardening
- GitHub Actions CI/CD with canary deploy + auto-rollback
- Incident runbooks for RDS failover, Redis failure, bad deploy

## SLOs at a Glance

| Signal | Target |
|--------|--------|
| Availability | 99.95% / month |
| Order Submit P99 Latency | < 500ms |
| Order Submit P50 Latency | < 100ms |
| Error Rate | < 0.1% |
| Durability | 99.999999% (8 nines) |
| RTO | < 5 minutes |
| RPO | < 30 seconds |

## Quick Reference

| Environment | Region | RTO | RPO |
|-------------|--------|-----|-----|
| Production | us-east-1 (primary) | 5 min | 30 sec |
| DR | us-west-2 (standby) | 30 min | 5 min |
| Staging | us-east-1 | Best effort | N/A |

## Repository Structure

```
.
├── ARCHITECTURE.md               ← Full architecture design document
├── README.md
├── infrastructure/
│   ├── README.md
│   └── terraform/
│       ├── compute/              ← EC2, ASG, ALB
│       ├── database/             ← RDS, RDS Proxy, Read Replicas
│       ├── cache/                ← ElastiCache Redis
│       ├── network/              ← VPC, subnets, security groups
│       ├── observability/        ← CloudWatch dashboards, alarms
│       └── security/             ← WAF, KMS, IAM roles
└── .github/
    └── workflows/
        └── deploy.yml            ← CI/CD pipeline
```

## Contributing

All changes require:
1. Code review from a second engineer
2. 85% test coverage maintained
3. Passing security scan (bandit + safety)
4. Manual approval gate before production deploy
5. Pre-deploy RDS snapshot (automated in CI/CD)

## On-Call

P1 incidents → PagerDuty `investment-orders-oncall` schedule.
Post-mortems required for any incident consuming > 50% of the monthly error budget.
