# Chapter 6: Messaging & Async Processing

## 6.1 Why Async Processing?

```
┌─────────────────────────────────────────────────────────────────┐
│  SYNCHRONOUS: Request blocks until ALL work completes           │
│                                                                 │
│  User ──► POST /order ──► Validate ──► Charge ──► Reserve       │
│         │              │           │          │                 │
│         │              │           │          └─ 200ms          │
│         │              │           └─ 300ms (payment gateway)   │
│         │              └─ 50ms                                  │
│         └─ TOTAL: 550ms + network                               │
│                                                                 │
│  ASYNCHRONOUS: Request returns immediately, work continues      │
│                                                                 │
│  User ──► POST /order ──► Validate ──► Queue ──► 202 Accepted   │
│         │              │           │                            │
│         │              │           └─ Background workers:       │
│         │              │                • Charge payment        │
│         │              │                • Reserve inventory     │
│         │              │                • Send confirmation     │
│         └─ TOTAL: 50ms (immediate response)                     │
└─────────────────────────────────────────────────────────────────┘
```

**Benefits**:
- **Decoupling** — producer doesn't know consumers
- **Resilience** — queue buffers traffic spikes
- **Scalability** — scale consumers independently
- **Reliability** — retries, dead letter queues, replay

---

## 6.2 Message Queue vs Pub/Sub vs Event Streaming

| Aspect | Message Queue | Pub/Sub | Event Streaming (Kafka) |
|--------|---------------|---------|-------------------------|
| **Model** | Point-to-point | Broadcast | Log (append-only) |
| **Consumers** | 1 per message | All subscribers | Consumer groups |
| **Ordering** | Per queue | Per topic (best effort) | Per partition |
| **Replay** | No (acked = gone) | No | **Yes** (retention) |
| **Latency** | ~1–5 ms | ~1–5 ms | ~2–10 ms |
| **Throughput** | 10K–100K/s | 100K–1M/s | 1M+/s |
| **Use Case** | Tasks, commands | Notifications, fan-out | Audit, CDC, analytics |
| **Examples** | RabbitMQ, SQS | Redis PubSub, SNS | Kafka, Pulsar, Redpanda |

---

## 6.3 Message Queue Deep Dive (RabbitMQ / SQS)

### Core Concepts

```
┌─────────────────────────────────────────────────────────────────┐
│  RABBITMQ ARCHITECTURE                                          │
└─────────────────────────────────────────────────────────────────┘

Producer ──► [Exchange] ──(binding)──► [Queue] ──► Consumer
                │                        │
                │                        ├── Consumer 1
                │                        ├── Consumer 2  (competing)
                │                        └── Consumer 3
                │
                ├── Direct (exact routing key)
                ├── Topic (wildcard routing)
                ├── Fanout (broadcast)
                └── Headers (header matching)
```

### Delivery Guarantees

| Guarantee | Description | How to Achieve |
|-----------|-------------|----------------|
| **At-most-once** | Fire and forget | Auto-ack, no persistence |
| **At-least-once** | May duplicate, never lost | Manual ack, persistent messages |
| **Exactly-once** | Processed once | Idempotent consumer + transactions |

> **Exactly-once is hard**. Most systems use **at-least-once + idempotency**.

### Dead Letter Queue (DLQ)

```
Main Queue ──(reject/nack)──► DLX (Dead Letter Exchange) ──► DLQ
         │
         ├── Max retries exceeded
         ├── TTL expired
         └── Queue length limit
```

---

## 6.4 Event Streaming (Kafka)

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  KAFKA: DISTRIBUTED COMMIT LOG                                  │
└─────────────────────────────────────────────────────────────────┘

Topic: "orders"
       │
       ├── Partition 0 ──► [offset 0] [offset 1] [offset 2] ...
       ├── Partition 1 ──► [offset 0] [offset 1] [offset 2] ...
       ├── Partition 2 ──► [offset 0] [offset 1] [offset 2] ...
       └── Partition N ...

Producer ──► partitioner(key) ──► Partition
                              │
Consumer Group "inventory" ──► Partition 0, 2
Consumer Group "analytics" ──► Partition 0, 1, 2
Consumer Group "notifications" ──► Partition 1
```

### Key Properties

| Property | Detail |
|----------|--------|
| **Retention** | Time (7 days) or size (1 TB) — configurable |
| **Ordering** | Per-partition (key → partition) |
| **Replication** | RF=3 typical (ISR = In-Sync Replicas) |
| **Consumer Groups** | Each partition → 1 consumer in group |
| **Offsets** | Committed per partition, per group |
| **Log Compaction** | Keep latest value per key (changelog) |

### When to Use Kafka

```
✅ Event sourcing / audit log
✅ CDC (Change Data Capture) — Debezium
✅ High-throughput event processing
✅ Multiple consumer groups with different logic
✅ Replay / reprocessing needed
✅ Stream processing (ksqlDB, Flink)

❌ Simple task queue (use RabbitMQ/SQS)
❌ Low latency RPC replacement (use gRPC)
❌ Small team, low throughput (operational overhead)
```

---

## 6.5 Message Patterns

### Command vs Event

| Type | Intent | Naming | Example |
|------|--------|--------|---------|
| **Command** | Request action | Imperative: `CreateOrder` | `ChargePayment`, `ReserveInventory` |
| **Event** | Fact happened | Past tense: `OrderCreated` | `PaymentCharged`, `InventoryReserved` |

### Request-Reply over Messaging

```
Client ──► [Request Queue] ──► Worker
              │                     │
              │◄── [Reply Queue] ───┘
              │     (correlation_id)
              ▼
         Response
```

---

## 6.6 Exactly-Once Semantics

```
┌─────────────────────────────────────────────────────────────────┐
│  EXACTLY-ONCE = IDEMPOTENT CONSUMER + TRANSACTIONAL OUTBOX      │
└─────────────────────────────────────────────────────────────────┘

APPROACH 1: Idempotent Consumer
  • Message carries unique ID (idempotency key)
  • Consumer tracks processed IDs (Redis SET with TTL)
  • On duplicate: return cached result, don't reprocess

APPROACH 2: Transactional Outbox
  ┌─────────────────────────────────────────────────────────────┐
  │  DB Transaction:                                            │
  │    INSERT INTO orders ...                                   │
  │    INSERT INTO outbox (event_type, payload) ...             │
  │  COMMIT                                                     │
  │                                                             │
  │  Separate Relay Process:                                    │
  │    SELECT * FROM outbox WHERE published = false             │
  │    PUBLISH to Kafka                                         │
  │    UPDATE outbox SET published = true                       │
  └─────────────────────────────────────────────────────────────┘
```

---

## 6.7 Backpressure

```
Producer ──► [Queue: 100/1000] ──► Consumer (slow)
                    │
                    ▼
            Queue fills up
                    │
                    ▼
         ┌─────────────────────┐
         │ BACKPRESSURE SIGNAL │
         ├─────────────────────┤
         │ • TCP window (slow) │
         │ • HTTP 429 / 503    │
         │ • Credit-based flow │
         │ • Reactive Streams  │
         └─────────────────────┘
```

| Mechanism | Where It Works |
|-----------|----------------|
| **TCP Flow Control** | Network level (slow) |
| **HTTP 429/503** | API Gateway / Load Balancer |
| **Reactive Streams** | Application level (Project Reactor, RxJava) |
| **Kafka Consumer `max.poll.records`** | Consumer level |
| **RabbitMQ `prefetch_count`** | Per-consumer limit |

---

## 6.8 Pattern: Event Carried State Transfer

```
┌─────────────────────────────────────────────────────────────────┐
│  EVENT CARRIED STATE TRANSFER (Martin Fowler)                   │
└─────────────────────────────────────────────────────────────────┘

Service A (Source)                    Service B (Consumer)
    │                                      │
    ├─ CustomerCreated {id, name, email}──►│
    │         {id, name, email, tier}      │
    │                                      ▼
    │                              Local Copy:
    │                              Customer {id, name, email, tier}
    │
    ├─ CustomerUpdated {id, tier: Gold}──►│
    │                                      ▼
    │                              Update local copy
```

- Consumer maintains **own copy** of data it needs
- No synchronous calls to Service A
- Event contains **full state** (not just delta)
- **Trade-off**: Data duplication vs autonomy

---

## 6.9 Schema Evolution

```
┌─────────────────────────────────────────────────────────────────┐
│  MESSAGE SCHEMA EVOLUTION (Avro / Protobuf)                     │
└─────────────────────────────────────────────────────────────────┘

RULES FOR BACKWARD COMPATIBILITY (Consumer reads old + new):
  ✅ Add optional field
  ✅ Remove field (consumer ignores)
  ✅ Add enum value (consumer handles unknown)

RULES FOR FORWARD COMPATIBILITY (New consumer reads old):
  ✅ Remove optional field
  ✅ Add field with default
  ❌ Change field type
  ❌ Remove required field

FULL COMPATIBILITY (both directions):
  ✅ Add optional field with default
  ✅ Remove optional field

TOOLING:
  • Schema Registry (Confluent, Apicurio) — enforces compatibility
  • CI check: `mvn schema-registry:validate`
```

---

## ✅ Exercises

1. **Queue vs Stream**: Order processing — commands (charge, reserve) vs events (created, shipped). Which broker for each?
2. **Idempotency**: Design idempotent consumer for `ChargePayment` with idempotency key.
3. **Backpressure**: Your consumers lag behind. Kafka consumer lag = 1M messages. What do you do?
4. **Schema**: `OrderCreated` event adds `discount_code` (optional). Backward compatible? Forward compatible?

---

## 🔁 Review Prompts

- When do you use a message queue vs event streaming?
- What's the difference between at-least-once and exactly-once? How do you achieve exactly-once?
- Explain consumer groups in Kafka. How do they enable parallel processing?
- What is the transactional outbox pattern? Why is it needed?

---

## 📚 Further Reading

- *Enterprise Integration Patterns* — Hohpe & Woolf
- *Kafka: The Definitive Guide* — Neha Narkhede et al.
- [RabbitMQ Tutorials](https://www.rabbitmq.com/tutorials.html)
- *Designing Event-Driven Systems* — Ben Stopford