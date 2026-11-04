# Chapter 2: Networking & Communication Protocols

## 2.1 Networking Fundamentals

### OSI vs TCP/IP Model

```
OSI (7 layers)              TCP/IP (4 layers)          Example Protocols
────────────────────────────────────────────────────────────────────────────
7. Application                                 HTTP, DNS, gRPC, SMTP
6. Presentation                                TLS/SSL, Compression
5. Session                                     WebSocket, RPC
────────────────────────────────────────────────────────────────────────────
4. Transport                                   TCP, UDP, QUIC
3. Network                3. Internet          IP, ICMP, BGP
2. Data Link              2. Network Access    Ethernet, WiFi, MAC
1. Physical
```

### Key Protocols at Each Layer

| Layer | Protocol | Purpose | Key Characteristics |
|-------|----------|---------|---------------------|
| **Application** | HTTP/1.1 | Request-response | Text, stateless, head-of-line blocking |
| | HTTP/2 | Multiplexed streams | Binary, header compression, server push |
| | HTTP/3 (QUIC) | UDP-based, 0-RTT | No HOL blocking, connection migration |
| | gRPC | RPC framework | HTTP/2, Protobuf, streaming, contracts |
| | WebSocket | Full-duplex | Persistent, bidirectional, ~2KB overhead |
| | DNS | Name resolution | UDP/TCP port 53, hierarchical, cached |
| **Transport** | TCP | Reliable, ordered | 3-way handshake, congestion control |
| | UDP | Unreliable, fast | No handshake, no ordering, low overhead |
| | QUIC | UDP + reliability | 0-RTT, streams, encryption built-in |

---

## 2.2 DNS — The Internet's Phonebook

### Resolution Flow

```
User types "api.example.com"
        │
        ▼
┌───────────────────┐
│ Browser Cache     │ ──► Hit? Return IP
└─────────┬─────────┘
          │ Miss
          ▼
┌───────────────────┐
│ OS Cache          │ ──► Hit? Return IP
└─────────┬─────────┘
          │ Miss
          ▼
┌───────────────────┐
│ Router/ISP Cache  │ ──► Hit? Return IP
└─────────┬─────────┘
          │ Miss
          ▼
┌───────────────────┐
│ Root DNS (.)      │ ──► .com TLD
└─────────┬─────────┘
          ▼
┌───────────────────┐
│ TLD DNS (.com)    │ ──► example.com authoritative NS
└─────────┬─────────┘
          ▼
┌───────────────────┐
│ Authoritative NS  │ ──► A/AAAA/CNAME record → IP
└───────────────────┘
```

### DNS Record Types

| Type | Purpose | Example |
|------|---------|---------|
| **A** | IPv4 address | `api.example.com → 192.0.2.1` |
| **AAAA** | IPv6 address | `api.example.com → 2001:db8::1` |
| **CNAME** | Alias | `cdn.example.com → d123.cloudfront.net` |
| **TXT** | Text data | SPF, DKIM, verification |
| **MX** | Mail exchanger | `example.com → mail.example.com` |
| **NS** | Name server | `example.com → ns1.provider.com` |
| **SRV** | Service location | `_grpc._tcp.example.com → host:port` |

### DNS in System Design

- **TTL** controls cache duration — lower for failover, higher for stability
- **GeoDNS** / **GSLB** routes to nearest region
- **Health checks** can remove failing IPs from DNS (but TTL limits speed)
- **CNAME at apex** not allowed — use ALIAS/ANAME (provider-specific)

---

## 2.3 Load Balancing

### L4 vs L7 Load Balancing

| Aspect | L4 (Transport) | L7 (Application) |
|--------|----------------|------------------|
| **Visibility** | IP + Port only | Full HTTP request (headers, path, body) |
| **Decisions** | 5-tuple hash | Path, headers, cookies, JWT claims |
| **TLS Termination** | Passthrough (or separate) | Native (offloads app servers) |
| **Features** | Limited | Routing, auth, rate limit, rewrites |
| **Latency** | Lower | Slightly higher |
| **Examples** | NLB, IPVS, LVS | ALB, NGINX, Envoy, HAProxy |

### Load Balancing Algorithms

| Algorithm | How It Works | Best For |
|-----------|--------------|----------|
| **Round Robin** | Cycle through backends | Homogeneous, equal capacity |
| **Weighted Round Robin** | Proportional to weight | Mixed instance sizes |
| **Least Connections** | Fewest active connections | Long-lived, variable duration |
| **Least Response Time** | Lowest latency + fewest conns | Heterogeneous performance |
| **IP Hash / Consistent Hash** | `hash(client_ip) % N` | Session affinity (sticky) |
| **Power of Two Choices** | Pick 2 random, use least loaded | Near-optimal, simple |

> **Power of Two Choices**: Randomly select 2 backends, route to the one with fewer connections. Mathematically near-optimal, used in production at Google, Uber.

### Health Checks

```
Active Health Check:    LB ──► GET /health ──► Backend (every 10-30s)
Passive Health Check:   LB observes 5xx/timeout ──► Mark unhealthy
```

| Check Type | Pros | Cons |
|------------|------|------|
| **Active** | Proactive, catches silent failures | Extra traffic, false positives |
| **Passive** | No overhead, real traffic | Reactive, needs traffic to detect |

---

## 2.4 API Paradigms

### REST vs GraphQL vs gRPC

| Aspect | REST | GraphQL | gRPC |
|--------|------|---------|------|
| **Paradigm** | Resource-oriented | Query-oriented | RPC / Contract-first |
| **Transport** | HTTP/1.1 or HTTP/2 | HTTP/1.1 or HTTP/2 | HTTP/2 (required) |
| **Serialization** | JSON | JSON | Protobuf (binary) |
| **Schema** | OpenAPI/Swagger | SDL (Schema Definition) | .proto files |
| **Fetching** | Multiple round trips | Single request, exact data | Streaming, bidirectional |
| **Caching** | Native HTTP caching | Complex (query-level) | Limited (custom) |
| **Versioning** | URL / Header | Schema evolution | Package versioning |
| **Browser Support** | Native | Native (fetch) | Requires proxy (gRPC-Web) |
| **Best For** | Public APIs, CRUD | Complex UIs, mobile | Microservice communication |

### When to Use Each

```
┌─────────────────────────────────────────────────────────────────┐
│  CHOOSE REST WHEN:                                              │
│  • Public / partner APIs (broad compatibility)                  │
│  • Simple CRUD, resource-oriented                               │
│  • Team lacks gRPC/GraphQL expertise                            │
│  • HTTP caching is valuable                                     │
├─────────────────────────────────────────────────────────────────┤
│  CHOOSE GRAPHQL WHEN:                                           │
│  • Complex UIs with varying data needs                          │
│  • Mobile clients (bandwidth-constrained)                       │
│  • Multiple frontend teams, single backend                      │
│  • Need to avoid over/under-fetching                            │
├─────────────────────────────────────────────────────────────────┤
│  CHOOSE gRPC WHEN:                                              │
│  • Service-to-service (internal)                                │
│  • High throughput, low latency                                 │
│  • Streaming required (bidirectional, server/client)            │
│  • Strong contracts, code generation                            │
│  • Polyglot microservices                                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2.5 Real-Time Communication

### WebSocket vs SSE vs Long Polling

| Feature | WebSocket | SSE (Server-Sent Events) | Long Polling |
|---------|-----------|--------------------------|--------------|
| **Direction** | Full-duplex | Server → Client only | Client → Server (poll) |
| **Protocol** | `ws://` / `wss://` | `text/event-stream` over HTTP | Standard HTTP |
| **Max Connections** | ~65K per IP (browser limit) | Unlimited (HTTP/1.1) / High (HTTP/2) | Limited |
| **Auto-Reconnect** | Manual | **Native** (EventSource) | Manual |
| **Binary Data** | Native | Base64 encoded | Base64 encoded |
| **Proxy/Firewall** | Sometimes blocked | Works over HTTP | Works everywhere |
| **Best For** | Chat, gaming, collab | Feeds, notifications, dashboards | Legacy, simple cases |

### WebSocket Connection Lifecycle

```
Client                         Server
  │                               │
  ├─ HTTP GET /chat (Upgrade) ───►│
  │◄── 101 Switching Protocols ───┤
  │                               │
  ├─ WS Frame: "Hello" ──────────►│
  │◄── WS Frame: "Welcome" ───────┤
  │              ...              │
  ├─ Ping ───────────────────────►│
  │◄── Pong ──────────────────────┤
  │                               │
  ├─ Close Frame ────────────────►│
  │◄── Close Frame ───────────────┤
```

---

## 2.6 Proxies: Forward vs Reverse

| Type | Position | Purpose | Examples |
|------|----------|---------|----------|
| **Forward Proxy** | Client side | Mask client IP, filter content, cache | Corporate proxy, VPN |
| **Reverse Proxy** | Server side | TLS termination, load balance, cache, protect | NGINX, Envoy, ALB, Cloudflare |

### Reverse Proxy Responsibilities

```
Client ──► [Reverse Proxy] ──► App Servers
              │
              ├── TLS Termination (offload CPU)
              ├── Load Balancing (L7 routing)
              ├── Request/Response Transformation
              ├── Rate Limiting / Auth
              ├── Caching (static + dynamic)
              ├── Compression (gzip/brotli)
              ├── WAF / Security Headers
              └── Observability (metrics, logs, traces)
```

---

## 2.7 API Gateway

### Core Functions

| Function | Description |
|----------|-------------|
| **Routing** | Path/host/header-based to backend services |
| **Authentication** | JWT validation, OAuth2, API keys, mTLS |
| **Rate Limiting** | Per-client, per-endpoint, token bucket |
| **Request/Response Transform** | Protocol translation, field mapping |
| **Aggregation** | Fan-out to multiple services, compose response |
| **Circuit Breaking** | Fail fast, fallback responses |
| **Observability** | Metrics, logging, tracing injection |

### API Gateway vs Service Mesh

| Aspect | API Gateway | Service Mesh (Istio/Linkerd) |
|--------|-------------|------------------------------|
| **Scope** | North-South (ingress) | East-West (service-to-service) |
| **Layer** | L7 | L4/L7 (sidecar) |
| **Features** | Auth, rate limit, routing | mTLS, retries, telemetry, traffic split |
| **Complexity** | Lower | Higher (sidecar per pod) |
| **Use Together** | ✅ Gateway at edge, mesh inside | |

---

## 2.8 Communication Patterns

### Synchronous (Request-Response)

```
Client ──► [Request] ──► Service ──► [Response] ──► Client
              │
              └── Blocks until response or timeout
```

- **Pros**: Simple, immediate consistency, natural for CRUD
- **Cons**: Coupling, latency = sum of all hops, cascade failures

### Asynchronous (Message-Based)

```
Producer ──► [Message] ──► Queue/Topic ──► Consumer(s)
              │
              └── Fire-and-forget (or acked)
```

| Pattern | Delivery | Ordering | Use Case |
|---------|----------|----------|----------|
| **Message Queue** | At-least-once, 1 consumer | Per-queue | Task distribution, commands |
| **Pub/Sub** | At-least-once, all subscribers | Per-partition | Events, notifications, fan-out |
| **Event Streaming** | At-least-once, replayable | Per-partition | Audit, analytics, CDC |

---

## ✅ Exercises

1. **DNS**: Your service fails over to a backup region. How do you update DNS? What's the minimum failover time with TTL=60s?
2. **LB Algorithm**: You have 3 backends: 2× large (8 vCPU), 1× small (2 vCPU). Which algorithm? How to configure?
3. **API Choice**: Design a mobile app backend. Users fetch personalized feeds with varying fields. REST, GraphQL, or gRPC? Justify.
4. **Real-time**: Stock trading platform needs sub-100ms price updates to 100K concurrent users. WebSocket or SSE? Why?
5. **Protocol**: Service A calls Service B 100×/sec. gRPC or REST? Calculate bandwidth difference for 1KB payload.

---

## 🔁 Review Prompts

- What's the difference between L4 and L7 load balancing? When do you use each?
- Why does HTTP/2 solve head-of-line blocking? Why does HTTP/3 solve it better?
- When would you use a forward proxy vs reverse proxy?
- What are the trade-offs between WebSocket and SSE for server-to-client streaming?

---

## 📚 Further Reading

- [HTTP/2 RFC 7540](https://httpwg.org/specs/rfc7540.html)
- [HTTP/3 RFC 9114](https://httpwg.org/specs/rfc9114.html)
- [gRPC Documentation](https://grpc.io/docs/)
- *Designing Data-Intensive Applications*, Ch 4 (Encoding & Evolution)
- [The Tail at Scale](https://research.google/pubs/pub40801/) — Dean & Barroso