# Chapter 13: Cheatsheets & References

> **Quick reference for interviews, design reviews, and daily work**

---

## 13.1 Latency Numbers Every Engineer Should Know

```
┌─────────────────────────────────────────────────────────────────┐
│                    LATENCY HIERARCHY                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  L1 Cache Reference           0.5 ns        ────────────────    │
│  Branch Mispredict            5 ns          ────────────────    │
│  L2 Cache Reference           7 ns          ────────────────    │
│  Mutex Lock/Unlock           25 ns          ────────────────    │
│  Main Memory Reference      100 ns          ────────────────    │
│  Compress 1KB (Zippy)       3,000 ns       ────────────────     │
│  Send 1KB over 1 Gbps       10,000 ns      ────────────────     │
│  Read 1MB sequentially      250,000 ns     ────────────────     │
│  Round Trip in Datacenter   500,000 ns     ────────────────     │
│  Read 1MB from SSD          1,000,000 ns   ────────────────     │
│  Read 1MB from Network     10,000,000 ns   ────────────────     │
│  Disk Seek (HDD)           10,000,000 ns   ────────────────     │
│  Send 1MB over 1 Gbps LAN  10,000,000 ns   ────────────────     │
│  Cross-Region RTT          50,000,000 ns   ────────────────     │
│  Read 1MB from S3          100,000,000 ns  ────────────────     │
│                                                                 │
│  RULE: Each level is ~100× slower than the previous             │
│        Cache hit at RAM saves 10ms vs SSD                       │
│        Cache hit at CDN saves 100ms vs origin                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 13.2 Throughput & Capacity Anchors

```
┌─────────────────────────────────────────────────────────────────┐
│                    THROUGHPUT ANCHORS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SINGLE NODE LIMITS (Approximate)                               │
│  ─────────────────────────────────                              │
│  PostgreSQL (writes, leader)      1,000–5,000 QPS               │
│  PostgreSQL (reads, replicas)    10,000–50,000 QPS              │
│  MySQL (writes, leader)           2,000–10,000 QPS              │
│  Redis (single-threaded)         100,000–500,000 QPS            │
│  Redis Cluster (sharded)         1M+ QPS                        │
│  Kafka (single partition)         50,000–100,000 msgs/sec       │
│  Kafka (partitioned)              1M+ msgs/sec                  │
│  Load Balancer (L7, NGINX)       100,000–1M RPS                 │
│  API Gateway (Kong/Envoy)         50,000–200,000 RPS            │
│  CDN Edge                         1M+ RPS (static)              │
│                                                                 │
│  HARDWARE LIMITS                                                │
│  ─────────────────                                              │
│  NVMe SSD (random 4K)             500K IOPS, 3 GB/s             │
│  NVMe SSD (sequential)            7 GB/s                        │
│  DDR4 RAM                         25–50 GB/s                    │
│  10 Gbps Network                  1.25 GB/s                     │ 
│  25 Gbps Network                  3 GB/s                        │
│  100 Gbps Network                 12.5 GB/s                     │
│  PCIe 4.0 x16                     32 GB/s                       │
│  NVLink (H100)                    900 GB/s                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 13.3 Estimation Formulas

```
┌─────────────────────────────────────────────────────────────────┐
│                    BACK-OF-ENVELOPE MATH                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  QPS CALCULATIONS                                               │
│  ─────────────────                                              │
│  avg_qps = (DAU × actions_per_user_per_day) / 86,400            │
│  peak_qps = avg_qps × peak_multiplier (2–5×)                    │
│                                                                 │
│  STORAGE                                                        │
│  ───────                                                        │
│  daily_storage = DAU × objects_per_user × size_per_object       │
│  total_storage = daily_storage × retention_days × replication   │
│                                                                 │
│  BANDWIDTH                                                      │
│  ──────────                                                     │
│  bandwidth_mbps = qps × avg_response_kb × 8 / 1000              │
│                                                                 │
│  MEMORY (CACHE)                                                 │
│  ────────────────                                               │
│  cache_memory = hot_keys × avg_value_size × replication         │
│                                                                 │
│  POWERS OF TWO (Memorize)                                       │
│  ─────────────────────────                                      │
│  2^10  = 1,024        ≈ 1K     (KB)                             │
│  2^20  = 1,048,576    ≈ 1M     (MB)                             │
│  2^30  = 1,073,741,824≈ 1B     (GB)                             │
│  2^40  = 1,099,511,627,776 ≈ 1T (TB)                            │
│  2^10 × 2^10 = 2^20  (1K × 1K = 1M)                             │
│  2^10 × 2^20 = 2^30  (1K × 1M = 1B)                             │
│                                                                 │
│  COMMON MULTIPLIERS                                             │
│  ───────────────────                                            │
│  Peak multiplier: 2–5× (typical 3×)                             │
│  Read:Write ratio: 10:1 to 100:1 (social), 1:1 (payment)        │
│  Replication factor: 3 (standard)                               │
│  Compression: 2–10× (text), 1.5–2× (images)                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 13.4 Database Quick Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│              SQL vs NoSQL vs NewSQL QUICK REF                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  USE SQL (PostgreSQL, MySQL) WHEN:                              │
│  ✅ Complex relationships, JOINs                                │
│  ✅ ACID transactions required                                  │
│  ✅ Ad-hoc queries, reporting                                   │
│  ✅ Mature tooling, team expertise                              │
│  ✅ Data fits on 1-2 nodes (vertical scale OK)                  │
│                                                                 │
│  USE NoSQL WHEN:                                                │
│  ✅ Massive scale (horizontal required)                         │
│  ✅ Simple access patterns (known queries)                      │
│  ✅ Flexible schema (rapid iteration)                           │
│  ✅ High write throughput                                       │
│  ✅ Specific data model fit:                                    │
│     • Key-Value: Redis, DynamoDB (simple lookups)               │
│     • Document: MongoDB (nested, variable schema)               │
│     • Wide-Column: Cassandra, Scylla (time-series, logs)        │
│     • Graph: Neo4j (relationships, traversals)                  │
│                                                                 │
│  USE NewSQL (CockroachDB, TiDB, Spanner) WHEN:                  │
│  ✅ Need horizontal scale + strong consistency                  │
│  ✅ Multi-region active-active                                  │
│  ✅ SQL interface preferred                                     │
│  ✅ Can accept higher latency vs single-node SQL                │
│                                                                 │
│  STORAGE ENGINE TRADE-OFFS                                      │
│  ───────────────────────────────                                │
│  B-Tree (PostgreSQL, MySQL): Read-heavy, range queries, OLTP    │
│  LSM (Cassandra, RocksDB): Write-heavy, time-series, compaction │
│  Columnar (ClickHouse, Parquet): Analytics, OLAP, compression   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 13.5 Consistency Models Cheatsheet

```
┌─────────────────────────────────────────────────────────────────┐
│              CONSISTENCY SPECTRUM (Weakest → Strongest)         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  EVENTUAL              │ Replicas converge if no new writes     │
│  ──────────────────────┼────────────────────────────────────    │
│  DNS, Cassandra (default), DynamoDB (eventual)                  │
│                                                                 │
│  READ-YOUR-WRITES      │ User sees own updates immediately      │
│  ──────────────────────┼────────────────────────────────────    │
│  Session stickiness, user-scoped routing                        │
│                                                                 │
│  MONOTONIC READS       │ Never see older data after newer       │
│  ──────────────────────┼────────────────────────────────────    │
│  Route to same replica                                          │
│                                                                 │
│  CONSISTENT PREFIX     │ Causal order preserved                 │
│  ──────────────────────┼────────────────────────────────────    │
│  Kafka partition ordering                                       │
│                                                                 │
│  LINEARIZABLE          │ Appears instantaneous global order     │
│  ──────────────────────┼────────────────────────────────────    │
│  etcd, ZooKeeper, Spanner, Consul                               │
│  (Required for: locks, leader election, counters)               │
│                                                                 │
│  SERIALIZABLE          │ Transactions as if serial execution    │
│  ──────────────────────┼────────────────────────────────────    │
│  PostgreSQL (SERIALIZABLE), Spanner                             │
│  (Required for: banking, inventory)                             │
│                                                                 │
│  CAP THEOREM (During Partition)                                 │
│  ──────────────────────────────                                 │
│  CP: Reject writes if quorum unavailable (etcd, ZooKeeper)      │
│  AP: Accept writes, reconcile later (Cassandra, DynamoDB)       │
│  CA: Impossible in distributed systems                          │
│                                                                 │
│  PACELC (Extended)                                              │
│  ────────────────                                               │
│  If Partitioned (P): Choose Availability (A) or Consistency (C) │
│  Else (E): Choose Latency (L) or Consistency (C)                │
│                                                                 │
│  System      │ PA/PC  │ EL/EC  │ Notes                          │
│  ────────────┼────────┼────────┼─────────────────────────────── │
│  etcd        │ PC     │ EC     │ Strong always                  │
│  ZooKeeper   │ PC     │ EC     │ Strong always                  │
│  Cassandra   │ PA     │ EC*    │ Tunable (QUORUM=EC, ONE=EL)    │
│  DynamoDB    │ PA     │ EC*    │ Strong read = EC, eventual=EL  │
│  Kafka       │ PC     │ EC     │ ISR = in-sync replicas         │
│  Spanner     │ PC     │ EC     │ TrueTime, external consistency │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 13.6 Caching Patterns Quick Reference

```
┌─────────────────────────────────────────────────────────────────┐
│                    CACHING PATTERNS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CACHE-ASIDE (Lazy Loading)          WRITE-THROUGH              │
│  ─────────────────────────           ─────────────              │
│  App checks cache                    App writes to cache        │
│  On miss → DB → populate cache       Cache writes to DB         │
│  Most common                         Strong consistency         │
│  Stale on write                      Slower writes              │
│                                                                 │
│  READ-THROUGH                         WRITE-BEHIND (Write-Back) │
│  ─────────────                         ──────────────────────   │
│  Cache loads on miss                  App writes to cache       │
│  Transparent to app                   Async flush to DB         │
│  Simpler app logic                    Fast writes               │
│  Cache = data access layer            Risk of data loss         │
│                                                                 │
│  WRITE-AROUND                         REFRESH-AHEAD             │
│  ─────────────                         ─────────────            │
│  Write to DB only                     Pre-load before expiry    │
│  Cache populated on read              Reduces miss latency      │
│  Avoids cache pollution               Complex to tune           │
│                                                                 │
│  INVALIDATION STRATEGIES                                        │
│  ─────────────────────────────                                  │
│  TTL: Simple, eventual consistency                              │
│  Active: On write, delete/update keys (strong)                  │
│  Pub/Sub: Write publishes event, caches subscribe (distributed) │
│  Versioned Keys: key:v{version}, write new version              │
│                                                                 │
│  STAMPEDE PREVENTION                                            │
│  ───────────────────────                                        │
│  1. Probabilistic early expiration                              │
│  2. Mutex/lock (single recomputation)                           │
│  3. Lease-based (Redis SETNX)                                   │
│  4. Request collapsing (single flight)                          │
│                                                                 │
│  EVICTION POLICIES                                              │
│  ───────────────────                                            │
│  LRU: General purpose, temporal locality                        │
│  LFU: Stable popularity (top items)                             │
│  TTL: Naturally stale data (feeds, config)                      │
│  Random: Memcached default at scale                             │
│  ARC: Adaptive (LRU + LFU)                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 13.7 Resilience Patterns Quick Reference

```
┌─────────────────────────────────────────────────────────────────┐
│                    RESILIENCE PATTERNS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TIMEOUT                    RETRY WITH BACKOFF                  │
│  ─────────                  ─────────────────                   │
│  Never wait forever         Exponential backoff + jitter        │
│  Hierarchy: client > gateway > service > DB                     │
│  Connection vs request timeout                                  │
│                                                                 │
│  CIRCUIT BREAKER            BULKHEAD                            │
│  ────────────────           ────────                            │
│  Closed → Open → Half-Open  Isolate failure domains             │
│  Trip: error rate > 50%     Thread pools per downstream         │
│  Fail fast in Open          Connection pools per downstream     │
│  Probe in Half-Open         Process/container per service       │
│                                                                 │
│  RATE LIMITING              IDEMPOTENCY                         │
│  ─────────────              ───────────                         │
│  Token bucket (burst OK)    Same request N times = same result  │
│  At edge (API Gateway)      Client generates key                │
│  Per user/IP/key            Server stores key → response        │
│  Return 429 + Retry-After   TTL 24h                             │
│                                                                 │
│  GRACEFUL DEGRADATION       DEAD LETTER QUEUE                   │
│  ─────────────────────      ────────────────                    │
│  Tier 1: Never degrade      Quarantine failed messages          │
│  Tier 2: Cached fallback    Retry with backoff                  │
│  Tier 3: Disable feature    Alert after max retries             │
│  Tier 4: Static fallback    Manual replay                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 13.8 Load Balancing Algorithms

```
┌─────────────────────────────────────────────────────────────────┐
│              LOAD BALANCING ALGORITHMS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ROUND ROBIN              WEIGHTED ROUND ROBIN                  │
│  ────────────             ──────────────────                    │
│  Cycle through            Weight = capacity                     │
│  Equal distribution       Server A (5): 5 requests              │
│  Homogeneous only         Server B (3): 3 requests              │
│                           Server C (2): 2 requests              │
│                                                                 │
│  LEAST CONNECTIONS        LEAST RESPONSE TIME                   │
│  ──────────────────       ──────────────────                    │
│  Fewest active conns      Lowest latency + fewest conns         │
│  Variable duration        Heterogeneous performance             │
│  Long-lived requests      Best for user-facing                  │
│                                                                 │
│  IP HASH / STICKY         CONSISTENT HASH                       │
│  ─────────────────        ───────────────                       │
│  hash(IP) % N             Hash ring + virtual nodes             │
│  Session affinity         Minimal reshuffle on add/remove       │
│  Legacy, no cookies       Cache, sharding, sticky sessions      │
│                                                                 │
│  POWER OF TWO CHOICES (Recommended for large scale)             │
│  ─────────────────────────────────────────                      │
│  Pick 2 random, route to least loaded                           │
│  Near-optimal, simple, no central state                         │
│  Used by Google, Uber, Dropbox                                  │
│                                                                 │
│  L4 vs L7                                                       │
│  ───────                                                        │
│  L4 (TCP): IP+Port, fast, no TLS term, no routing               │
│  L7 (HTTP): Full request, TLS term, routing, auth, rewrite      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 13.9 Message Queue vs Pub/Sub vs Streaming

```
┌─────────────────────────────────────────────────────────────────┐
│              MESSAGING PATTERN COMPARISON                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MESSAGE QUEUE              PUB/SUB              EVENT STREAM   │
│  ───────────────            ────────             ────────────   │
│  RabbitMQ, SQS              Redis PubSub, SNS    Kafka, Pulsar  │
│  Point-to-point             Broadcast            Log (replay)   │
│  1 consumer per msg         All subscribers      Consumer group │
│  At-least-once              At-least-once        At-least-once  │
│  Per-queue ordering         Best-effort          Per-partition  │
│  No replay                  No replay            Retention      │
│  Tasks, commands            Notifications        Audit, CDC,    │
│                             Fan-out              Analytics      │
│                                                                 │
│  DELIVERY GUARANTEES                                            │
│  ──────────────────────                                         │
│  At-most-once: Fire & forget (may lose)                         │
│  At-least-once: May duplicate (standard)                        │
│  Exactly-once: Processed once (hard, needs idempotency)         │
│                                                                 │
│  ORDERING                                                       │
│  ────────                                                       │
│  Per-queue: Total order within queue                            │
│  Per-partition: Total order within partition                    │
│  Global: Requires single partition (bottleneck)                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 13.10 API Design Quick Reference

```
┌─────────────────────────────────────────────────────────────────┐
│                    API DESIGN QUICK REF                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HTTP METHODS                                                   │
│  ─────────────                                                  │
│  GET    - Read (safe, idempotent)                               │
│  POST   - Create, actions (not idempotent)                      │
│  PUT    - Full replace (idempotent)                             │
│  PATCH  - Partial update (not idempotent*)                      │
│  DELETE - Remove (idempotent)                                   │
│                                                                 │
│  STATUS CODES                                                   │
│  ─────────────                                                  │
│  200  OK                    400  Bad Request                    │
│  201  Created               401  Unauthorized                   │
│  202  Accepted (async)      403  Forbidden                      │
│  204  No Content            404  Not Found                      │
│  304  Not Modified          409  Conflict                       │
│                             422  Unprocessable Entity           │
│                             429  Too Many Requests              │
│                             500  Internal Server Error          │
│                             503  Service Unavailable            │
│                                                                 │
│  PAGINATION                                                     │
│  ───────────                                                    │
│  Cursor-based (preferred):                                      │
│  ?limit=20&cursor=eyJpZCI6MTAwfQ==                              │
│  Offset-based (small datasets only):                            │
│  ?limit=20&offset=100                                           │
│                                                                 │
│  VERSIONING                                                     │
│  ───────────                                                    │
│  URL: /api/v1/users (major versions)                            │
│  Header: Accept: application/vnd.api.v1+json (minor)            │
│                                                                 │
│  ERROR FORMAT                                                   │
│  ─────────────                                                  │
│  { "error": { "code": "VALIDATION_ERROR", "message": "...",     │
│    "details": [{"field": "email", "code": "INVALID"}] } }       │
│                                                                 │
│  REST vs GRAPHQL vs GRPC                                        │
│  ─────────────────────────                                      │
│  REST: Public APIs, simple CRUD, caching                        │
│  GraphQL: Complex UIs, varying data needs, mobile               │
│  gRPC: Service-to-service, high throughput, streaming           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 13.11 Availability Math

```
┌─────────────────────────────────────────────────────────────────┐
│                    AVAILABILITY & SLOS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NINES OF AVAILABILITY                                          │
│  ──────────────────────────                                     │
│  99%      (two nines)    = 3.65 days/year  = 7.2 hours/month    │
│  99.9%    (three nines)  = 8.76 hours/year = 43.8 minutes/month │
│  99.99%   (four nines)   = 52.6 minutes/year = 4.38 min/month   │
│  99.999%  (five nines)   = 5.26 minutes/year = 26.3 sec/month   │
│                                                                 │
│  SLI / SLO / SLA                                                │
│  ─────────────────                                              │
│  SLI = Service Level Indicator (metric measured)                │
│  SLO = Service Level Objective (target)                         │
│  SLA = Service Level Agreement (contract with penalty)          │
│                                                                 │
│  ERROR BUDGET                                                   │
│  ─────────────                                                  │
│  Error Budget = 1 - SLO                                         │
│  SLO 99.9% → Error budget = 0.1% = 43.2 min/month               │
│  If exhausted → freeze features, invest in reliability          │
│                                                                 │
│  BURN RATE ALERTING                                             │
│  ─────────────────────                                          │
│  Burn Rate = Error Rate / (1 - SLO)                             │
│  SLO 99.9% (0.1% budget):                                       │
│    1% error rate   → 10× burn → Alert 2h window                 │
│    5% error rate   → 50× burn → Alert 15m window (page)         │
│    10% error rate  → 100× burn → Alert immediate (page)         │
│                                                                 │
│  RTO / RPO                                                      │
│  ─────────                                                      │
│  RTO = Recovery Time Objective (how fast to restore)            │
│  RPO = Recovery Point Objective (how much data loss OK)         │
│                                                                 │
│  Tier 1 (Payments):  RTO < 5 min, RPO = 0                       │
│  Tier 2 (Core API):  RTO < 30 min, RPO < 1 min                  │
│  Tier 3 (Analytics): RTO < 4 hours, RPO < 1 hour                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 13.12 AI System Design Quick Reference

```
┌─────────────────────────────────────────────────────────────────┐
│                    AI/LLM SYSTEM ANCHORS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MODEL SIZES (FP16)                                             │
│  ────────────────                                               │
│  7B params    → 14 GB                                           │
│  13B params   → 26 GB                                           │
│  34B params   → 68 GB                                           │
│  70B params   → 140 GB (needs 2× 80GB GPU)                      │
│  405B params  → 810 GB (needs 10× 80GB GPU or tensor parallel)  │
│                                                                 │
│  KV CACHE PER TOKEN (70B, GQA 8 heads)                          │
│  ─────────────────────────────────────────                      │
│  FP16: 0.33 MB/token                                            │
│  4K context: 1.3 GB/request                                     │
│  128K context: 43 GB/request  ⚠️                                │
│                                                                 │
│  SERVING THROUGHPUT (H100, vLLM)                                │
│  ─────────────────────────────────                              │
│  7B:  ~8,000 tok/s/GPU                                          │
│  13B: ~5,000 tok/s/GPU                                          │
│  70B: ~3,000 tok/s/GPU (with continuous batching)               │
│  405B: ~500 tok/s/GPU (tensor parallel 8-way)                   │
│                                                                 │
│  OPTIMIZATIONS                                                  │
│  ─────────────                                                  │
│  Continuous batching:  2–4× throughput                          │
│  PagedAttention:       Near-zero fragmentation                  │
│  INT8 quantization:    2× throughput, minimal quality loss      │
│  INT4 (AWQ/GPTQ):      3–4× throughput, 1–2% quality loss       │
│  Speculative decoding: 2–3× (predictable outputs)               │
│  Prefix caching:       30–70% prefill savings                   │
│  Semantic caching:     30–70% hit rate (similar queries)        │
│                                                                 │
│  PARALLELISM                                                    │
│  ───────────                                                    │
│  Tensor Parallel (TP): Split weights, NVLink, intra-node        │
│  Pipeline Parallel (PP): Split layers, Ethernet OK, inter-node  │
│  Expert Parallel (EP): MoE experts, low bandwidth               │
│  Data Parallel (DP): Replicate model, shard data                │
│                                                                 │
│  RAG ANCHORS                                                    │
│  ──────────                                                     │
│  Embedding dim: 384 (small) – 3072 (large)                      │
│  Chunk size: 512–1024 tokens                                    │
│  Top-K retrieval: 5–20                                          │
│  Reranker adds: 50–200ms latency                                │
│  Hybrid (dense+sparse): Best recall                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 13.13 Common Ports & Protocols

```
┌─────────────────────────────────────────────────────────────────┐
│                    COMMON PORTS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HTTP/HTTPS       80 / 443                                      │
│  DNS              53 (UDP/TCP)                                  │
│  SSH              22                                            │
│  MySQL            3306                                          │
│  PostgreSQL       5432                                          │
│  Redis            6379                                          │
│  MongoDB          27017                                         │
│  Cassandra        9042                                          │
│  Kafka            9092 (plaintext), 9093 (SSL)                  │
│  ZooKeeper        2181                                          │
│  etcd             2379 (client), 2380 (peer)                    │
│  Consul           8500                                          │
│  Prometheus       9090                                          │
│  Grafana          3000                                          │
│  Jaeger           16686 (UI), 6831 (UDP agent)                  │
│  gRPC             50051 (default)                               │
│  WebSocket        80/443 (ws://, wss://)                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 13.14 Glossary

| Term | Definition |
|------|------------|
| **HLD** | High-Level Design — components, data flow, interfaces |
| **LLD** | Low-Level Design — classes, schemas, sequences |
| **QPS** | Queries Per Second |
| **p50/p95/p99** | Percentile latencies |
| **CAP** | Consistency, Availability, Partition Tolerance |
| **PACELC** | If Partitioned → A/C; Else → Latency/Consistency |
| **RTO/RPO** | Recovery Time/Point Objective |
| **SLO/SLI/SLA** | Objective / Indicator / Agreement |
| **Fan-out** | Distributing one event to many consumers |
| **Backpressure** | Downstream signaling upstream to slow down |
| **Idempotency** | Repeating operation = same result |
| **Sharding** | Horizontal partitioning of data |
| **Consistent Hashing** | Minimal-remapping hash ring |
| **CQRS** | Command Query Responsibility Segregation |
| **RESHADED** | Requirements, Estimation, Storage, HLD, API, Deep dives, Evaluation, Discussion |
| **TTFT** | Time To First Token (LLM) |
| **ITL** | Inter-Token Latency (LLM) |
| **KV Cache** | Key-Value cache for transformer attention |
| **LoRA** | Low-Rank Adaptation (efficient fine-tuning) |
| **RAG** | Retrieval-Augmented Generation |

---

## 13.15 Interview Checklist

```
┌─────────────────────────────────────────────────────────────────┐
│              SYSTEM DESIGN INTERVIEW CHECKLIST                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PHASE 1: REQUIREMENTS (5 min)                                  │
│  ☐ Functional requirements listed                               │
│  ☐ Non-functional: QPS, latency, availability, consistency      │
│  ☐ Constraints: budget, team, timeline, compliance              │
│  ☐ Explicitly stated out-of-scope                               │
│                                                                 │
│  PHASE 2: ESTIMATION (5 min)                                    │
│  ☐ Avg QPS, Peak QPS calculated                                 │
│  ☐ Storage (daily, total, replication)                          │
│  ☐ Bandwidth                                                    │
│  ☐ Binding constraint IDENTIFIED                                │
│                                                                 │
│  PHASE 3: HIGH-LEVEL DESIGN (10 min)                            │
│  ☐ Data model + schema                                          │
│  ☐ DB choice justified (SQL/NoSQL/NewSQL)                       │
│  ☐ Component diagram drawn                                      │
│  ☐ Data flow labeled                                            │
│  ☐ APIs defined (endpoints, schemas)                            │
│                                                                 │
│  PHASE 4: DEEP DIVES (10 min)                                   │
│  ☐ Deep dive 1: ____________________ (trade-offs discussed)     │
│  ☐ Deep dive 2: ____________________ (trade-offs discussed)     │
│  ☐ Deep dive 3: ____________________ (trade-offs discussed)     │
│                                                                 │
│  PHASE 5: TRADE-OFFS & FAILURE (5 min)                          │
│  ☐ SPOFs identified                                             │
│  ☐ Failure modes per component                                  │
│  ☐ Scaling limits discussed                                     │
│  ☐ Cost at scale estimated                                      │
│  ☐ Operational concerns (deploy, monitoring, rollback)          │
│                                                                 │
│  COMMUNICATION                                                  │
│  ☐ Narrated thinking throughout                                 │
│  ☐ Checked in with interviewer                                  │
│  ☐ Handled pushback gracefully                                  │
│  ☐ Finished within 40 min, left 5 min for follow-ups            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 13.16 Additional Resources

### Books (Priority Order)
1. **Designing Data-Intensive Applications** — Martin Kleppmann (The bible)
2. **System Design Interview Vol. 1 & 2** — Alex Xu (Case studies)
3. **Site Reliability Engineering** — Google (SLOs, incident response)
4. **Database Internals** — Alex Petrov (Storage engines)
5. **Understanding Distributed Systems** — Roberto Vitillo
6. **Release It!** — Michael Nygard (Stability patterns)
7. **Designing ML Systems** — Chip Huyen (MLOps)

### Blogs & Newsletters
- [ByteByteGo](https://blog.bytebytego.com/) — Weekly visual system design
- [High Scalability](http://highscalability.com/) — Real architectures
- [Netflix Tech Blog](https://netflixtechblog.com/) — Scale, chaos, streaming
- [Uber Engineering](https://eng.uber.com/) — Dispatch, geospatial, ML
- [Discord Engineering](https://discord.com/blog/engineering) — High-scale chat
- [AWS/GCP/Azure Architecture Blogs] — Managed service patterns

### Papers (Must-Read)
- [The Tail at Scale](https://research.google/pubs/pub40801/) — Dean & Barroso
- [Dynamo: Amazon's Highly Available Key-Value Store](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
- [Spanner: Google's Globally-Distributed Database](https://research.google/pubs/pub39966)
- [Paxos Made Simple](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf)
- [Raft Consensus](https://raft.github.io/raft.pdf)
- [PagedAttention](https://arxiv.org/abs/2309.06180) — vLLM
- [LLM Serving at Scale](https://arxiv.org/abs/2303.08774) — Google

---

> **End of Repository** — Happy System Designing! 🏗️