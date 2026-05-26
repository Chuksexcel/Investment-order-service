# Investment Order Service

Critical microservice handling all investment buy/sell/cancel operations.

## Architecture

See [ARCHITECTURE.md](./ARCHITECTURE.md) for the full deployment architecture including:

- SLO definitions and error budget policy
- EC2 compute + Auto Scaling configuration
- RDS PostgreSQL Multi-AZ + RDS Proxy + Read Replicas
- ElastiCache Redis cluster with automatic failover
- Blue/green deployment strategy
- Backup and point-in-time recovery
- Observability and alerting stack
- Security hardening (WAF, VPC isolation, Secrets Manager)
- Incident runbooks

## Quick Reference

| Environment | Region | RTO | RPO |
|-------------|--------|-----|-----|
| Production | us-east-1 (primary) | 5 min | 30 sec |
| DR | us-west-2 (standby) | 30 min | 5 min |
| Staging | us-east-1 | Best effort | N/A |

## SLOs at a Glance

| Signal | Target |
|--------|--------|
| Availability | 99.95% / month |
| Order P99 Latency | < 500ms |
| Order P50 Latency | < 100ms |
| Error Rate | < 0.1% |
| Durability | 99.999999% |

## Repository Structure

```
.
├── ARCHITECTURE.md          ← Full architecture design document
├── README.md
├── app/                     ← Django application code
├── infrastructure/
│   ├── README.md
│   └── terraform/           ← All AWS infrastructure as code
└── .github/
    └── workflows/
        └── deploy.yml       ← CI/CD pipeline
```

## Contributing

All changes to this service require:

1. Code review from a second engineer
2. 85% test coverage maintained
3. Passing security scan (bandit + safety)
4. Manual approval gate before production deploy
5. Pre-deploy RDS snapshot (automated in CI)

## On-Call

P1 incidents: page via PagerDuty → `investment-orders-oncall` schedule.
Post-mortems required for any incident consuming > 50% of monthly error budget.
