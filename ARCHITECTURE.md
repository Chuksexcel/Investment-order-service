# Investment Order Service — Deployment Architecture

> **Classification:** Internal Engineering  
> **Version:** 1.0  
> **Owner:** Platform Engineering  
> **Last Updated:** May 2026

---

## Table of Contents

1. [Overview](#overview)
2. [SLO Definitions](#slo-definitions)
3. [Compute Architecture](#compute-architecture)
4. [Database Architecture](#database-architecture)
5. [Caching Layer (Redis)](#caching-layer)
6. [Failover Strategy](#failover-strategy)
7. [Backup & Recovery](#backup--recovery)
8. [Deployment Pipeline](#deployment-pipeline)
9. [Observability Stack](#observability-stack)
10. [Security Hardening](#security-hardening)
11. [Runbooks](#runbooks)

---

## Overview

The Investment Order Service is the critical path for all buy/sell/cancel operations on investment instruments. Every user-facing transaction flows through this service. Downtime = direct financial loss and regulatory exposure.

### Architecture Principles

- **No single point of failure** at any layer
- **Degrade gracefully** — read operations survive write failures
- **Immutable audit trail** — every order state transition is persisted before acknowledged
- **Defence in depth** — security controls at network, application, and data layers

### High-Level Topology

```
                          ┌─────────────────────────────────────┐
                          │           Route 53 (DNS)            │
                          │     Health-check failover routing    │
                          └──────────────┬──────────────────────┘
                                         │
                          ┌──────────────▼──────────────────────┐
                          │     Application Load Balancer        │
                          │   (Multi-AZ, WAF attached, TLS)     │
                          └──────┬──────────────┬───────────────┘
                                 │              │
                   ┌─────────────▼──┐    ┌──────▼────────────────┐
                   │  EC2 ASG       │    │  EC2 ASG               │
                   │  (us-east-1a)  │    │  (us-east-1b)          │
                   │  t3.xlarge ×2  │    │  t3.xlarge ×2          │
                   └─────────────┬──┘    └──────┬─────────────────┘
                                 │              │
                    ┌────────────▼──────────────▼──────────────┐
                    │            Private Subnet                 │
                    │                                           │
                    │  ┌─────────────────┐  ┌───────────────┐  │
                    │  │ RDS PostgreSQL  │  │ ElastiCache   │  │
                    │  │ Multi-AZ        │  │ Redis Cluster │  │
                    │  │ (Primary +      │  │ (3-node w/    │  │
                    │  │  Standby)       │  │  replication) │  │
                    │  └────────┬────────┘  └───────────────┘  │
                    │           │                               │
                    │  ┌────────▼────────┐                     │
                    │  │ RDS Read        │                     │
                    │  │ Replicas (×2)   │                     │
                    │  └─────────────────┘                     │
                    └──────────────────────────────────────────┘
```

---

## SLO Definitions

### Why These Numbers?

Investment orders are financial transactions with regulatory obligations. Every SLO target below was derived from three inputs: (1) user tolerance for financial workflow latency, (2) regulatory guidance on trade execution timing, and (3) the cost of the infrastructure required to achieve it.

### SLO Table

| Signal | SLO | Rationale |
|--------|-----|-----------|
| **Availability** | 99.95% per month | ≤ 21.9 min downtime/month. Regulators expect near-continuous availability during market hours. 99.99% would cost ~4× more with marginal real-world benefit. |
| **Order Submission Latency (p99)** | < 500ms | User research: >500ms feels "broken" for a financial action. Orders must be confirmed before WebSocket push to client. |
| **Order Submission Latency (p50)** | < 100ms | Median should feel instant. Breaching this signals a systemic DB or cache issue before p99 spikes. |
| **Order Status Read Latency (p99)** | < 200ms | Reads served from Redis replica; 200ms is generous. Tight SLO catches cache invalidation bugs. |
| **Error Rate** | < 0.1% of requests | 1 in 1000 requests failing is visible to active traders. Anything above 0.1% gets a page. |
| **Durability** | 99.999999% (8 nines) | A lost order = regulatory incident + financial liability. Achieved via synchronous RDS write + WAL archiving to S3 before ACK. |
| **RTO** (Recovery Time Objective) | < 5 minutes | RDS Multi-AZ automatic failover: ~60–120s. Service restart via ASG: ~90s. Combined target: 5 min. |
| **RPO** (Recovery Point Objective) | < 30 seconds | RDS synchronous standby = 0 committed data loss. 30s accounts for in-flight transactions during a crash. |

### Error Budget Policy

With 99.95% availability, the monthly error budget is **21.9 minutes**.

- **> 50% budget consumed** in a single incident → mandatory post-mortem required
- **> 80% budget consumed** in a month → freeze all non-critical deploys for the remainder of the month
- **Budget exhausted** → no deploys until next month; incident review with engineering leadership

---

## Compute Architecture

### EC2 Configuration

**Instance type:** `t3.xlarge` (4 vCPU, 16 GB RAM)

Rationale: Investment order processing is CPU-bound during signature verification and validation, not memory-bound. `t3.xlarge` gives burst capacity without the premium of compute-optimised instances. Benchmark first; upgrade to `c6i.xlarge` if CPU steal > 5%.

**Auto Scaling Group settings:**

```hcl
# terraform/asg.tf
resource "aws_autoscaling_group" "investment_orders" {
  name                      = "investment-orders-asg"
  min_size                  = 4        # 2 per AZ minimum — never below this
  max_size                  = 20
  desired_capacity          = 4
  health_check_grace_period = 120      # seconds before ASG acts on health check
  health_check_type         = "ELB"    # not EC2 — use ALB health checks

  # Scale out fast, scale in slow — asymmetric policy for financial services
  # You want capacity instantly; you don't want to kill instances during a spike
}

resource "aws_autoscaling_policy" "scale_out" {
  name                   = "orders-scale-out"
  scaling_adjustment     = 2           # add 2 instances at a time
  adjustment_type        = "ChangeInCapacity"
  cooldown               = 60          # 60s between scale-out events
}

resource "aws_autoscaling_policy" "scale_in" {
  name                   = "orders-scale-in"
  scaling_adjustment     = -1          # remove 1 instance at a time
  adjustment_type        = "ChangeInCapacity"
  cooldown               = 300         # 5 min between scale-in events — conservative
}
```

**ALB health check:**

```hcl
resource "aws_lb_target_group" "investment_orders" {
  health_check {
    path                = "/health/live"
    interval            = 10           # check every 10 seconds
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 3            # 3 failures = 30 seconds before removal
    matcher             = "200"
  }
}
```

### Health Endpoint Contract

```python
# GET /health/live  — used by ALB (liveness only)
# Returns 200 if process is alive

# GET /health/ready — used by deployment checks (readiness)
# Returns 200 only if DB connection pool, Redis, and all dependencies are healthy
# Returns 503 with body {"status": "degraded", "failing": ["rds"]} if any dep is down

# GET /health/deep  — internal only, not exposed to ALB
# Performs a test DB write and read to verify full stack
```

### Deployment Strategy: Blue/Green

Never rolling-deploy a financial service. Use blue/green to ensure zero in-flight transaction corruption.

```
Step 1: Provision "green" ASG alongside existing "blue" ASG
Step 2: Run smoke tests and integration tests against green
Step 3: Shift 10% traffic to green via ALB weighted target groups
Step 4: Monitor error rate and latency for 5 minutes
Step 5: If healthy → shift 100% to green
Step 6: Keep blue warm for 15 minutes, then terminate
Step 7: If unhealthy at any step → shift 100% back to blue instantly
```

---

## Database Architecture

### RDS PostgreSQL — Multi-AZ

```hcl
resource "aws_db_instance" "investment_orders_primary" {
  identifier              = "investment-orders-primary"
  engine                  = "postgres"
  engine_version          = "15.4"
  instance_class          = "db.r6g.xlarge"  # memory-optimised for financial workloads
  allocated_storage       = 500              # GB
  max_allocated_storage   = 2000             # autoscaling ceiling
  storage_type            = "io1"
  iops                    = 3000             # provisioned IOPS — consistent latency

  multi_az                = true             # synchronous standby in second AZ
  deletion_protection     = true             # cannot be deleted without disabling this

  backup_retention_period = 35               # 35 days of automated snapshots
  backup_window           = "02:00-03:00"    # 2 AM UTC — lowest traffic
  maintenance_window      = "Mon:03:00-Mon:04:00"

  # Critical: parameter group tuning
  parameter_group_name = aws_db_parameter_group.orders.name

  # Encryption at rest
  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn

  # Audit logging
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]
}
```

**Key PostgreSQL parameters:**

```sql
-- Connection pooling (via RDS proxy — see below)
max_connections = 1000

-- Write-ahead log for durability
wal_level = replica
synchronous_commit = on           -- never set to off for financial data
fsync = on                        -- never disable

-- Statement timeout — catch runaway queries before they cause P99 spikes
statement_timeout = 5000          -- 5 seconds max per query

-- Lock timeout — prevent lock convoy
lock_timeout = 2000               -- 2 seconds

-- Slow query logging
log_min_duration_statement = 100  -- log queries > 100ms
```

### RDS Proxy

Connection pool exhaustion is the #1 cause of latency spikes in Django services. RDS Proxy sits between your EC2 fleet and RDS, maintaining a warm pool of authenticated connections.

```hcl
resource "aws_db_proxy" "investment_orders" {
  name                   = "investment-orders-proxy"
  debug_logging          = false
  engine_family          = "POSTGRESQL"
  idle_client_timeout    = 1800    # 30 minutes
  require_tls            = true
  role_arn               = aws_iam_role.rds_proxy.arn

  auth {
    auth_scheme = "SECRETS"
    iam_auth    = "REQUIRED"       # IAM auth — no static passwords
    secret_arn  = aws_secretsmanager_secret.db_credentials.arn
  }

  target_group {
    connection_pool_config {
      connection_borrow_timeout    = 120    # seconds to wait for a connection
      max_connections_percent      = 80     # use 80% of RDS max_connections
      max_idle_connections_percent = 50
    }
  }
}
```

### Read Replicas

```hcl
resource "aws_db_instance" "investment_orders_replica" {
  count                  = 2
  identifier             = "investment-orders-replica-${count.index}"
  replicate_source_db    = aws_db_instance.investment_orders_primary.id
  instance_class         = "db.r6g.large"   # smaller — reads only
  publicly_accessible    = false
  auto_minor_version_upgrade = true
}
```

Route read traffic (order history, portfolio positions) to replicas in Django:

```python
# settings.py
DATABASE_ROUTERS = ['app.routers.OrdersRouter']

# routers.py
class OrdersRouter:
    READ_MODELS = {'Order', 'OrderHistory', 'Position'}

    def db_for_read(self, model, **hints):
        if model.__name__ in self.READ_MODELS:
            return random.choice(['replica_1', 'replica_2'])
        return 'default'

    def db_for_write(self, model, **hints):
        return 'default'   # always write to primary
```

---

## Caching Layer

### ElastiCache Redis — Cluster Mode

```hcl
resource "aws_elasticache_replication_group" "investment_orders" {
  replication_group_id          = "investment-orders-redis"
  description                   = "Investment orders session and rate-limiting cache"

  node_type                     = "cache.r6g.large"
  num_cache_clusters            = 3      # 1 primary + 2 replicas
  automatic_failover_enabled    = true
  multi_az_enabled              = true

  # Encryption
  at_rest_encryption_enabled    = true
  transit_encryption_enabled    = true
  auth_token                    = var.redis_auth_token  # from Secrets Manager

  # Backup
  snapshot_retention_limit      = 7      # 7 days
  snapshot_window               = "03:00-04:00"
}
```

### What Lives in Redis

| Key Pattern | TTL | Purpose |
|-------------|-----|---------|
| `order:status:{order_id}` | 300s | Order status cache for read endpoints |
| `ratelimit:order:{user_id}` | 60s | Per-user order rate limiting |
| `idempotency:{idempotency_key}` | 86400s | Duplicate order prevention (24h) |
| `session:{token}` | 3600s | Auth session tokens |
| `portfolio:{user_id}:positions` | 30s | Portfolio position cache |

### Idempotency Pattern (Critical)

Every order submission must be idempotent. Clients send an `Idempotency-Key` header.

```python
def submit_order(request):
    idem_key = request.headers.get('Idempotency-Key')
    if not idem_key:
        return Response({"error": "Idempotency-Key header required"}, status=400)

    # Check Redis first
    cached = redis.get(f"idempotency:{idem_key}")
    if cached:
        return Response(json.loads(cached), status=200)  # return same response

    # Process order
    order = process_order(request.data)

    # Store result in Redis with 24h TTL
    redis.setex(
        f"idempotency:{idem_key}",
        86400,
        json.dumps(order.to_response())
    )

    return Response(order.to_response(), status=201)
```

---

## Failover Strategy

### Scenario Matrix

| Failure Scenario | Detection Time | Automatic Recovery | Manual Action Needed |
|------------------|---------------|-------------------|---------------------|
| Single EC2 instance fails | 30s (ALB health check) | Yes — ASG replaces | None |
| AZ failure (full AZ outage) | 30–60s | Yes — traffic shifts to healthy AZ | Monitor; verify ASG replaces instances |
| RDS Primary fails | 60–120s | Yes — Multi-AZ automatic failover | Update connection strings if not using RDS Proxy |
| Redis Primary fails | 30–60s | Yes — ElastiCache auto-promotes replica | Monitor; verify replication catches up |
| RDS Proxy fails | 10–30s | Yes — multi-endpoint redundancy | None |
| Full region failure | N/A | **No** — requires manual DR activation | See DR Runbook |

### RDS Failover Behaviour

When Multi-AZ failover triggers:

1. RDS detects primary failure (60–120s)
2. Standby promoted to primary (DNS CNAME updated)
3. RDS Proxy detects new primary endpoint (< 30s reconnect)
4. Service continues; in-flight transactions during the ~90s window may need retry

Application-level retry configuration (critical):

```python
# settings.py — Django database retry config
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': os.environ['RDS_PROXY_ENDPOINT'],  # always use proxy, not RDS direct
        'CONN_MAX_AGE': 60,
        'OPTIONS': {
            'connect_timeout': 5,
            'options': '-c statement_timeout=5000 -c lock_timeout=2000',
        },
    }
}

# Retry decorator for order writes
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type
from django.db import OperationalError

@retry(
    retry=retry_if_exception_type(OperationalError),
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=0.5, max=5),
)
def write_order_to_db(order_data):
    with transaction.atomic():
        order = Order.objects.create(**order_data)
        OrderAuditLog.objects.create(order=order, event='CREATED')
    return order
```

### Circuit Breaker

Prevent cascade failures when a downstream dependency is slow:

```python
# utils/circuit_breaker.py
import pybreaker
import logging

db_breaker = pybreaker.CircuitBreaker(
    fail_max=5,          # open after 5 consecutive failures
    reset_timeout=30,    # try again after 30 seconds
    listeners=[pybreaker.CircuitBreakerListener()]
)

redis_breaker = pybreaker.CircuitBreaker(
    fail_max=3,
    reset_timeout=15,
)

@db_breaker
def fetch_order_from_db(order_id):
    return Order.objects.get(id=order_id)

@redis_breaker
def fetch_order_from_cache(order_id):
    return redis.get(f"order:status:{order_id}")
```

---

## Backup & Recovery

### RDS Backup Strategy

| Backup Type | Frequency | Retention | Location |
|-------------|-----------|-----------|----------|
| Automated RDS snapshots | Daily (02:00 UTC) | 35 days | AWS managed S3 |
| WAL archiving (continuous) | Continuous | 7 days | s3://investment-orders-wal-archive |
| On-demand snapshot (pre-deploy) | Before every production deploy | 90 days | s3://investment-orders-snapshots |
| Cross-region copy | Daily | 35 days | us-west-2 (DR region) |

```bash
# Pre-deploy snapshot (run from CI/CD pipeline)
aws rds create-db-snapshot \
  --db-instance-identifier investment-orders-primary \
  --db-snapshot-identifier "pre-deploy-$(date +%Y%m%d-%H%M%S)" \
  --tags Key=Type,Value=pre-deploy Key=DeployId,Value="$DEPLOY_ID"
```

### Point-in-Time Recovery

RDS with WAL archiving supports PITR to any second within the retention window:

```bash
# Restore to 5 minutes before an incident
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier investment-orders-primary \
  --target-db-instance-identifier investment-orders-recovered \
  --restore-time "2026-05-26T22:55:00Z"
```

### Redis Backup

Redis snapshots (RDB files) taken daily. For the idempotency cache and order status cache, Redis data is **reconstructible** from PostgreSQL — Redis loss is a latency event, not a data loss event.

```python
# On Redis unavailability, degrade gracefully to DB reads
def get_order_status(order_id):
    try:
        cached = redis_breaker.call(fetch_order_from_cache, order_id)
        if cached:
            return json.loads(cached)
    except (pybreaker.CircuitBreakerError, redis.RedisError):
        logger.warning("redis_unavailable_fallback", extra={"order_id": order_id})

    # Fall through to DB
    return Order.objects.get(id=order_id).to_dict()
```

---

## Deployment Pipeline

```
Developer pushes → GitHub PR
                      │
                      ▼
              GitHub Actions CI
              ┌─────────────────────────────────────────┐
              │ 1. Unit tests (pytest)                  │
              │ 2. Integration tests (test DB + Redis)  │
              │ 3. Static analysis (ruff, mypy)         │
              │ 4. Security scan (bandit, safety)       │
              │ 5. Docker image build + push to ECR     │
              └──────────────────┬──────────────────────┘
                                 │ PR merged to main
                                 ▼
              Staging Deploy (automatic)
              ┌─────────────────────────────────────────┐
              │ 1. Pre-deploy RDS snapshot              │
              │ 2. Blue/green deploy to staging         │
              │ 3. Smoke tests against staging          │
              │ 4. Load test (k6): 500 req/s for 5 min │
              └──────────────────┬──────────────────────┘
                                 │ Manual approval gate
                                 ▼
              Production Deploy
              ┌─────────────────────────────────────────┐
              │ 1. Pre-deploy RDS snapshot              │
              │ 2. Blue/green deploy (10% → 100%)       │
              │ 3. Automated rollback trigger if:       │
              │    - Error rate > 0.5% for 2 min       │
              │    - P99 latency > 1s for 2 min        │
              │ 4. Notify #deployments Slack channel    │
              └─────────────────────────────────────────┘
```

---

## Observability Stack

### Metrics (CloudWatch + Prometheus)

```python
# Key metrics to instrument in application code
ORDER_SUBMIT_DURATION = Histogram(
    'order_submit_duration_seconds',
    'Time to process an order submission',
    ['status'],
    buckets=[0.05, 0.1, 0.2, 0.5, 1.0, 2.0, 5.0]
)

ORDER_COUNT = Counter(
    'orders_total',
    'Total orders processed',
    ['type', 'status']  # type: buy/sell; status: success/failed
)

DB_POOL_WAIT = Histogram(
    'db_pool_wait_seconds',
    'Time waiting for a DB connection from pool'
)
```

### Alerting Policy

| Alert | Threshold | Severity | Page? |
|-------|-----------|----------|-------|
| Error rate elevated | > 0.1% for 5 min | P2 | Slack only |
| Error rate critical | > 1% for 2 min | P1 | Page on-call |
| P99 latency | > 500ms for 5 min | P2 | Slack only |
| P99 latency critical | > 2s for 2 min | P1 | Page on-call |
| RDS failover detected | Any | P1 | Page on-call |
| Error budget > 50% consumed | Monthly | P2 | Slack + email |
| Disk usage | > 80% | P2 | Slack only |
| Idempotency collision rate | > 5% | P2 | Investigate replay attacks |

### Structured Logging

```python
import structlog

logger = structlog.get_logger()

def submit_order(request):
    log = logger.bind(
        user_id=request.user.id,
        order_type=request.data['type'],
        trace_id=request.headers.get('X-Trace-ID'),
    )

    t0 = time.perf_counter()
    try:
        order = process_order(request.data)
        log.info("order_submitted",
            order_id=str(order.id),
            duration_ms=round((time.perf_counter() - t0) * 1000, 2),
            status="success"
        )
        return Response(order.to_response(), status=201)
    except Exception as e:
        log.error("order_submission_failed",
            error=str(e),
            duration_ms=round((time.perf_counter() - t0) * 1000, 2),
        )
        raise
```

---

## Security Hardening

### Network Isolation

```
Internet → WAF → ALB (public subnet)
                  │
                  ▼
              EC2 ASG (private subnet, no public IP)
                  │
                  ▼
          RDS + Redis (isolated subnet, no route to internet)
```

- EC2 instances have **no public IP**; outbound traffic via NAT Gateway
- RDS security group: allow TCP 5432 from EC2 security group only
- Redis security group: allow TCP 6379 from EC2 security group only
- ALB security group: allow HTTPS 443 from 0.0.0.0/0, HTTP 80 redirect only

### Secrets Management

```python
# Never hardcode credentials. Use Secrets Manager.
import boto3, json

def get_db_credentials():
    client = boto3.client('secretsmanager', region_name='us-east-1')
    secret = client.get_secret_value(SecretId='investment-orders/db-credentials')
    return json.loads(secret['SecretString'])

# Rotate secrets on a 30-day schedule via Secrets Manager rotation Lambda
```

### WAF Rules (on ALB)

- AWS Managed Rules: CommonRuleSet + KnownBadInputsRuleSet
- Rate limiting: 1000 requests per IP per 5 minutes
- Geo-blocking: configure per regulatory requirements
- SQL injection protection: enabled

---

## Runbooks

### Runbook: RDS Failover Recovery

```bash
# 1. Confirm failover completed
aws rds describe-db-instances \
  --db-instance-identifier investment-orders-primary \
  --query 'DBInstances[0].{Status:DBInstanceStatus,AZ:AvailabilityZone}'

# 2. Verify RDS Proxy is connected to new primary
aws rds describe-db-proxy-targets --db-proxy-name investment-orders-proxy

# 3. Check application error rate in CloudWatch (should recover automatically)
# If error rate persists after 3 minutes, restart EC2 instances to force new connections
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name investment-orders-asg \
  --preferences '{"MinHealthyPercentage": 80, "InstanceWarmup": 60}'

# 4. Run smoke tests
curl -f https://api.yourdomain.com/health/deep || echo "DEEP HEALTH CHECK FAILED"

# 5. Post incident update to #incidents within 10 minutes
```

### Runbook: Redis Failure

```bash
# 1. Application should auto-degrade to DB reads via circuit breaker
# Verify logs show: redis_unavailable_fallback

# 2. Check ElastiCache status
aws elasticache describe-replication-groups \
  --replication-group-id investment-orders-redis

# 3. If cluster is in a bad state, force failover
aws elasticache test-failover \
  --replication-group-id investment-orders-redis \
  --node-group-id 0001

# 4. Monitor: once Redis recovers, circuit breaker will close automatically after 15s
# Watch for idempotency cache warm-up (first 30s will have cache misses — normal)
```

### Runbook: Rollback a Bad Deploy

```bash
# Immediate: shift 100% traffic back to blue target group
aws elbv2 modify-listener \
  --listener-arn $LISTENER_ARN \
  --default-actions Type=forward,TargetGroupArn=$BLUE_TG_ARN

# Terminate green instances
aws autoscaling delete-auto-scaling-group \
  --auto-scaling-group-name investment-orders-green-asg \
  --force-delete

# If DB migration was applied and needs reverting:
# 1. Restore from pre-deploy snapshot (see Backup section)
# 2. This is a 5-10 minute operation with data loss since snapshot
# 3. ONLY do this if the migration caused data corruption — not for code bugs
```

---

## Infrastructure as Code

All resources above are defined in Terraform under `infrastructure/terraform/`. See:

- `infrastructure/terraform/compute/` — EC2, ASG, ALB
- `infrastructure/terraform/database/` — RDS, RDS Proxy, Read Replicas
- `infrastructure/terraform/cache/` — ElastiCache
- `infrastructure/terraform/network/` — VPC, subnets, security groups
- `infrastructure/terraform/observability/` — CloudWatch dashboards, alarms
- `infrastructure/terraform/security/` — WAF, KMS, IAM roles

Apply workflow: `terraform plan` → peer review → `terraform apply` (never apply without review on production).
