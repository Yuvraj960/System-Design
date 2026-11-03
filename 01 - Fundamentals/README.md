# Chapter 1: Foundations — Scale, Estimation & Design Template

## 1.1 Scale & Performance Fundamentals

### Latency vs Throughput

| Metric | What It Measures | When It Matters |
|--------|------------------|-----------------|
| **Latency** | Time per request (p50, p95, p99) | User-facing APIs, interactive apps |
| **Throughput** | Requests per second (QPS) | Batch jobs, data ingestion, analytics |
| **Capacity** | Max concurrent requests | Capacity planning, autoscaling |

```
┌─────────────────────────────────────────────────────────────────┐
│  LATENCY vs THROUGHPUT TRADE-OFF                                │
│                                                                 │
│  Low Latency (batch=1)     ──────────────────►  High Throughput │
│  • Immediate processing           (large batches)               │
│  • Poor GPU/CPU utilization       • Max hardware utilization    │
│  • High cost per request          • Low cost per request        │
│  • Chat, gaming                   • Batch analytics, training   │
│                                                                 │
│  SWEET SPOT: Continuous batching (vLLM) — both!                 │
└─────────────────────────────────────────────────────────────────┘
```

### Percentiles Matter

```
p50 = 50ms  → "Typical user experience"
p95 = 200ms → "Most users" (SLO target often here)
p99 = 800ms → "Tail latency" — affects 1 in 100 requests
p99.9 = 2s  → "Pain threshold" — users notice, may retry
```

> **Interview tip**: Always specify which percentile. "Latency < 200ms" is meaningless without p50/p95/p99.

### Amdahl's Law

```
Speedup = 1 / [(1 - P) + P/N]
  P = parallelizable fraction
  N = number of processors
```

If 90% of your workload is parallelizable (P=0.9):
- 2 cores → 1.8× speedup
- 10 cores → 5.3× speedup
- 100 cores → 9.2× speedup (diminishing returns!)

**The serial portion (1-P) dominates at scale.** This is why you must minimize coordination, locks, and sequential bottlenecks.

---

## 1.2 Back-of-Envelope Estimation

### The Estimation Framework

```
STEP 1: Clarify scale assumptions
  • DAU / MAU
  • Actions per user per day
  • Read:Write ratio
  • Data size per object
  • Geographic distribution

STEP 2: Calculate core metrics
  • QPS (average & peak)
  • Storage (growth rate)
  • Bandwidth (ingress/egress)
  • Memory (cache, KV cache for LLMs)

STEP 3: Identify bottlenecks
  • Which resource hits limit first?
  • CPU? Disk I/O? Network? Memory? GPU?
```

### Powers of Two — Memorize These

| 2^n | Value | Use Case |
|-----|-------|----------|
| 2^10 | 1,024 ≈ 1K | KB |
| 2^20 | 1,048,576 ≈ 1M | MB |
| 2^30 | 1,073,741,824 ≈ 1B | GB |
| 2^40 | ≈ 1T | TB |
| 2^10 × 2^10 = 2^20 | 1M | Quick multiplication |

### Quick Formulas

```
Average QPS = (DAU × actions_per_user_per_day) / 86,400
Peak QPS    = Average QPS × peak_multiplier (typically 2–5×)
Daily Storage = DAU × objects_per_user × size_per_object
Monthly Storage = Daily Storage × 30 × replication_factor
Bandwidth (Mbps) = QPS × avg_response_bytes × 8 / 1,000,000
```

### 📐 Estimation Drill: Twitter-like Feed

```
Assumptions:
  • 300M DAU
  • 500 tweets/user/day (mostly reads)
  • 1 KB per tweet (text + metadata)
  • Read:Write = 100:1
  • Peak multiplier = 3×
  • 3-year retention

Calculations:
  1. Write QPS = 300M × 500 / 86,400 ≈ 1,736 QPS
  2. Read QPS = 1,736 × 100 = 173,600 QPS
  3. Peak Write QPS = 1,736 × 3 ≈ 5,200 QPS
  4. Peak Read QPS = 173,600 × 3 ≈ 520,000 QPS
  5. Daily Storage = 300M × 500 × 1KB = 150 GB/day
  6. 3-Year Storage = 150 GB × 365 × 3 × 3 (replication) ≈ 490 TB
  5. Read Bandwidth = 520K QPS × 1KB × 8 ≈ 4.2 Gbps

Binding Constraints:
  • Write: Single DB leader can handle ~5K QPS → needs sharding
  • Read: 520K QPS → MUST use aggressive caching + read replicas
  • Storage: 490 TB → needs distributed storage (S3 + columnar for analytics)
```

---

## 1.3 The Design Template (RESHADED)

### For Interviews: RESHADED Framework

| Step | Time | What You Do | Interviewer Expects |
|------|------|-------------|---------------------|
| **R**equirements | 5 min | Clarify functional + non-functional | Structured questions, scope narrowing |
| **E**stimation | 5 min | Back-of-envelope math | Numbers on whiteboard, bottleneck ID |
| **S**torage / Schema | 5 min | Data model, DB choice | Justified SQL vs NoSQL, partitioning |
| **H**igh-Level Design | 10 min | Component diagram, data flow | Clean diagram, labeled components |
| **A**PI Design | 5 min | Endpoints, payloads, contracts | REST/gRPC, pagination, versioning |
| **D**eep Dives | 10 min | 2-3 critical components | Trade-offs, failure modes, scaling |
| **E**valuation | 5 min | Trade-offs, SPOFs, costs | Proactive weakness identification |
| **D**iscussion | 5 min | Follow-ups, alternatives | Handles pushback, communicates clearly |

### For Production: Architecture Decision Record (ADR)

```
# ADR-0042: Choose Cassandra over PostgreSQL for Event Store

## Context
Ingestion pipeline receives 500K events/sec from IoT devices.
Events are immutable, append-only, time-series.
Query pattern: 95% time-range scans, 5% device-lookup.

## Decision
Use Apache Cassandra (wide-column) with:
  - Partition key: device_id
  - Clustering key: timestamp DESC
  - TTL: 90 days
  - RF=3, CL=QUORUM for writes, CL=ONE for reads

## Trade-offs
+ Linear write scaling (no leader bottleneck)
+ Native time-series compaction
+ Tunable consistency per query
- No JOINs (denormalize device metadata)
- Read-before-write for device lookup (use cache)
- Operational complexity vs managed PostgreSQL

## Consequences
- Estimated 10× write throughput vs PostgreSQL
- ~$15K/month on 6-node cluster vs $50K for equivalent PG
- Team needs Cassandra expertise (training budget approved)

## Status: Accepted (2024-01-15)
```

---

## 1.4 Consistency Models

### CAP Theorem (Refined)

```
┌─────────────────────────────────────────────────────────────────┐
│  During a NETWORK PARTITION (P), you MUST choose:               │
│                                                                 │
│  CP  →  Consistency + Partition Tolerance                       │
│        Reject writes if quorum unavailable                      │
│        Example: etcd, ZooKeeper, CP databases                   │
│                                                                 │
│  AP  →  Availability + Partition Tolerance                      │
│        Accept writes on any node, reconcile later               │
│        Example: Cassandra, DynamoDB, Riak                       │
│                                                                 │
│  CA  →  Impossible in distributed systems                       │
│        (Network partitions WILL happen)                         │
└─────────────────────────────────────────────────────────────────┘
```

### PACELC Theorem (More Nuanced)

```
If Partitioned (P):  Choose Availability (A) or Consistency (C)
Else (E):            Choose Latency (L) or Consistency (C)

┌──────────┬──────────┬────────────────────────────────────────┐
│ System   │ PA/PC    │ EL/EC                                  │
├──────────┼──────────┼────────────────────────────────────────┤
│ etcd     │ PC       │ EC  (Strong consistency always)        │
│ ZooKeeper│ PC       │ EC                                     │
│ Cassandra│ PA       │ EC  (Tunable: QUORUM = EC, ONE = EL)   │
│ DynamoDB │ PA       │ EC  (Strong read = EC, eventual = EL)  │
│ Kafka    │ PC       │ EC  (ISR = in-sync replicas)           │
└──────────┴──────────┴────────────────────────────────────────┘
```

### Consistency Spectrum (Weakest → Strongest)

| Model | Guarantee | Use Case |
|-------|-----------|----------|
| **Eventual** | Replicas converge if no new writes | DNS, social feeds |
| **Read-your-writes** | User sees own updates | Session stickiness |
| **Monotonic reads** | Never go back in time | Route to same replica |
| **Consistent prefix** | Causal order preserved | Kafka partition ordering |
| **Linearizable** | Appears instantaneous global order | etcd, Spanner, locks |
| **Serializable** | Transactions as if serial | Banking, inventory |

---

## 1.5 Availability & Reliability Math

### Nines of Availability

| Availability | Downtime/Year | Downtime/Month | Typical Use Case |
|--------------|---------------|----------------|------------------|
| 99% (two nines) | 3.65 days | 7.2 hours | Dev/test, batch jobs |
| 99.9% (three nines) | 8.76 hours | 43 minutes | Standard production |
| 99.99% (four nines) | 52 minutes | 4 minutes | Payments, core APIs |
| 99.999% (five nines) | 5 minutes | 26 seconds | Life-critical, trading |

### SLI / SLO / SLA

| Term | Definition | Example |
|------|------------|---------|
| **SLI** (Service Level Indicator) | Measurable metric | Request latency p99 |
| **SLO** (Service Level Objective) | Target for SLI | p99 latency < 200ms |
| **SLA** (Service Level Agreement) | Contract with penalty | 99.9% uptime or credit |

```
Error Budget = 1 - SLO
  • SLO 99.9% → Error budget = 0.1% = 43 min/month
  • If budget exhausted → freeze features, invest in reliability
  • Burn rate alerts predict SLO violation early
```

---

## ✅ Exercises

1. **Estimation**: Design a URL shortener for 100M DAU. Calculate QPS, storage for 5 years, bandwidth.
2. **CAP Choice**: You're building a multiplayer game leaderboard. CP or AP? Why?
3. **PACELC**: Your e-commerce inventory system needs strong consistency for stock deduction but low latency for product browsing. What PACELC category? How do you achieve both?
4. **Error Budget**: Your SLO is 99.95% availability. You've had 30 minutes of downtime this month (30 days). How much error budget remains?

---

## 🔁 Review Prompts

- What's the difference between p95 and p99 latency? Which do you optimize for?
- Explain Amdahl's Law in your own words. Why does it matter for system design?
- When would you choose CP over AP? Give a concrete example.
- What does "error budget" mean operationally?

---

## 📚 Further Reading

- [Latency Numbers Every Programmer Should Know](https://gist.github.com/jboner/2841832)
- *Designing Data-Intensive Applications*, Ch 9 (Consistency) — Kleppmann
- [SRE Workbook: SLOs](https://sre.google/workbook/table-of-contents/) — Google
- [PACELC Theorem Explained](https://www.vldb.org/pvldb/vol3/p1441-abadi.pdf) — Abadi