# Glossary

Quick reference for system design terminology used throughout this repository.

---

## A–C

| Term | Definition |
|------|------------|
| **ACID** | Atomicity, Consistency, Isolation, Durability — transaction guarantees |
| **API Gateway** | Entry point handling auth, routing, rate limiting, aggregation |
| **Async Processing** | Non-blocking work via queues/streams; decouples producer/consumer |
| **Availability** | System uptime % (e.g., 99.9% = ~8.76h downtime/year) |
| **Backpressure** | Downstream signals upstream to slow down (prevents overload) |
| **BASE** | Basically Available, Soft state, Eventual consistency (NoSQL model) |
| **Bloom Filter** | Probabilistic data structure for membership test (may false positive) |
| **Bulkhead** | Isolation pattern: separate resource pools per downstream |
| **CAP Theorem** | During Partition: choose Consistency OR Availability |
| **CDN** | Content Delivery Network — caches content at edge PoPs globally |
| **Circuit Breaker** | Fail fast when dependency fails; states: Closed → Open → Half-Open |
| **Consistent Hashing** | Hash ring minimizing remapping on node add/remove |
| **CQRS** | Command Query Responsibility Segregation — separate read/write models |
| **Cron** | Time-based job scheduler |

## D–F

| Term | Definition |
|------|------------|
| **Dead Letter Queue (DLQ)** | Queue for messages failing repeatedly; quarantine for inspection |
| **Denormalization** | Duplicate data to avoid JOINs; optimizes reads |
| **Distributed Tracing** | Follow request across services via trace_id/span_id |
| **DMA** | Direct Memory Access — hardware accesses memory without CPU |
| **DNS** | Domain Name System — resolves hostnames to IPs |
| **Docker** | Containerization platform |
| **DDoS** | Distributed Denial of Service — flood with traffic |

## E–G

| Term | Definition |
|------|------------|
| **E2E Encryption** | End-to-End Encryption — only endpoints can decrypt |
| **ELB/ALB/NLB** | AWS Load Balancers (Classic/Application/Network) |
| **Event Sourcing** | Store state changes as event log; state = replay(events) |
| **Eventual Consistency** | Replicas converge if no new writes |
| **Exactly-Once** | Processed once (requires idempotency + transactional outbox) |
| **Fan-out** | One event → many consumers (write vs read path trade-off) |
| **Feature Flag** | Toggle feature without deploy; enables gradual rollout |
| **gRPC** | High-performance RPC over HTTP/2 with Protobuf |
| **GSLB** | Global Server Load Balancing — geo-aware DNS routing |

## H–K

| Term | Definition |
|------|------------|
| **HLD** | High-Level Design — system components, data flow, interfaces |
| **Hot Key / Hot Shard** | Disproportionate traffic to one key/partition |
| **HTTP/2** | Multiplexed streams, header compression, server push |
| **HTTP/3 (QUIC)** | UDP-based, 0-RTT, no head-of-line blocking |
| **Idempotency** | f(f(x)) = f(x) — repeating request = same result |
| **Inverted Index** | Term → document mapping (search engines) |
| **ITL** | Inter-Token Latency — time between generated tokens (LLM) |
| **Kafka** | Distributed event streaming platform (log-based) |
| **Kubernetes (K8s)** | Container orchestration platform |
| **KV Cache** | Key-Value cache storing attention states in LLM decoding |

## L–N

| Term | Definition |
|------|------------|
| **L4 / L7** | Layer 4 (TCP) vs Layer 7 (HTTP) load balancing |
| **Latency** | Time per request (p50, p95, p99) |
| **LLD** | Low-Level Design — classes, schemas, sequences |
| **Load Balancer** | Distributes traffic across backends |
| **LoRA** | Low-Rank Adaptation — efficient fine-tuning (small adapter matrices) |
| **LRU / LFU** | Least Recently/Frequently Used — cache eviction policies |
| **LSA** | Latency-Sensitive Application |
| **LSM Tree** | Log-Structured Merge Tree — write-optimized storage engine |
| **Microservices** | Small, independently deployable services |
| **Monolith** | Single deployable unit |
| **mTLS** | Mutual TLS — both client and server authenticate |
| **MVCC** | Multi-Version Concurrency Control (DB isolation) |

## O–Q

| Term | Definition |
|------|------------|
| **Observability** | Metrics + Logs + Traces (three pillars) |
| **OOM** | Out Of Memory |
| **PACELC** | If Partitioned → A/C; Else → Latency/Consistency |
| **Partition Tolerance** | System works despite network failures |
| **Pipelining** | Overlap stages (e.g., HTTP/2 multiplexing, PP in LLM) |
| **P95 / P99** | 95th/99th percentile latency |
| **Pub/Sub** | Publish-Subscribe — broadcast to all subscribers |
| **QPS** | Queries Per Second — throughput measure |

## R–S

| Term | Definition |
|------|------------|
| **RAG** | Retrieval-Augmented Generation — ground LLM in external data |
| **Rate Limiting** | Cap request rate (token bucket, leaky bucket, sliding window) |
| **RDBMS** | Relational Database Management System (PostgreSQL, MySQL) |
| **Read-Through** | Cache loads on miss (transparent to app) |
| **Read-Your-Writes** | User sees own updates immediately |
| **Redis** | In-memory data structure store (cache, queue, pub/sub) |
| **RESHADED** | Interview framework: Requirements, Estimation, Storage, HLD, API, Deep dives, Evaluation, Discussion |
| **Retry** | Re-attempt failed operation (exponential backoff + jitter) |
| **RPO** | Recovery Point Objective — max data loss tolerated |
| **RTO** | Recovery Time Objective — max downtime tolerated |
| **Saga** | Distributed transaction as chain of local steps + compensations |
| **Semantic Caching** | Cache by meaning (embedding similarity) not exact key match |
| **SLA / SLO / SLI** | Agreement / Objective / Indicator — reliability contracts |
| **Snowflake ID** | 64-bit unique ID: timestamp + worker + sequence |
| **SPOF** | Single Point of Failure |
| **SSD** | Solid State Drive (NVMe, SATA) |
| **Stateless** | No local session/state; any instance handles any request |

## T–Z

| Term | Definition |
|------|------------|
| **Tensor Parallelism** | Split weight matrices across GPUs (per layer) |
| **Throughput** | Requests processed per second (QPS, tokens/sec) |
| **TLB** | Translation Lookaside Buffer (CPU virtual memory cache) |
| **Token Bucket** | Rate limiting: tokens refill at rate, bucket = burst capacity |
| **TPU** | Tensor Processing Unit (Google's AI accelerator) |
| **TTFT** | Time To First Token — latency to first generated token (LLM) |
| **Two-Phase Commit (2PC)** | Distributed transaction: prepare → commit/abort |
| **Vector Database** | Stores embeddings; supports similarity search (ANN) |
| **vLLM** | High-throughput LLM serving with PagedAttention |
| **WebSocket** | Full-duplex persistent connection over HTTP upgrade |
| **Write-Through** | Write to cache AND DB synchronously |
| **Write-Behind** | Write to cache, async flush to DB |
| **ZooKeeper** | Coordination service (config, naming, leader election) |

---

## AI/ML Specific Terms

| Term | Definition |
|------|------------|
| **Batch Inference** | High-throughput, offline prediction (hours latency OK) |
| **Continuous Batching** | Dynamic batch scheduling per decode step (vLLM) |
| **Distillation** | Train smaller model to mimic larger teacher |
| **Embedding** | Dense vector representation of text/data |
| **Fine-tuning** | Adapt pre-trained model to specific task/data |
| **Guardrails** | Safety filters on LLM input/output |
| **Hallucination** | LLM generating factually incorrect but plausible output |
| **Inference** | Running model to generate predictions |
| **LoRA** | Low-Rank Adaptation (few MB adapters vs full model) |
| **Model Parallelism** | Split model across GPUs (TP, PP, EP) |
| **Online Serving** | Real-time prediction (ms latency) |
| **PagedAttention** | Virtual memory paging for KV cache (vLLM) |
| **Prompt Injection** | Malicious input overriding model instructions |
| **Quantization** | Reduce precision (FP16→INT8→INT4) for speed/size |
| **RAG** | Retrieval-Augmented Generation |
| **Real-time Inference** | Sub-second latency, streaming |
| **Speculative Decoding** | Small draft model proposes tokens; large model verifies |
| **Streaming Inference** | Return tokens as generated (SSE/WebSocket) |
| **Temperature** | Sampling randomness (0=deterministic, 1=creative) |
| **Token** | Unit of text (word piece, ~4 chars in English) |
| **Transformer** | Attention-based architecture (GPT, BERT, Llama) |

---

## Interview Frameworks

| Acronym | Expansion |
|---------|-----------|
| **RESHADED** | Requirements, Estimation, Storage, HLD, API, Deep dives, Evaluation, Discussion |
| **RED** | Rate, Errors, Duration (per-service metrics) |
| **USE** | Utilization, Saturation, Errors (per-resource metrics) |
| **RTO/RPO** | Recovery Time/Point Objective |
| **SLO/SLI/SLA** | Service Level Objective/Indicator/Agreement |

---

*This glossary is a living document. See something missing? [Contribute!](CONTRIBUTING.md)*