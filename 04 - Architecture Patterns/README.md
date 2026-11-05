# Chapter 4: Architecture Patterns

## 4.1 Monolith vs Microservices

### Monolith

```
┌─────────────────────────────────────────────────────────────┐
│                     MONOLITH                                │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐            │
│  │  Auth   │ │ Catalog │ │ Orders  │ │ Payments│  ...       │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘            │
│         │          │          │          │                  │
│         └──────────┼──────────┼──────────┘                  │
│                    ▼                                        │
│            ┌─────────────┐                                  │
│            │  DATABASE   │                                  │
│            │  (Shared)   │                                  │
│            └─────────────┘                                  │
└─────────────────────────────────────────────────────────────┘
```

| Pros | Cons |
|------|------|
| Simple to develop, test, deploy | Single codebase = coupling |
| Easy transactions (ACID) | Scale = scale everything |
| Simple debugging, profiling | Tech lock-in |
| Low operational overhead | Deploy risk = whole app |

### Microservices

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Auth      │  │  Catalog    │  │  Orders     │  │ Payments    │
│  Service    │  │  Service    │  │  Service    │  │  Service    │
│  (DB)       │  │  (DB)       │  │  (DB)       │  │  (DB)       │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │                │
       └────────────────┼────────────────┼────────────────┘
                        ▼
              ┌─────────────────┐
              │  API Gateway    │
              └─────────────────┘
```

| Pros | Cons |
|------|------|
| Independent deploy & scale | Distributed system complexity |
| Team autonomy | Network latency, failures |
| Tech heterogeneity | Data consistency (no distributed ACID) |
| Fault isolation | Operational overhead (observability, deploy) |

### When to Split

```
┌─────────────────────────────────────────────────────────────────┐
│  START WITH MONOLITH. Split when:                               │
│                                                                 │
│  ✅ Team > 10 engineers (communication overhead)                │
│  ✅ Different scaling needs (catalog read-heavy, orders write)  │
│  ✅ Different deploy cadences (payments stable, UI fast)        │
│  ✅ Clear domain boundaries (DDD bounded contexts)              │
│  ✅ Need tech heterogeneity (ML in Python, core in Go)          │
│                                                                 │
│  ❌ "Microservices are modern"                                  │
│  ❌ Premature optimization                                      │
│  ❌ No clear domain boundaries                                  │
│  ❌ Team too small to own multiple services                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4.2 Communication Patterns

### Synchronous (Request-Response)

```
Client ──► API Gateway ──► Service A ──► Service B
                    │              │           │
                    │              │           └─ gRPC/REST
                    │              └─ gRPC/REST
                    └─ HTTP/REST
```

- **Use for**: Queries, commands needing immediate response
- **Risk**: Cascade failures, latency accumulation

### Asynchronous (Event-Driven)

```
Service A ──► [Event] ──► Message Broker ──► Service B
                          │                  │
                          └──────────────────┴──► Service C
                                                     └──► Service D
```

| Pattern | Broker | Delivery | Ordering | Use Case |
|---------|--------|----------|----------|----------|
| **Message Queue** | RabbitMQ, SQS | At-least-once, 1 consumer | Per queue | Commands, tasks |
| **Pub/Sub** | Kafka, Redis PubSub | At-least-once, all subs | Per partition | Events, notifications |
| **Event Streaming** | Kafka, Pulsar | At-least-once, replayable | Per partition | Audit, CDC, analytics |

### Event-Driven Architecture (EDA)

```
┌─────────────────────────────────────────────────────────────────┐
│  EVENT-DRIVEN ARCHITECTURE                                      │
│                                                                 │
│  Producer          Event Bus / Broker          Consumers        │
│  ────────          ──────────────────          ─────────        │
│                                                                 │
│  Order Service ──► │ OrderCreated │ ──► Inventory (reserve)     │
│                    │              │ ──► Payment (charge)        │
│                    │              │ ──► Notification (email)    │
│                    │              │ ──► Analytics (track)       │
│                    │              │                             │
│                    │              ▼                             │
│                    │         Dead Letter Queue                  │
│                    │         (failed after retries)             │
└─────────────────────────────────────────────────────────────────┘
```

**Key Principles**:
- Events are facts (past tense): `OrderCreated`, `PaymentCharged`
- Consumers are independent — no coupling to producer
- Idempotency required (at-least-once delivery)
- Event schema evolution (backward/forward compatibility)

---

## 4.3 CQRS (Command Query Responsibility Segregation)

```
┌─────────────────────────────────────────────────────────────────┐
│  CQRS: SEPARATE READ AND WRITE MODELS                           │
└─────────────────────────────────────────────────────────────────┘

          COMMAND SIDE (Write)              QUERY SIDE (Read)
          ─────────────────────              ──────────────────
                    │                              │
                    ▼                              ▼
         ┌─────────────────┐            ┌─────────────────┐
         │  Write Model    │            │  Read Model(s)  │
         │  (Normalized,   │            │  (Denormalized, │
         │   Domain Logic) │            │   Optimized)    │
         └────────┬────────┘            └────────┬────────┘
                  │                              │
                  │ Event                        │ Projection
                  ▼                              ▼
         ┌─────────────────┐            ┌─────────────────┐
         │  Event Store    │            │  Read DBs       │
         │  (Source of     │            │  (PostgreSQL,   │
         │   Truth)        │            │   Elasticsearch,│
         └─────────────────┘            │   Redis, etc.)  │
                                        └─────────────────┘
```

| Benefit | Trade-off |
|---------|-----------|
| Optimize read/write independently | Eventual consistency (read model lags) |
| Different schemas for read/write | Complexity (two models, projections) |
| Scales reads without affecting writes | Event schema evolution |
| Audit trail (event store) | Debugging across models |

---

## 4.4 Event Sourcing

```
┌─────────────────────────────────────────────────────────────────┐
│  EVENT SOURCING: STATE = FOLD(EVENTS)                           │
└─────────────────────────────────────────────────────────────────┘

Traditional:     UPDATE account SET balance = 100 WHERE id = 1
Event Sourcing:  EventStore.append([
                    AccountCreated { id: 1, balance: 0 },
                    Deposited      { id: 1, amount: 50 },
                    Deposited      { id: 1, amount: 50 }
                  ])
                  // Current state = replay all events
```

| Benefit | Trade-off |
|---------|-----------|
| Complete audit trail | Replay for current state (slow for large aggregates) |
| Temporal queries ("state at time T") | Event schema migration |
| Easy debugging (reproduce from events) | Snapshotting needed for performance |
| Natural fit for CQRS | Learning curve |

---

## 4.5 Saga Pattern (Distributed Transactions)

```
┌─────────────────────────────────────────────────────────────────┐
│  SAGA: CHAIN OF LOCAL TRANSACTIONS WITH COMPENSATIONS           │
└─────────────────────────────────────────────────────────────────┘

Order Service          Payment Service      Inventory Service
    │                      │                      │
    ├─ Create Order ──────►│                      │
    │◄─ OrderCreated ──────┤                      │
    │                      ├─ Charge Payment ────►│
    │                      │◄─ PaymentCharged ────┤
    │                      │                      ├─ Reserve Inventory ──►
    │                      │                      │◄─ InventoryReserved ───┤
    │                      │                      │
    │         (FAILURE: Inventory unavailable)    │
    │                      │                      │
    ├─ Cancel Order ──────►│                      │
    │                      ├─ Refund Payment ────►│
    │                      │                      ├─ Release Inventory ────►
```

### Orchestration vs Choreography

| Aspect | Orchestration | Choreography |
|--------|---------------|--------------|
| **Coordinator** | Central (Saga Orchestrator) | None (event-driven) |
| **Visibility** | Clear flow, easy debugging | Implicit, hard to trace |
| **Coupling** | Services know orchestrator | Services only know events |
| **Failure Handling** | Centralized | Distributed |
| **Best For** | Complex workflows, strict ordering | Simple, decoupled flows |

---

## 4.6 API Gateway Patterns

### Backend for Frontend (BFF)

```
Mobile App ──► Mobile BFF ──► Services
Web App    ──► Web BFF    ──► Services
Third Party ──► Partner BFF ──► Services
```

- Tailored API per client type
- Aggregates multiple service calls
- Handles client-specific concerns (pagination, field selection)

### API Gateway Responsibilities

```
Request ──► [Gateway] ──► Services
              │
              ├── Auth (JWT, OAuth, API Key)
              ├── Rate Limiting (token bucket)
              ├── Routing (path, header, weight)
              ├── Transformation (protocol, format)
              ├── Aggregation (fan-out + compose)
              ├── Circuit Breaker
              ├── Observability (metrics, trace, log)
              └── Caching (response cache)
```

---

## 4.7 Service Mesh (East-West Traffic)

```
┌─────────────────────────────────────────────────────────────────┐
│  SERVICE MESH: SIDECAR PROXY PER SERVICE                        │
└─────────────────────────────────────────────────────────────────┘

Service A          Service B             Service C
  │                    │                     │
  ▼                    ▼                     ▼
┌───────┐            ┌───────┐            ┌───────┐
│Envoy  │            │Envoy  │            │Envoy  │
│Sidecar│            │Sidecar│            │Sidecar│
└───────┘            └───────┘            └───────┘
  │                  │                        │
  └──────────────────┼────────────────────────┘
                     ▼
            ┌────────────────┐
            │ Control Plane  │
            │ (Istio/Linkerd)│
            └────────────────┘
```

**Capabilities**:
- mTLS (automatic cert rotation)
- Traffic splitting (canary, A/B)
- Retries, timeouts, circuit breakers
- Distributed tracing injection
- Authorization policies

---

## 4.8 Pattern Selection Guide

```
┌─────────────────────────────────────────────────────────────────┐
│  NEED TO...                          USE THIS PATTERN           │
├─────────────────────────────────────────────────────────────────┤
│  Scale reads independently          CQRS + Read Replicas        │
│  Audit every change                 Event Sourcing              │
│  Coordinate multi-service tx        Saga (Orchestration)        │
│  Decouple services                  Event-Driven (Pub/Sub)      │
│  Handle traffic spikes              Async Queue + Workers       │
│  Route by client type               BFF (Backend for Frontend)  │
│  Secure service-to-service          Service Mesh (mTLS)         │
│  Gradual rollout                    Canary + Feature Flags      │
└─────────────────────────────────────────────────────────────────┘
```

---

## ✅ Exercises

1. **Architecture**: Design a ride-sharing system. Monolith or microservices? Justify.
2. **CQRS**: For an e-commerce product catalog (read-heavy, complex filters), design the read model.
3. **Saga**: Implement "transfer money between banks" as a saga with compensations.
4. **Event Sourcing**: Model a bank account with event sourcing. Show events for deposit, withdrawal, transfer.

---

## 🔁 Review Prompts

- What's the difference between a message queue and pub/sub? When to use each?
- When does CQRS add value vs complexity?
- What makes a good event in event-driven architecture?
- Orchestration vs choreography for sagas — how to decide?

---

## 📚 Further Reading

- *Enterprise Integration Patterns* — Hohpe & Woolf
- *Building Microservices* — Sam Newman
- *Designing Event-Driven Systems* — Ben Stopford
- *Microservices Patterns* — Chris Richardson