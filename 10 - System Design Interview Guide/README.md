# Chapter 10: System Design Interview Framework

## 10.1 The 14-Step Interview Framework

```
┌─────────────────────────────────────────────────────────────────┐
│  45-MINUTE SYSTEM DESIGN INTERVIEW                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PHASE 1: UNDERSTAND & SCOPE (10 min)                           │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Step 1: Clarify Requirements (5 min)                    │    │
│  │   • Functional: What does it do?                        │    │
│  │   • Non-functional: Scale, latency, availability        │    │
│  │   • Constraints: Budget, team, timeline, compliance     │    │
│  ├─────────────────────────────────────────────────────────┤    │
│  │ Step 2: Define Functional Requirements                  │    │
│  │   • Core features (CRUD, real-time, search, etc.)       │    │
│  │   • Out of scope (explicitly state)                     │    │
│  ├─────────────────────────────────────────────────────────┤    │
│  │ Step 3: Define Non-Functional Requirements              │    │
│  │   • QPS (read/write), DAU, data size                    │    │
│  │   • Latency (p50, p95, p99), Availability (99.9%?)      │    │
│  │   • Consistency (strong? eventual? per feature?)        │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  PHASE 2: MATH & INTERFACE (10 min)                             │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Step 4: Capacity Estimation (5 min)                     │    │
│  │   • QPS, Storage, Bandwidth, Memory                     │    │
│  │   • Identify BINDING CONSTRAINT                         │    │
│  ├─────────────────────────────────────────────────────────┤    │
│  │ Step 5: Define System APIs (5 min)                      │    │
│  │   • REST/gRPC endpoints, request/response schemas       │    │
│  │   • Pagination, versioning, auth                        │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  PHASE 3: HIGH-LEVEL ARCHITECTURE (10 min)                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Step 6: Data Model & Schema (3 min)                     │    │
│  │   • Entities, relationships, access patterns            │    │
│  ├─────────────────────────────────────────────────────────┤    │
│  │ Step 7: Database Selection (2 min)                      │    │
│  │   • SQL vs NoSQL vs NewSQL — justify                    │    │
│  ├─────────────────────────────────────────────────────────┤    │
│  │ Step 8: High-Level Diagram (5 min)                      │    │
│  │   • Client → LB → Services → DB/Cache/Queue             │    │
│  │   • Label components, data flow, protocols              │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  PHASE 4: DETAILED DESIGN & DEEP DIVES (10 min)                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Step 9: Component Deep Dive (3 min)                     │    │
│  │   • Hardest logical challenge (ranking, matching, etc.) │    │
│  ├─────────────────────────────────────────────────────────┤    │
│  │ Step 10: Partitioning & Sharding (2 min)                │    │
│  │   • Shard key, strategy, resharding plan                │    │
│  ├─────────────────────────────────────────────────────────┤    │
│  │ Step 11: Caching Strategy (2 min)                       │    │
│  │   • What, where, invalidation, stampede prevention      │    │
│  ├─────────────────────────────────────────────────────────┤    │
│  │ Step 12: Load Balancing (1 min)                         │    │
│  │   • L4/L7, algorithm, health checks                     │    │
│  ├─────────────────────────────────────────────────────────┤    │
│  │ Step 13: Async Processing (2 min)                       │    │
│  │   • Queues, events, delivery guarantees, ordering       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  PHASE 5: EVALUATION (5 min)                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Step 14: Trade-offs Analysis (5 min)                    │    │
│  │   • SPOFs, failure modes, scaling limits                │    │
│  │   • Cost at scale, operational complexity               │    │
│  │   • What you'd do differently with more time            │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 10.2 Requirements Clarification — Questions to Ask

### Functional

```
□ What are the core features? (Create, Read, Update, Delete, Search, Real-time?)
□ Who are the users? (Consumers, enterprises, internal?)
□ What are the main user flows?
□ Any admin/management features?
□ Any integrations? (Third-party APIs, webhooks)
□ Data retention / deletion requirements?
□ Multi-tenancy? (Isolation level)
□ Geographic requirements? (Data residency, latency)
```

### Non-Functional

```
□ Scale: DAU / MAU? Peak QPS? Read:Write ratio?
□ Latency: p50, p95, p99 targets? (Interactive vs batch)
□ Availability: 99.9%? 99.99%? RTO/RPO?
□ Consistency: Strong per feature? Eventual OK where?
□ Durability: Data loss tolerance? (Zero? Seconds? Minutes?)
□ Security: AuthZ/AuthN? Encryption? Compliance (GDPR, HIPAA, PCI)?
□ Cost: Budget constraints? Cost per request target?
□ Team: Team size? Expertise? Operational maturity?
```

---

## 10.3 Capacity Estimation Cheatsheet

### Quick Formulas

```
# QPS
avg_qps = (DAU × actions_per_user_per_day) / 86400
peak_qps = avg_qps × peak_multiplier  # 2–5× typical

# Storage
daily_storage = DAU × objects_per_user × size_per_object
total_storage = daily_storage × retention_days × replication_factor

# Bandwidth
bandwidth_mbps = qps × avg_response_kb × 8 / 1000

# Memory (Cache)
cache_memory = hot_keys × avg_value_size × replication_factor
```

### Numbers to Memorize

| Metric | Value |
|--------|-------|
| Seconds per day | 86,400 |
| 1 KB | 1,024 bytes |
| 1 MB | 1,048,576 bytes |
| 1 GB | 1,073,741,824 bytes |
| 1 TB | 1,099,511,627,776 bytes |
| L1 cache | 0.5 ns |
| RAM | 100 ns |
| SSD read | 16 µs |
| Datacenter RTT | 0.5 ms |
| Redis RTT | 0.5–1 ms |
| Cross-region RTT | 50–150 ms |

---

## 10.4 Common System Design Problems

### Tier 1: Fundamentals (New Grad / Junior)

| Problem | Key Concepts |
|---------|--------------|
| **URL Shortener** | Hashing, Base62, ID generation, caching |
| **Key-Value Store** | Consistent hashing, replication, CAP |
| **Rate Limiter** | Token bucket, sliding window, Redis |
| **Unique ID Generator** | Snowflake, UUID, coordination |

### Tier 2: Core Patterns (Mid-Level)

| Problem | Key Concepts |
|---------|--------------|
| **Social News Feed** | Fan-out on write vs read, ranking, pagination |
| **Chat System** | WebSocket, presence, message ordering, sharding |
| **Notification System** | Pub/sub, deduplication, preferences, channels |
| **Search Autocomplete** | Trie, ranking, debouncing, caching |
| **Distributed Cache** | Consistent hashing, hot keys, replication |

### Tier 3: Distributed Systems (Senior)

| Problem | Key Concepts |
|---------|--------------|
| **Video Streaming** | Transcoding, adaptive bitrate, CDN, signed URLs |
| **Ride Sharing** | Geospatial (geohash/quadtree), matching, surge |
| **E-commerce/Orders** | Inventory, saga, idempotency, exactly-once |
| **Payment System** | Idempotency, reconciliation, ledger, audit |
| **Web Crawler** | Politeness, dedup, frontier, prioritization |

### Tier 4: Scale + Trade-offs (Staff+)

| Problem | Key Concepts |
|---------|--------------|
| **Ad-Tech/RTB** | Sub-10ms, bidding, pacing, frequency capping |
| **Metrics/Monitoring** | High cardinality, downsampling, retention |
| **Distributed Logging** | Ingestion, indexing, compression, query |
| **Global CDN** | Cache invalidation, edge compute, routing |

---

## 10.5 Anti-Patterns to Avoid

```
┌─────────────────────────────────────────────────────────────────┐
│  INTERVIEW ANTI-PATTERNS                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. SOLUTIONING BEFORE SCOPING                                  │
│     ❌ Jump straight to drawing database                        │
│     ✅ Clarify requirements first (5 min)                       │
│                                                                 │
│  2. BUZZWORD BINGO                                              │
│     ❌ "We'll use Kafka, Cassandra, Kubernetes, Redis..."       │
│     ✅ "We need async processing → Kafka because..."            │
│                                                                 │
│  3. SINGLE POINT OF FAILURE                                     │
│     ❌ Single DB, no replicas, no LB health checks              │
│     ✅ Identify every SPOF, add redundancy                      │
│                                                                 │
│  4. IGNORING BOTTLENECKS                                        │
│     ❌ "It scales" without identifying limits                   │
│     ✅ "At 100K QPS, DB write leader is bottleneck → shard"     │
│                                                                 │
│  5. SILENT TREATMENT                                            │
│     ❌ Think silently for 5 minutes                             │
│     ✅ Think OUT LOUD, narrate decisions                        │
│                                                                 │
│  6. NO TRADE-OFF DISCUSSION                                     │
│     ❌ "This is the best design"                                │
│     ✅ "Trade-off: CP for payments, AP for feed"                │
│                                                                 │
│  7. OVER-ENGINEERING                                            │
│     ❌ Multi-region active-active for MVP                       │
│     ✅ Start simple, add complexity when justified              │
│                                                                 │
│  8. FORGETTING OPERATIONS                                       │
│     ❌ No monitoring, deployments, rollback plan                │
│     ✅ Mention: health checks, metrics, alerts, canary deploy   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 10.6 Time Management Tips

```
┌─────────────────────────────────────────────────────────────────┐
│  TIME BOXING (45 min)                                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  0–5 min   : Requirements (clarify, scope)                      │
│  5–10 min  : Estimation (numbers on board)                      │
│  10–20 min : HLD + API (diagram, data flow)                     │
│  20–35 min : Deep Dives (2–3 areas, interviewer guides)         │
│  35–42 min : Trade-offs, Failure Modes, Operations              │
│  42–45 min : Follow-ups, Wrap-up                                │
│                                                                 │
│  ⚠️  If spending >25 min on HLD → you're over-engineering       │
│  ⚠️  If no deep dive by 25 min → interviewer will push          │
│  ⚠️  Always leave 5 min for trade-offs                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 10.7 Mock Interview Practice

### Practice Structure

```
Week 1-2:  Learn framework, do estimation drills
Week 3:    1 design/day (35 min design + 10 min review)
Week 4:    3 mock interviews (peer or Pramp/Interviewing.io)
Week 5:    Target company-specific problems
```

### Self-Grading Rubric

| Criteria | Weight | Strong (5) | Weak (1) |
|----------|--------|------------|----------|
| **Requirements** | 15% | Structured clarifying questions, explicit scope | Assumes, vague |
| **Estimation** | 15% | Quantitative, identifies bottleneck | Hand-wavy |
| **HLD** | 20% | Clean diagram, justified components | Boxes without logic |
| **Deep Dives** | 30% | 2–3 areas with trade-offs | Shallow everywhere |
| **Trade-offs** | 10% | Proactive SPOF, cost, failure modes | Only when asked |
| **Communication** | 10% | Narrates, checks in, handles pushback | Silent or defensive |

---

## 10.8 Company-Specific Flavors

| Company | Focus | Style |
|---------|-------|-------|
| **Google** | Scale, correctness, algorithms | Deep technical, expect math |
| **Meta** | Product sense, trade-offs, move fast | Practical, "how would you build this" |
| **Amazon** | Operational excellence, customer obsession | Leadership principles, "bias for action" |
| **Netflix** | Resilience, chaos, freedom & responsibility | Failure modes, degradation |
| **Uber** | Real-time, geospatial, matching | Scale + correctness |
| **Stripe** | Correctness, reliability, API design | Payments domain, idempotency |
| **Airbnb** | Trust, marketplace, search | Two-sided platform |
| **Startups** | Velocity, pragmatism, MVP | "Ship it", iterate |

---

## ✅ Exercises

1. **Timed Design**: Set 45-min timer. Design "Design a pastebin service". Grade yourself.
2. **Estimation Drill**: 5 min each — estimate QPS/storage for: Instagram, Uber, Spotify, Dropbox.
3. **Trade-off Practice**: For each: SQL vs NoSQL, CP vs AP, Sync vs Async, Monolith vs Micro — write 3 pros/cons.
4. **Failure Mode**: Pick your last design. List 5 failure scenarios and mitigations.

---

## 🔁 Review Prompts

- What are the 4 pillars interviewers score? (Requirements, HLD, Deep Dives, Trade-offs)
- How much time should you spend on estimation? What must you produce?
- What's the #1 anti-pattern in system design interviews?
- Name 3 questions you must ask in requirements clarification.

---

## 📚 Further Reading

- [System Design Interview Framework — DesignGurus](https://designgurus.io/system-design-interview-guide)
- *System Design Interview Vol. 1 & 2* — Alex Xu
- *Grokking the System Design Interview* — Educative
- [ByteByteGo Newsletter](https://blog.bytebytego.com/)
