# Investment Order Service — Deployment Architecture

> **Classification:** Internal Engineering
> **Version:** 1.0
> **Owner:** Platform Engineering
> **Last Updated:** May 2026

---

## Table of Contents

1. [Overview](#1-overview)
2. [SLO Definitions](#2-slo-definitions)
3. [Compute Architecture](#3-compute-architecture)
4. [Database Architecture](#4-database-architecture)
5. [Caching Layer](#5-caching-layer)
6. [Failover Strategy](#6-failover-strategy)
7. [Backup & Recovery](#7-backup--recovery)
8. [Deployment Pipeline](#8-deployment-pipeline)
9. [Observability](#9-observability)
10. [Security Hardening](#10-security-hardening)
11. [Runbooks](#11-runbooks)

---

## 1. Overview

The Investment Order Service is the critical path for all buy/sell/cancel operations.
Every user-facing transaction flows through this service — downtime equals direct
financial loss and regulatory exposure.

### Architecture Principles

- **No single point of failure** at any layer
- **Degrade gracefully** — read operations survive write failures
- **Immutable audit trail** — every order state transition persisted before ACK
- **Defence in depth** — security controls at network, application, and data layers

### High-Level Topology

```
                   ┌──────────────────────────────────┐
                   │         Route 53 (DNS)           │
                   │   Health-check failover routing  │
                   └─────────────┬────────────────────┘
                                 │
                   ┌─────────────▼────────────────────┐
                   │    Application Load Balancer      │
                   │  (Multi-AZ, WAF attached, TLS)   │
                   └──────┬──────────────┬────────────┘
                          │              │
            ┌─────────────▼──┐    ┌──────▼─────────────┐
            │  EC2 ASG       │    │  EC2 ASG            │
            │  us-east-1a    │    │  us-east-1b         │
            │  t3.xlarge ×2  │    │  t3.xlarge ×2       │
            └──────┬─────────┘    └──────┬──────────────┘
                   │                     │
            ┌──────▼─────────────────────▼──────────────┐
            │               Private Subnet              │
            │                                           │
            │  ┌──────────────────┐  ┌───────────────┐  │
            │  │  RDS PostgreSQL  │  │ ElastiCache   │  │
            │  │  Multi-AZ        │  │ Redis Cluster │  │
            │  │  Primary +       │  │ 3-node with   │  │
            │  │  Standby         │  │ replication   │  │
            │  └────────┬─────────┘  └───────────────┘  │
            │           │                               │
            │  ┌────────▼─────────┐                    │
            │  │  Read Replicas   │                    │
            │  │  × 2             │                    │
            │  └──────────────────┘                    │
            └──────────────────────────────────────────┘
```

---

## 2. SLO Definitions

### Why These Numbers?

Each target was derived from three inputs: (1) user tolerance for financial workflow
latency, (2) regulatory guidance on trade execution, (3) cost of infrastructure
required to achieve it.

### SLO Table

| Signal | Target | Rationale |
|--------|--------|-----------|
| **Availability** | 99.95% / month | ≤ 21.9 min downtime/month. 99.99% costs ~4× more for marginal real-world gain at this scale. |
| **Order Submit P99** | < 500ms | >500ms feels "broken" for a confirmatory financial action. Orders confirmed before WebSocket push to client. |
| **Order Submit P50** | < 100ms | Median should feel instant. Breaching this signals DB or cache issues before P99 spikes. |
| **Read Latency P99** | < 200ms | Reads from Redis replica. Tight target catches cache invalidation bugs early. |
| **Error Rate** | < 0.1% | 1 in 1000 requests failing is visible to active traders. Above 0.1% triggers a page. |
| **Durability** | 99.999999% | Lost order = regulatory incident + financial liability. Achieved via synchronous RDS write + WAL to S3 before ACK. |
| **RTO** | < 5 minutes | RDS Multi-AZ failover ~90–120s + ASG replace ~90s = 5 min combined target. |
| **RPO** | < 30 seconds | RDS synchronous standby = 0 committed data loss. 30s covers in-flight transactions during a crash. |

### Error Budget Policy

Monthly error budget at 99.95% = **21.9 minutes**.

| Threshold | Action |
|-----------|--------|
| > 50% consumed in single incident | Mandatory post-mortem |
| > 80% consumed in a month | Freeze non-critical deploys for remainder of month |
| 100% exhausted | No deploys until next month; leadership review |

---

## 3. Compute Architecture

### EC2 Configuration

**Instance:** `t3.xlarge` (4 vCPU, 16 GB RAM)

Order processing is CPU-bound (signature verification, validation) not memory-bound.
t3.xlarge provides burst capacity without compute-optimised pricing. Upgrade to
`c6i.xlarge` if sustained CPU steal exceeds 5%.

### Auto Scaling Group

```hcl
resource "aws_autoscaling_group" "investment_orders" {
  name                      = "investment-orders-asg"
  min_size                  = 4        # 2 per AZ — never below this
  max_size                  = 20
  desired_capacity          = 4
  health_check_grace_period = 120
  health_check_type         = "ELB"   # ALB health checks, not EC2
}

# Scale out fast — add 2 at a time, 60s cooldown
resource "aws_autoscaling_policy" "scale_out" {
  scaling_adjustment = 2
  adjustment_type    = "ChangeInCapacity"
  cooldown           = 60
}

# Scale in slow — remove 1 at a time, 5 min cooldown
resource "aws_autoscaling_policy" "scale_in" {
  scaling_adjustment = -1
  adjustment_type    = "ChangeInCapacity"
  cooldown           = 300
}
```

### ALB Health Checks

```hcl
health_check {
  path                = "/health/live"
  interval            = 10
  timeout             = 5
  healthy_threshold   = 2
  unhealthy_threshold = 3      # 30s before instance is pulled
  matcher             = "200"
}
```

### Health Endpoint Contract

```
GET /health/live   → 200 if process alive (used by ALB)
GET /health/ready  → 200 if DB + Redis healthy; 503 with {"failing":["rds"]} if not
GET /health/deep   → internal only; performs test DB write+read (used pre-deploy)
```

### Blue/Green Deployment

Never rolling-deploy a financial service — use blue/green to prevent in-flight
transaction corruption.

```
1. Provision green ASG alongside blue
2. Run smoke + integration tests against green
3. Shift 10% traffic via ALB weighted target groups
4. Monitor error rate + p99 for 5 minutes
5. If healthy  → shift 100% to green
6. Keep blue warm for 15 minutes, then terminate
7. If unhealthy → shift 100% back to blue instantly
```

---

## 4. Database Architecture

### RDS PostgreSQL — Multi-AZ

```hcl
resource "aws_db_instance" "primary" {
  engine                          = "postgres"
  engine_version                  = "15.4"
  instance_class                  = "db.r6g.xlarge"   # memory-optimised
  allocated_storage               = 500
  max_allocated_storage           = 2000
  storage_type                    = "io1"
  iops                            = 3000               # consistent latency
  multi_az                        = true               # synchronous standby
  deletion_protection             = true
  backup_retention_period         = 35
  backup_window                   = "02:00-03:00"
  storage_encrypted               = true
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]
}
```

### Critical PostgreSQL Parameters

```sql
synchronous_commit        = on      -- NEVER off for financial data
fsync                     = on      -- NEVER disable
wal_level                 = replica
statement_timeout         = 5000    -- 5s max per query
lock_timeout              = 2000    -- 2s, prevents lock convoys
log_min_duration_statement = 100   -- log queries > 100ms
```

### RDS Proxy (Connection Pooling)

Connection pool exhaustion is the #1 cause of P99 spikes in Django services.
RDS Proxy maintains a warm pool of authenticated connections between EC2 and RDS.

```hcl
resource "aws_db_proxy" "investment_orders" {
  engine_family          = "POSTGRESQL"
  require_tls            = true
  idle_client_timeout    = 1800

  auth {
    auth_scheme = "SECRETS"
    iam_auth    = "REQUIRED"
    secret_arn  = aws_secretsmanager_secret.db_credentials.arn
  }

  target_group {
    connection_pool_config {
      connection_borrow_timeout    = 120
      max_connections_percent      = 80
      max_idle_connections_percent = 50
    }
  }
}
```

### Read Replicas

```hcl
resource "aws_db_instance" "replica" {
  count               = 2
  replicate_source_db = aws_db_instance.primary.id
  instance_class      = "db.r6g.large"
  publicly_accessible = false
}
```

Route reads to replicas in Django:

```python
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

## 5. Caching Layer

### ElastiCache Redis — Cluster Mode

```hcl
resource "aws_elasticache_replication_group" "orders" {
  node_type                  = "cache.r6g.large"
  num_cache_clusters         = 3        # 1 primary + 2 replicas
  automatic_failover_enabled = true
  multi_az_enabled           = true
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  snapshot_retention_limit   = 7
}
```

### Cache Key Strategy

| Key Pattern | TTL | Purpose |
|-------------|-----|---------|
| `order:status:{id}` | 300s | Order status for read endpoints |
| `ratelimit:order:{user_id}` | 60s | Per-user order rate limiting |
| `idempotency:{key}` | 86400s | Duplicate order prevention |
| `session:{token}` | 3600s | Auth session tokens |
| `portfolio:{user_id}:positions` | 30s | Position cache |

### Idempotency Pattern

```python
def submit_order(request):
    idem_key = request.headers.get('Idempotency-Key')
    if not idem_key:
        return Response({"error": "Idempotency-Key header required"}, status=400)

    cached = redis.get(f"idempotency:{idem_key}")
    if cached:
        return Response(json.loads(cached), status=200)

    order = process_order(request.data)
    redis.setex(f"idempotency:{idem_key}", 86400, json.dumps(order.to_response()))
    return Response(order.to_response(), status=201)
```

---

## 6. Failover Strategy

### Scenario Matrix

| Failure | Detection | Auto-Recovery | Manual Action |
|---------|-----------|---------------|---------------|
| Single EC2 fails | 30s | Yes — ASG replaces | None |
| Full AZ outage | 30–60s | Yes — traffic shifts | Monitor ASG replacement |
| RDS primary fails | 60–120s | Yes — Multi-AZ promotes standby | Verify via RDS Proxy |
| Redis primary fails | 30–60s | Yes — replica promoted | Monitor replication lag |
| Full region failure | — | No | Manual DR activation |

### Application Retry Config

```python
DATABASES = {
    'default': {
        'HOST': os.environ['RDS_PROXY_ENDPOINT'],  # always proxy, never direct
        'CONN_MAX_AGE': 60,
        'OPTIONS': {
            'connect_timeout': 5,
            'options': '-c statement_timeout=5000 -c lock_timeout=2000',
        },
    }
}

@retry(
    retry=retry_if_exception_type(OperationalError),
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=0.5, max=5),
)
def write_order(order_data):
    with transaction.atomic():
        order = Order.objects.create(**order_data)
        OrderAuditLog.objects.create(order=order, event='CREATED')
    return order
```

### Circuit Breaker

```python
db_breaker = pybreaker.CircuitBreaker(
    fail_max=5,       # open after 5 consecutive failures
    reset_timeout=30, # retry after 30s
)

redis_breaker = pybreaker.CircuitBreaker(
    fail_max=3,
    reset_timeout=15,
)
```

---

## 7. Backup & Recovery

### RDS Backup Strategy

| Type | Frequency | Retention | Location |
|------|-----------|-----------|----------|
| Automated snapshots | Daily 02:00 UTC | 35 days | AWS managed S3 |
| WAL archiving | Continuous | 7 days | s3://orders-wal-archive |
| Pre-deploy snapshot | Every production deploy | 90 days | s3://orders-snapshots |
| Cross-region copy | Daily | 35 days | us-west-2 (DR) |

### Point-in-Time Recovery

```bash
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier investment-orders-primary \
  --target-db-instance-identifier investment-orders-recovered \
  --restore-time "2026-05-26T22:55:00Z"
```

### Redis Resilience

Redis data is **reconstructible from PostgreSQL** — Redis loss is a latency event,
not a data loss event. On unavailability, degrade gracefully:

```python
def get_order_status(order_id):
    try:
        cached = redis_breaker.call(fetch_from_cache, order_id)
        if cached:
            return json.loads(cached)
    except (pybreaker.CircuitBreakerError, redis.RedisError):
        logger.warning("redis_unavailable_fallback", extra={"order_id": order_id})
    return Order.objects.get(id=order_id).to_dict()
```

---

## 8. Deployment Pipeline

```
Push to main
     │
     ▼
GitHub Actions CI
  ├── Unit tests (pytest, 85% coverage minimum)
  ├── Integration tests (test DB + Redis)
  ├── Lint (ruff) + Type check (mypy)
  ├── Security scan (bandit + safety)
  └── Docker build + push to ECR
     │
     ▼
Staging Deploy (automatic)
  ├── Pre-deploy RDS snapshot
  ├── Blue/green deploy
  ├── Smoke tests
  └── Load test (k6): 500 req/s for 5 min
     │
     ▼ Manual approval gate
     │
Production Deploy
  ├── Pre-deploy RDS snapshot (wait for completion)
  ├── Canary: 10% traffic for 5 min
  ├── Auto-rollback if error rate > 0.5% or p99 > 1s
  └── Notify #deployments
```

---

## 9. Observability

### Key Metrics

```python
ORDER_SUBMIT_DURATION = Histogram(
    'order_submit_duration_seconds',
    buckets=[0.05, 0.1, 0.2, 0.5, 1.0, 2.0, 5.0]
)
ORDER_COUNT = Counter('orders_total', ['type', 'status'])
DB_POOL_WAIT = Histogram('db_pool_wait_seconds')
```

### Alerting Policy

| Alert | Threshold | Severity | Action |
|-------|-----------|----------|--------|
| Error rate elevated | > 0.1% for 5 min | P2 | Slack |
| Error rate critical | > 1% for 2 min | P1 | Page on-call |
| P99 elevated | > 500ms for 5 min | P2 | Slack |
| P99 critical | > 2s for 2 min | P1 | Page on-call |
| RDS failover | Any | P1 | Page on-call |
| Error budget > 50% | Monthly | P2 | Slack + email |
| Disk > 80% | Any | P2 | Slack |

### Structured Logging

```python
logger = structlog.get_logger()

def submit_order(request):
    log = logger.bind(user_id=request.user.id, trace_id=request.headers.get('X-Trace-ID'))
    t0 = time.perf_counter()
    try:
        order = process_order(request.data)
        log.info("order_submitted", order_id=str(order.id),
                 duration_ms=round((time.perf_counter() - t0) * 1000, 2))
    except Exception as e:
        log.error("order_failed", error=str(e),
                  duration_ms=round((time.perf_counter() - t0) * 1000, 2))
        raise
```

---

## 10. Security Hardening

### Network Isolation

```
Internet → WAF → ALB (public subnet)
                  ↓
            EC2 ASG (private subnet, no public IP)
                  ↓
        RDS + Redis (isolated subnet, no internet route)
```

Security group rules:
- ALB: allow 443 inbound from 0.0.0.0/0 only
- EC2: allow inbound from ALB security group only
- RDS: allow 5432 from EC2 security group only
- Redis: allow 6379 from EC2 security group only

### Secrets Management

```python
# Never hardcode credentials
import boto3, json

def get_db_credentials():
    client = boto3.client('secretsmanager')
    secret = client.get_secret_value(SecretId='investment-orders/db-credentials')
    return json.loads(secret['SecretString'])
```

Secrets rotated on a 30-day schedule via Secrets Manager rotation Lambda.

### WAF Rules

- AWS Managed: CommonRuleSet + KnownBadInputsRuleSet
- Rate limit: 1000 req / IP / 5 min
- SQL injection protection enabled
- Geo-blocking configurable per regulatory requirements

---

## 11. Runbooks

### RDS Failover Recovery

```bash
# 1. Confirm failover completed
aws rds describe-db-instances \
  --db-instance-identifier investment-orders-primary \
  --query 'DBInstances[0].{Status:DBInstanceStatus,AZ:AvailabilityZone}'

# 2. Verify RDS Proxy reconnected
aws rds describe-db-proxy-targets --db-proxy-name investment-orders-proxy

# 3. If error rate persists after 3 min, force instance refresh
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name investment-orders-asg \
  --preferences '{"MinHealthyPercentage":80,"InstanceWarmup":60}'

# 4. Run deep health check
curl -f https://api.yourdomain.com/health/deep
```

### Redis Failure

```bash
# App should auto-degrade via circuit breaker — check logs for:
# redis_unavailable_fallback

# Check cluster status
aws elasticache describe-replication-groups \
  --replication-group-id investment-orders-redis

# Force failover if needed
aws elasticache test-failover \
  --replication-group-id investment-orders-redis \
  --node-group-id 0001
```

### Rollback a Bad Deploy

```bash
# Immediately shift 100% traffic back to blue
aws elbv2 modify-listener \
  --listener-arn $LISTENER_ARN \
  --default-actions Type=forward,TargetGroupArn=$BLUE_TG_ARN

# Terminate green
aws autoscaling delete-auto-scaling-group \
  --auto-scaling-group-name investment-orders-green-asg \
  --force-delete

# DB rollback (ONLY if migration caused data corruption — not for code bugs)
# Restore from pre-deploy snapshot: 5–10 min operation
```
