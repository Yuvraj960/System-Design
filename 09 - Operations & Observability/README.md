# Chapter 9: Operations & Observability

## 9.1 The Three Pillars of Observability

```
┌─────────────────────────────────────────────────────────────────┐
│  METRICS ──► What happened? (Aggregated, numerical)             │
│  LOGS    ──► Why did it happen? (Discrete, detailed)            │
│  TRACES  ──► Where did it happen? (Request flow, latency)       │
└─────────────────────────────────────────────────────────────────┘
```

### Metrics (Prometheus / StatsD / CloudWatch)

```prometheus
# Counter: monotonically increasing
http_requests_total{method="GET",handler="/users",status="200"} 12345

# Gauge: point-in-time value
memory_usage_bytes{instance="api-1"} 536870912

# Histogram: distribution (buckets)
http_request_duration_seconds_bucket{handler="/users",le="0.1"} 5000
http_request_duration_seconds_bucket{handler="/users",le="0.5"} 9000
http_request_duration_seconds_bucket{handler="/users",le="+Inf"} 10000

# Summary: quantiles (client-side)
http_request_duration_seconds{handler="/users",quantile="0.99"} 0.234
```

### RED Metrics (Per Service)

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| **Rate** | Requests per second | Drop > 50% |
| **Errors** | Error rate (5xx / total) | > 1% |
| **Duration** | Latency p50, p95, p99 | p99 > SLO |

### USE Metrics (Per Resource)

| Metric | Description |
|--------|-------------|
| **Utilization** | % busy (CPU, disk, network) |
| **Saturation** | Queue depth, wait time |
| **Errors** | Error count (CRC, timeout, OOM) |

---

## 9.2 Distributed Tracing

### Trace Context Propagation

```
Trace: abc123 (trace-id)
  │
  ├── Span: api-gateway (span-id: 111)
  │     │
  │     ├── Span: auth-service (span-id: 222, parent: 111)
  │     │     └── Span: redis (span-id: 333, parent: 222)
  │     │
  │     └── Span: user-service (span-id: 444, parent: 111)
  │           └── Span: postgres (span-id: 555, parent: 444)
  │
  └── Span: async-worker (span-id: 666, parent: 111, follow_up: true)
        └── Span: email-service (span-id: 777, parent: 666)
```

### W3C Trace Context Headers

```
traceparent: 00-abc123-111-01
  │      │       │    │
  │      │       │    └─ flags (01 = sampled)
  │      │       └────── parent-span-id
  │      └────────────── trace-id
  └───────────────────── version

tracestate: vendor=value,vendor2=value2  (vendor-specific)
```

### Sampling Strategies

| Strategy | Rate | Use Case |
|----------|------|----------|
| **Head-based** | Fixed % (e.g., 10%) | Default, simple |
| **Tail-based** | 100% errors, 1% success | Cost control, keep errors |
| **Adaptive** | Dynamic based on throughput | High-scale |
| **Priority** | 100% for VIP users / critical paths | Business-critical |

---

## 9.3 Structured Logging

### JSON Log Format

```json
{
  "timestamp": "2024-01-15T10:30:00.123Z",
  "level": "INFO",
  "service": "user-service",
  "trace_id": "abc123",
  "span_id": "444",
  "message": "User retrieved",
  "user_id": "user_789",
  "duration_ms": 45,
  "db_query_count": 2,
  "cache_hit": true
}
```

### Log Levels

| Level | When | Retention |
|-------|------|-----------|
| **DEBUG** | Detailed diagnostics | 1–7 days |
| **INFO** | Business events, requests | 30 days |
| **WARN** | Recoverable issues, retries | 90 days |
| **ERROR** | Failed operations, exceptions | 1+ years |
| **FATAL** | Process crash, data loss | Forever |

---

## 9.4 SLI / SLO / SLA

### Definitions

| Term | Definition | Example |
|------|------------|---------|
| **SLI** (Service Level Indicator) | Measurable metric | Request latency p99 |
| **SLO** (Service Level Objective) | Target for SLI | p99 latency < 200ms |
| **SLA** (Service Level Agreement) | Contract with penalty | 99.9% uptime or 10% credit |

### Error Budget

```
SLO: 99.9% availability over 30 days
     = 43.2 minutes downtime allowed

Error Budget = 1 - SLO = 0.1% = 43.2 min

If budget exhausted:
  → Freeze feature deployments
  → Focus on reliability work
  → Page on-call for investigation
```

### Burn Rate Alerting

```
Burn Rate = (Error Rate) / (1 - SLO)

SLO 99.9% (0.1% error budget)
  1% error rate  → 10× burn rate → Alert: 2h window
  5% error rate  → 50× burn rate → Alert: 15m window (page)
  10% error rate → 100× burn rate → Alert: immediate (page)
```

---

## 9.5 Deployment Strategies

### Blue-Green Deployment

```
        BLUE (v1)          GREEN (v2)
    ┌──────────────┐    ┌─────────────┐
    │  Active      │    │  Staging    │
    │  100% traffic│    │  0% traffic │
    └──────┬───────┘    └──────┬──────┘
           │                   │
           ▼                   ▼
    ┌─────────────────────────────────┐
    │         LOAD BALANCER           │
    │    (Instant switch DNS/VIP)     │
    └─────────────────────────────────┘
```

| Pros | Cons |
|------|------|
| Instant rollback | 2× infrastructure cost |
| Zero downtime | State migration needed |
| Full validation before switch | Database migrations tricky |

### Canary Deployment

```
Traffic:  100% ──► [90% v1] [10% v2] ──► [50% v1] [50% v2] ──► [0% v1] [100% v2]
Metrics:       ▲                    ▲                    ▲
               │                    │                    │
              Compare: latency, errors, business metrics
              Automated rollback on regression
```

### Rolling Deployment

```
Kubernetes RollingUpdate:
  maxSurge: 25%        (extra pods during rollout)
  maxUnavailable: 25%  (pods down during rollout)
  
  Pod v1 ──► Pod v2 (wait for Ready) ──► Pod v2 ──► ...
```

### Feature Flags

```python
# LaunchDarkly / Unleash / homegrown
if feature_flag.enabled("new-checkout-flow", user_id):
    return new_checkout(user)
else:
    return legacy_checkout(user)
```

| Use Case | Flag Type |
|----------|-----------|
| Gradual rollout | Percentage (1% → 100%) |
| Targeted release | User segment (beta users) |
| Kill switch | Boolean (instant off) |
| Experiment | A/B variants |

---

## 9.6 Capacity Planning

### Capacity Model

```
┌─────────────────────────────────────────────────────────────────┐
│  CAPACITY = (HEADROOM) × (PEAK LOAD)                            │
└─────────────────────────────────────────────────────────────────┘

Current:  50% CPU at 10K QPS
Peak:     50K QPS (5×)
Headroom: 2× (for failures, growth)
Target:   50K QPS × 2 = 100K QPS capacity

Instances needed:
  Current: 10 instances @ 10K QPS each (50% CPU)
  Target:  20 instances (100K QPS / 5K QPS per instance at 50% CPU)
          + 20% buffer = 24 instances
```

### Autoscaling Signals

| Signal | Metric | Scale-Up Trigger | Scale-Down Trigger |
|--------|--------|------------------|-------------------|
| **CPU** | `cpu_usage` | > 70% for 5m | < 30% for 20m |
| **Request Rate** | `rps` | > 80% capacity | < 40% capacity |
| **Queue Depth** | `queue_length` | > 100 | < 10 |
| **Latency** | `p99_latency` | > SLO threshold | < 50% SLO |
| **Custom** | Business metric | Domain-specific | Domain-specific |

### Cooldown & Hysteresis

```
Scale-up:  Fast (1–2 min) — respond to load quickly
Scale-down: Slow (10–20 min) — avoid flapping

Hysteresis: Different thresholds for up vs down
  Scale-up:  CPU > 70%
  Scale-down: CPU < 30%  (not 70%!)
```

---

## 9.7 Incident Response

### Incident Severity

| Severity | Definition | Response | Example |
|----------|------------|----------|---------|
| **SEV-1** | Customer-facing outage, data loss | Page immediately, war room | Payments down, data corruption |
| **SEV-2** | Degraded functionality, partial impact | Page on-call, 30min response | High latency, feature broken |
| **SEV-3** | Minor issue, workaround exists | Ticket, business hours | Non-critical bug, cosmetic |
| **SEV-4** | Low priority, tech debt | Backlog | Missing docs, minor optimization |

### Runbook Template

```markdown
# Runbook: High Latency on /api/orders

## Symptoms
- p99 latency > 2s for > 5min
- Error rate < 1% (not erroring, just slow)

## Diagnosis Steps
1. Check RED metrics for order-service
2. Check downstream: payment-service, inventory-service
3. Check DB: slow queries, connection pool, replication lag
4. Check Redis: hit rate, memory, evictions
5. Check Kafka: consumer lag

## Common Causes & Fixes
| Cause | Detection | Fix |
|-------|-----------|-----|
| DB connection pool exhausted | `db_pool_wait > 0` | Increase pool, fix leaks |
| Slow query (missing index) | `pg_stat_statements` | Add index, analyze |
| Kafka consumer lag | `consumer_lag > 10K` | Scale consumers, check processing |
| Redis memory pressure | `used_memory > 80% maxmemory` | Increase memory, evict |

## Escalation
- 15 min: Page secondary on-call
- 30 min: Page manager, start war room
- 60 min: Executive notification
```

---

## 9.8 Chaos Engineering (Revisited)

```
┌─────────────────────────────────────────────────────────────────┐
│  CHAOS EXPERIMENT TEMPLATE                                      │
├─────────────────────────────────────────────────────────────────┤
│  Hypothesis: "If Redis primary fails, read replicas serve       │
│  traffic within 30s with < 5% error rate increase"              │
│                                                                 │
│  Steady State:                                                  │
│  - p99 latency: 45ms                                            │
│  - Error rate: 0.01%                                            │
│  - Cache hit rate: 95%                                          │
│                                                                 │
│  Experiment:                                                    │
│  1. Kill Redis primary pod                                      │
│  2. Observe failover time                                       │
│  3. Measure error rate, latency during failover                 │
│  4. Verify replica promotion                                    │
│                                                                 │
│  Rollback: Restore primary, verify replication                  │
│                                                                 │
│  Results:                                                       │
│  - Failover time: 12s                                           │
│  - Error rate spike: 2% (client retries)                        │
│  - Latency spike: p99 800ms (15s)                               │
│  - ✅ Hypothesis CONFIRMED                                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9.9 Cost Optimization

### Cost Visibility

```
┌─────────────────────────────────────────────────────────────────┐
│  UNIT ECONOMICS: Cost per meaningful unit                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Cost per request      = Total infra cost / Requests/month      │
│  Cost per active user  = Total infra cost / MAU                 │
│  Cost per transaction  = Total infra cost / Transactions        │
│  Cost per GB stored    = Storage cost / GB                      │
│                                                                 │
│  TARGET: Track trends, not absolutes.                           │
│  "Cost per request decreased 20% after caching"                 │
└─────────────────────────────────────────────────────────────────┘
```

### Optimization Levers

| Lever | Typical Savings | Effort |
|-------|-----------------|--------|
| **Right-sizing** | 20–40% | Low (vertical) |
| **Spot/Preemptible** | 60–90% (batch) | Medium |
| **Autoscaling to zero** | 30–50% (dev/staging) | Low |
| **Compression** | 30–50% bandwidth | Low |
| **Columnar storage** | 5–10× (analytics) | Medium |
| **Caching** | 60–80% origin offload | Medium |
| **Data tiering** | 50–80% (cold data) | Medium |

---

## ✅ Exercises

1. **SLI/SLO**: Define SLIs and SLOs for a payment API. What's the error budget for 99.95% availability?
2. **Tracing**: Design trace propagation for: Client → Gateway → Auth → User DB → Redis → Payment Service → Kafka.
3. **Deployment**: Compare blue-green vs canary for a stateful service with database migrations.
4. **Autoscaling**: Design autoscaling policy for a service with: variable load (10K–100K QPS), p99 SLO 200ms, cold start 30s.

---

## 🔁 Review Prompts

- What's the difference between metrics, logs, and traces? When do you use each?
- How does error budget drive operational decisions?
- What are the trade-offs between blue-green and canary deployments?
- How do you choose autoscaling metrics? Why not just CPU?

---

## 📚 Further Reading

- *Site Reliability Engineering* — Google (Ch on Monitoring, SLOs, Incident Response)
- *Observability Engineering* — Charity Majors et al.
- [Prometheus Best Practices](https://prometheus.io/docs/practices/)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [Google SRE Workbook](https://sre.google/workbook/table-of-contents/)