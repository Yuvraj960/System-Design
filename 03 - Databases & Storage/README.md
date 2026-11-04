# Chapter 3: Databases & Storage

## 3.1 SQL vs NoSQL vs NewSQL

### Quick Decision Guide

```
┌─────────────────────────────────────────────────────────────────────────┐
│  START: What is your PRIMARY query pattern?                             │
└─────────────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        COMPLEX JOINS    HIGH WRITE      NEED BOTH
        RELATIONAL       THROUGHPUT      ACID + SCALE
        AD-HOC QUERIES   FLEXIBLE SCHEMA  (HTAP)
              │               │               │
              ▼               ▼               ▼
          █ SQL █        █ NoSQL █       █ NewSQL █
          (PostgreSQL,   (Cassandra,     (CockroachDB,
           MySQL)        DynamoDB,       Spanner,
                        MongoDB)        TiDB, Yugabyte)
```

### Comparison Table

| Aspect | SQL (RDBMS) | NoSQL | NewSQL |
|--------|-------------|-------|--------|
| **Data Model** | Tables, rows, columns | Document, KV, Column, Graph | Tables (distributed) |
| **Schema** | Fixed, migrations | Flexible / schema-on-read | Fixed, online DDL |
| **Transactions** | Full ACID | Limited (per-doc/key) | Distributed ACID |
| **Joins** | Efficient | Application-side / denormalize | Distributed joins |
| **Horizontal Scale** | Difficult (sharding manual) | Native (partitioning built-in) | Native (transparent) |
| **Consistency** | Strong | Eventual / tunable | Strong (linearizable) |
| **Best For** | Complex relations, OLTP | High scale, simple access | Scale + strong consistency |
| **Examples** | PostgreSQL, MySQL | Cassandra, DynamoDB, MongoDB | CockroachDB, Spanner, TiDB |

---

## 3.2 Storage Engines

### B-Tree (PostgreSQL, MySQL InnoDB)

```
                    [Root: 50, 100]
                           │
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
        [10, 25]       [50, 75]       [100, 150]
          │              │              │
          ▼              ▼              ▼
       Leaf Pages    Leaf Pages     Leaf Pages
```

- **Read**: O(log n) — 3-4 I/Os typical
- **Write**: Read-modify-write, page splits on insert
- **Best for**: Read-heavy, random access, range queries

### LSM Tree (Cassandra, RocksDB, DynamoDB)

```
MemTable (RAM) ──flush──► SSTable 1 (SSD) ──compaction──► SSTable 2 ──► ...
   Sorted                                         Sorted (merged)
   Fast writes                                     Tombstones removed
```

- **Write**: O(1) append to MemTable → extremely fast
- **Read**: Check MemTable → SSTables (Bloom filter) → may need multiple files
- **Compaction**: Background merge, removes duplicates/tombstones
- **Best for**: Write-heavy, time-series, high ingestion

### Columnar (ClickHouse, Parquet, BigQuery)

```
Row-oriented:     [User1, A, X] [User2, B, Y] [User3, C, Z]
                   │      │      │      │      │      │
Column-oriented:  [User1, User2, User3]  [A, B, C]  [X, Y, Z]
                   ▲              ▲          ▲          ▲
              Compresses      Compresses  Compresses  Compresses
              well            well        well        well
```

- **Read**: Only read needed columns → 10-100× faster for analytics
- **Write**: Slow (append-only, batch)
- **Best for**: OLAP, analytics, aggregations

---

## 3.3 Replication

### Single-Leader (Primary-Replica)

```
┌─────────────┐     Async/Sync Replication      ┌─────────────┐
│   PRIMARY   │ ──────────────────────────────► │  REPLICA 1  │
│  (Writes)   │                                 │  (Reads)    │
└─────────────┘                                 └─────────────┘
       │                                                │
       │                                                ▼
       │                                         ┌─────────────┐
       └───────────────────────────────────────► │  REPLICA 2  │
                                                 │  (Reads)    │
                                                 └─────────────┘
```

| Replication Mode | Durability | Latency | Use Case |
|------------------|------------|---------|----------|
| **Synchronous** | No data loss | Higher (waits for replica) | Financial, inventory |
| **Asynchronous** | Possible loss | Lower | Analytics, feeds |
| **Semi-sync** | 1 replica ack | Balanced | General production |

### Multi-Leader

```
Region US:    PRIMARY ──► Replica
                 │
                 ▼ (async cross-region)
Region EU:    PRIMARY ──► Replica
```

- **Pros**: Write locally in each region, low latency
- **Cons**: Conflict resolution (last-write-wins, CRDTs, app-level)
- **Use**: Multi-region active-active, offline-first apps

### Leaderless (Dynamo-style)

```
Client ──► Write to W nodes (W = write quorum)
         ──► Read from R nodes (R = read quorum)
         ──► W + R > N = strong consistency
```

- **Quorum**: `W + R > N` for strong consistency
- **Hinted Handoff**: Temporarily store for down nodes
- **Read Repair**: Fix stale data on read
- **Anti-Entropy**: Background merkle tree sync

---

## 3.4 Partitioning (Sharding)

### Sharding Strategies

| Strategy | How It Works | Pros | Cons |
|----------|--------------|------|------|
| **Hash-Based** | `hash(key) % N` | Even distribution | Resharding moves all data |
| **Range-Based** | Key ranges per shard | Efficient range scans | Hot spots (recent data) |
| **Directory-Based** | Lookup table maps key→shard | Flexible, no hot spots | Directory is SPOF, extra hop |
| **Geo-Based** | Shard by user region | Data locality, compliance | Cross-region queries complex |

### Consistent Hashing (Solves Resharding)

```
                    Hash Ring (2^32)
                        
        S1 ●──────────────────────────● S2
         │                            │
         │                            │
    Keys │                            │ Keys
  map to │                            │ map to
         │                            │
        S4 ●──────────────────────────● S3
                        
Add S5: Only keys between S4 and S1 move (~1/N)
```

- **Virtual Nodes (vnodes)**: Each physical node gets multiple ring positions → better distribution
- **Used by**: DynamoDB, Cassandra, Redis Cluster, Riak

### Shard Key Selection — Critical Decision

```
┌─────────────────────────────────────────────────────────────────┐
│  WRONG SHARD KEY = HOT SHARD = SYSTEM FAILURE                   │
│                                                                 │
│  ❌ user_id for Twitter (celebrity = hot shard)                 │
│  ❌ timestamp for logs (recent = hot shard)                     │
│  ❌ tenant_id for SaaS (large tenant = hot shard)               │
│                                                                 │
│  ✅ user_id + consistent hashing + hot-key caching              │
│  ✅ compound key: (tenant_id, user_id) for SaaS                 │
│  ✅ separate hot entities to dedicated shards                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.5 Indexes

### Index Types

| Type | Structure | Best For | Write Cost |
|------|-----------|----------|------------|
| **B-Tree** | Balanced tree | Equality, range, ORDER BY | Moderate |
| **Hash** | Hash table | Exact equality only | Low |
| **GIN/GiST** | Inverted index | Full-text, JSONB, arrays | High |
| **BRIN** | Block ranges | Time-series, append-only | Very low |
| **Partial** | WHERE clause | Sparse filters (e.g., `WHERE active=true`) | Low |

### Composite Indexes — Column Order Matters

```
Index on (user_id, created_at DESC)
  ✅ WHERE user_id = ? ORDER BY created_at DESC
  ✅ WHERE user_id = ? AND created_at > ?
  ❌ WHERE created_at > ?  (user_id not in predicate)
```

**Rule**: Equality columns first, then range/sort columns.

---

## 3.6 Transactions & Isolation Levels

### ACID

| Property | Meaning |
|----------|---------|
| **Atomicity** | All or nothing |
| **Consistency** | DB constraints preserved |
| **Isolation** | Concurrent transactions don't interfere |
| **Durability** | Committed = survived crash |

### Isolation Levels (Weakest → Strongest)

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Serialization Anomaly |
|-------|------------|---------------------|--------------|----------------------|
| **Read Uncommitted** | ✅ | ✅ | ✅ | ✅ |
| **Read Committed** | ❌ | ✅ | ✅ | ✅ |
| **Repeatable Read** | ❌ | ❌ | ✅ | ✅ |
| **Serializable** | ❌ | ❌ | ❌ | ❌ |

> **PostgreSQL default**: Read Committed  
> **MySQL default**: Repeatable Read  
> **Serializable** = performance cost (locks or retries)

---

## 3.7 Distributed Transactions

### Two-Phase Commit (2PC)

```
Coordinator
    │
    ├─ Phase 1: PREPARE
    │   ├─ Node A: "Can you commit?" ──► "Yes" (locks resources)
    │   ├─ Node B: "Can you commit?" ──► "Yes" (locks resources)
    │   └─ Node C: "Can you commit?" ──► "No"  (abort)
    │
    └─ Phase 2: COMMIT/ABORT
        ├─ All "Yes" → COMMIT to all
        └─ Any "No"  → ABORT to all
```

- **Blocking**: Participants hold locks until coordinator decides
- **Coordinator failure** = blocked participants (need timeout/recovery)
- **Rarely used** in modern systems — prefer Saga

### Saga Pattern

```
Order Service          Payment Service          Inventory Service
    │                       │                       │
    ├─ Create Order ──────►│                       │
    │◄─ OrderCreated ──────┤                       │
    │                       ├─ Charge Payment ────►│
    │                       │◄─ PaymentCharged ────┤
    │                       │                       ├─ Reserve Inventory ──►
    │                       │                       │◄─ InventoryReserved ───┤
    │                       │                       │
    │                       │         (FAILURE)     │
    │                       │                       │
    ├─ Cancel Order ──────►│                       │
    │                       ├─ Refund Payment ────►│
    │                       │                       ├─ Release Inventory ────►
```

| Approach | Coordination | Pros | Cons |
|----------|--------------|------|------|
| **Choreography** | Events | Decoupled | Hard to track, cyclic deps |
| **Orchestration** | Central coordinator | Clear flow, compensation | Coordinator = SPOF |

> **Compensating transactions must be idempotent** — retries are inevitable.

---

## 3.8 Database Selection Checklist

```
□ Data model: Relational? Document? Graph? Time-series? KV?
□ Query patterns: Ad-hoc? Known access paths? Analytics?
□ Consistency needs: Strong? Eventual? Tunable?
□ Write throughput: <1K? 10K? 100K? 1M+ QPS?
□ Read throughput: Same as writes? 10×? 100×?
□ Data size: <100GB? 1TB? 100TB? PB?
□ Team expertise: SQL? NoSQL? Specific DB?
□ Operational maturity: Self-hosted? Managed? Cloud-native?
□ Cost sensitivity: Optimize for $ or performance?
□ Compliance: Data residency? Encryption? Audit?
```

---

## ✅ Exercises

1. **DB Choice**: 10K QPS writes, 100K QPS reads, 50TB, strong consistency, time-range queries. SQL, NoSQL, or NewSQL?
2. **Sharding**: Design sharding for a multi-tenant SaaS with 10K tenants, varying sizes (10 to 10M users). What shard key?
3. **Index**: Table `orders(user_id, status, created_at, total)`. Queries: by user+status, by date range, by user+date. Design indexes.
4. **Saga**: Implement a distributed transaction for "book flight + hotel + car" with compensations.

---

## 🔁 Review Prompts

- When do you choose LSM over B-Tree? Give a concrete workload.
- What's the difference between synchronous and asynchronous replication? When does each make sense?
- Explain consistent hashing. Why does it reduce data movement during resharding?
- What are the isolation levels? Which anomalies does each prevent?

---

## 📚 Further Reading

- *Database Internals* — Alex Petrov (storage engines deep dive)
- *Designing Data-Intensive Applications*, Ch 3 (Storage), Ch 5 (Replication), Ch 6 (Partitioning) — Kleppmann
- [Use The Index, Luke](https://use-the-index-luke.com/) — Markus Winand
- [Cassandra Data Modeling](https://cassandra.apache.org/doc/latest/data_modeling/) — Apache