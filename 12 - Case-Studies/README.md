# Chapter 12: Case Studies

> **Prerequisite**: Chapters 1-11

---

## How to Use These Case Studies

```
┌─────────────────────────────────────────────────────────────────┐
│  CASE STUDY TEMPLATE (Follow this for each)                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. REQUIREMENTS (2 min)                                        │
│     • Functional + Non-functional                               │
│     • Clarifying questions                                      │
│                                                                 │
│  2. ESTIMATION (3 min)                                          │
│     • QPS, Storage, Bandwidth                                   │
│     • Binding constraint identification                         │
│                                                                 │
│  3. HIGH-LEVEL DESIGN (5 min)                                   │
│     • Component diagram                                         │
│     • Data flow                                                 │
│     • API contracts                                             │
│                                                                 │
│  4. DEEP DIVES (10 min)                                         │
│     • 2-3 critical components                                   │
│     • Trade-offs for each                                       │
│                                                                 │
│  5. TRADE-OFFS & FAILURE MODES (5 min)                          │
│     • SPOFs, scaling limits, cost                               │
│                                                                 │
│  TOTAL: ~25 min (interview: 35-40 min + 5-10 follow-up)         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 12.1 URL Shortener (TinyURL / Bitly)

### Requirements
- **Functional**: Shorten long URL → short code; redirect short code → original URL; custom aliases; expiration; analytics
- **Non-functional**: 100M DAU, 100:1 read:write, p99 < 50ms redirect, 99.99% availability, 5-year retention

### Estimation
```
Write QPS: 100M DAU × 10 URLs/user/day / 86400 × 3 (peak) ≈ 3,500 QPS
Read QPS: 3,500 × 100 = 350,000 QPS
Storage: 100M × 10 × 365 × 5 × 500 bytes ≈ 90 TB (with replication)
Bandwidth: 350K QPS × 500 bytes × 8 ≈ 1.4 Gbps
Binding constraint: READ QPS (350K) → needs aggressive caching
```

### High-Level Design
```
Client ──► DNS ──► CDN ──► Load Balancer ──► API Servers (Stateless)
                                                │
                        ┌───────────────────────┼───────────────────────┐
                        ▼                       ▼                       ▼
                 ┌─────────────┐         ┌─────────────┐         ┌─────────────┐
                 │   Redis     │         │   Redis     │         │ PostgreSQL  │
                 │  (Cache:    │         │  (Counter:  │         │  (Primary:  │
                 │  code→URL)  │         │  ID gen)    │         │  Write,     │
                 └─────────────┘         └─────────────┘         │  Read Repl) │
                                                                 └─────────────┘
```

### Deep Dives

**ID Generation**:
| Approach | Pros | Cons |
|----------|------|------|
| **Base62(random)** | Simple | Collision check needed |
| **Counter + Base62** | Sequential, no collision | Single point (Redis INCR) |
| **Snowflake** | Distributed, time-ordered | 64-bit, needs worker IDs |
| **UUID** | No coordination | Long (22+ chars) |

**Recommendation**: Snowflake (41-bit timestamp + 10-bit worker + 12-bit sequence) → 64-bit → Base62 ≈ 11 chars

**Caching Strategy**:
- Redirect path: Cache code→URL in Redis (TTL 24h, LRU)
- Write-through on create
- Cache hit rate target: >99% (read-heavy)

**Analytics**: Async pipeline → Click events → Kafka → Flink → Druid/ClickHouse

---

## 12.2 Social News Feed (Facebook / X Timeline)

### Requirements
- **Functional**: Post status, follow/unfollow, home timeline (ranked), profile timeline (reverse chronological), like/comment/share
- **Non-functional**: 1B DAU, 500 posts/user/day, 100:1 read:write, p99 timeline < 200ms, 99.9% availability

### Estimation
```
Write QPS: 1B × 500 / 86400 × 3 ≈ 17,000 QPS
Read QPS: 17,000 × 100 = 1.7M QPS
Storage (posts): 1B × 500 × 365 × 1KB × 3 ≈ 550 PB (cold: object store)
Storage (timelines): 1B × 500 × 30 days × 100 bytes ≈ 1.5 PB (hot)
Binding constraint: TIMELINE GENERATION (fan-out)
```

### Fan-out Strategies

```
FAN-OUT ON WRITE (Push)              FAN-OUT ON READ (Pull)
─────────────────────────            ────────────────────────

User posts ──────► Fan-out to        User requests timeline
all followers'                     ──────► Merge followees'
timelines (async)                      posts (at read time)
     │                                       │
     ▼                                       ▼
Timeline = Pre-computed              Timeline = Computed on-demand
Read: O(1) fast                        Read: O(followees) slow
Write: O(followers) expensive          Write: O(1) cheap
                                    
✅ Celebrity: millions of writes     ✅ Celebrity: no write amplification
❌ Inactive user: wasted writes      ❌ Inactive user: no waste
                                   
HYBRID (Recommended):                Twitter/X uses hybrid:
• Push to active users               • Push to "high-engagement" followers
• Pull for inactive/celebrity        • Pull for rest + merge
```

### Deep Dives

**Ranking**:
```
Score = w1×engagement + w2×recency + w3×affinity + w4×content_type
       │
       ├── Training: LambdaMART / LightGBM on historical data
       ├── Serving: Pre-score candidates (top 1000), rerank top 100
       └── Freshness: Re-score on like/comment (async update)
```

**Timeline Storage**:
```
Redis Sorted Set (ZSET):
  Key: timeline:{user_id}
  Score: ranking_score (float)
  Member: post_id (or serialized post)
  TTL: 30 days
  Max: 1000 entries (trim old)
```

---

## 12.3 Chat System (WhatsApp / Discord)

### Requirements
- **Functional**: 1:1 chat, group chat (up to 256/100K), presence (online/offline), message history, media, read receipts, push notifications
- **Non-functional**: 2B users, 100B msg/day, p99 message delivery < 200ms, 99.99% availability, E2E encryption

### Estimation
```
Msg QPS: 100B / 86400 × 3 ≈ 3.5M QPS
Connections: 2B users × 10% online ≈ 200M concurrent WebSocket connections
Storage: 100B × 365 × 3 × 500 bytes ≈ 55 PB (hot: recent, cold: S3)
Bandwidth: 3.5M × 1KB × 8 ≈ 28 Gbps
Binding constraint: WEBSOCKET CONNECTIONS + MESSAGE FAN-OUT
```

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  WHATSAPP-STYLE (Erlang/FreeBSD)                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Client ──► Load Balancer ──► Chat Servers (Stateless)          │
│                              │                                  │
│                    ┌─────────┴─────────┐                        │
│                    ▼                   ▼                        │
│             ┌───────────┐       ┌───────────┐                   │
│             │  Mnesia   │       │   Redis   │                   │
│             │ (Routing, │       │ (Presence,│                   │
│             │  Session) │       │  Offline) │                   │
│             └───────────┘       └───────────┘                   │
│                    │                   │                        │
│                    ▼                   ▼                        │
│             ┌───────────┐       ┌───────────┐                   │
│             │ PostgreSQL│       │   Kafka   │                   │
│             │ (Messages,│       │ (Events,  │                   │
│             │  Groups)  │       │  Analytics)                   │
│             └───────────┘       └───────────┘                   │
└─────────────────────────────────────────────────────────────────┘
```

### Deep Dives

**WebSocket Scaling**:
```
Connection → Chat Server (affinity via consistent hashing)
  │
  ├── Presence: Redis SET (user_id → server_id) + Pub/Sub for status
  ├── Message: 
  │     1:1 → Route to recipient's server → Push via WS
  │     Group → Fan-out to all members' servers → Push
  ├── Offline: Store in Kafka → Push notification service → APNs/FCM
  └── History: PostgreSQL (partitioned by conversation_id)
```

**Message Ordering**:
- Per-conversation sequence number (monotonic)
- Client acknowledges receipt → server advances ack
- Gap detection → resend missing

**E2E Encryption**:
- Signal Protocol (Double Ratchet)
- Keys generated client-side, never touch server
- Server only routes encrypted blobs

---

## 12.4 Video Streaming (YouTube / Netflix)

### Requirements
- **Functional**: Upload, transcode (multi-bitrate), adaptive streaming (HLS/DASH), CDN delivery, recommendations, search, live streaming
- **Non-functional**: 2B MAU, 500h uploaded/min, p99 start < 2s, 99.99% availability, global

### Estimation
```
Upload: 500h/min = 30,000 min/min ≈ 500 concurrent uploads
Storage: 500h × 60min × 24h × 365 × 1GB/h × 3 ≈ 40 EB (exabytes!) → Tiered
Transcode: 500h × 5 qualities = 2,500h compute/min → massive fleet
CDN: 1B hours watched/day → 1B × 2Mbps avg = 2 Tbps egress
Binding constraint: TRANSCODE PIPELINE + CDN COST
```

### Architecture

```
Upload ──► Object Store (S3) ──► Transcode Queue (Kafka) ──► Transcode Workers
                                                                    │
                                                    ┌───────────────┼───────────────┐
                                                    ▼               ▼               ▼
                                              1080p           720p            480p
                                                    │               │               │
                                                    ▼               ▼               ▼
                                              Object Store (Segmented: .ts files, .m3u8 manifest)
                                                    │
                                                    ▼
                                              CDN (Global PoPs)
                                                    │
                                                    ▼
                                              Client (Adaptive Bitrate)
```

### Deep Dives

**Transcoding Pipeline**:
```
Priority Queue:
  P0: Live / Breaking news (immediate)
  P1: Creator with >1M subs (minutes)
  P2: Standard (hours)
  P3: Archive / Long-tail (batch, spot instances)

Spot Instances for P2/P3: 70% cost savings, handle interruption
```

**Adaptive Bitrate (ABR)**:
```
Manifest (.m3u8):
  #EXTM3U
  #EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
  1080p/playlist.m3u8
  #EXT-X-STREAM-INF:BANDWIDTH=2500000,RESOLUTION=1280x720
  720p/playlist.m3u8
  ...

Client: Measure throughput → select quality → switch seamlessly
```

---

## 12.5 Ride Sharing (Uber / Lyft)

### Requirements
- **Functional**: Request ride, match driver, track trip, payment, ratings, surge pricing, ETA
- **Non-functional**: 100M MAU, 1M concurrent trips, p99 match < 5s, 99.99% availability, global

### Estimation
```
Request QPS: 1M trips/day × 3 (searches) / 86400 × 10 (peak) ≈ 350 QPS
Driver location updates: 1M drivers × 4/sec = 4M QPS
Matching: 350 QPS × 100 candidates = 35K scoring ops/sec
Storage: Trips (small), Driver locations (Redis, TTL 30s)
Binding constraint: DRIVER LOCATION UPDATES + MATCHING COMPUTE
```

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  UBER DISPATCH (Ringpop + Geohash)                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Rider App ──► API Gateway ──► Dispatch Service                 │
│                                      │                          │
│                    ┌─────────────────┼─────────────────┐        │
│                    ▼                 ▼                 ▼        │
│             ┌────────────┐   ┌────────────┐   ┌────────────┐    │
│             │ Geohash    │   │  Supply    │   │  Pricing   │    │
│             │ Index      │   │  Service   │   │  Service   │    │
│             │ (Redis)    │   │ (Kafka)    │   │ (Surge)    │    │
│             └────────────┘   └────────────┘   └────────────┘    │
│                    │                 │                 │        │
│                    └─────────────────┼─────────────────┘        │
│                                      ▼                          │
│                              ┌──────────────┐                   │
│                              │ Matching     │                   │
│                              │ Algorithm    │                   │
│                              └──────────────┘                   │
│                                      │                          │
│                    ┌─────────────────┼─────────────────┐        │
│                    ▼                 ▼                 ▼        │
│             Driver App          Rider App         Trip Service  │
└─────────────────────────────────────────────────────────────────┘
```

### Deep Dives

**Geospatial Indexing**:
```
Geohash (Base32, hierarchical):
  9q8yy (≈ 4.7km × 4.7km) → 9q8yyz (≈ 590m × 590m)
  
Query: Nearby drivers
  1. Geohash rider location (precision 5-6)
  2. Check that cell + 8 neighbors
  3. Filter by actual distance (Haversine)
  4. Score: distance × ETA × rating × acceptance_rate
```

**Matching Algorithm**:
```
Bipartite Matching (Hungarian / KM) → O(n³) too slow
→ GREEDY: Sort riders by wait time, assign best available driver
→ BATCH: Every 2-5 seconds, match all pending
→ REAL-TIME: Stream processing (Flink) for continuous matching
```

---

## 12.6 Rate Limiter (API Gateway)

### Requirements
- **Functional**: Limit requests per user/IP/key; multiple tiers (free: 100/min, pro: 1000/min); distributed
- **Non-functional**: p99 < 5ms overhead, 99.99% availability, horizontal scaling

### Design

```
Algorithm: Token Bucket (allows burst)
  • Bucket size = tier limit (burst capacity)
  • Refill rate = tier limit / 60 (per second)
  
Storage: Redis (sorted set or INCR + TTL)
  Key: ratelimit:{tier}:{user_id}
  Value: tokens_remaining, last_refill
  
Lua Script (atomic):
  local key = KEYS[1]
  local capacity = tonumber(ARGV[1])
  local refill_rate = tonumber(ARGV[2])
  local now = tonumber(ARGV[3])
  
  local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
  local tokens = tonumber(bucket[1]) or capacity
  local last = tonumber(bucket[2]) or now
  
  -- Refill
  local elapsed = now - last
  tokens = math.min(capacity, tokens + elapsed * refill_rate)
  
  if tokens >= 1 then
    tokens = tokens - 1
    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
    redis.call('EXPIRE', key, 86400)
    return {1, tokens}  -- allowed
  else
    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
    redis.call('EXPIRE', key, 86400)
    return {0, tokens}  -- rejected
  end
```

---

## 12.7 Notification System

### Requirements
- **Functional**: Push (mobile), email, SMS, in-app; preferences; templates; scheduling; retry; deduplication
- **Non-functional**: 1B notifications/day, p99 delivery < 30s, 99.9% delivery rate

### Design

```
Event Source ──► Notification Service ──► Preference Check ──► Fan-out
                                                              │
                                    ┌─────────────────────────┼─────────────────────────┐
                                    ▼                         ▼                         ▼
                              ┌───────────┐             ┌───────────┐             ┌───────────┐
                              │  Push     │             │  Email    │             │  SMS      │
                              │  Workers  │             │  Workers  │             │  Workers  │
                              │  (FCM/    │             │  (SendGrid│             │  (Twilio) │
                              │  APNs)    │             │  /SES)    │             └───────────┘
                              └───────────┘             └───────────┘
                                    │                         │
                                    └─────────────────────────┘
                                                      ▼
                                              ┌───────────┐
                                              │  Status   │
                                              │  Tracker  │
                                              │  (Kafka   │
                                              │  + DB)    │
                                              └───────────┘
```

### Deduplication
```
Key: notify:{user_id}:{event_type}:{entity_id}:{window}
Example: notify:user123:like:post456:20240115_10 (10-min window)
→ Only 1 notification per event per window
```

---

## 12.8 Distributed Counter / Unique ID Generator

### Requirements
- **Functional**: Generate globally unique 64-bit IDs; monotonically increasing (roughly); high throughput
- **Non-functional**: 1M IDs/sec, 99.999% availability, no coordination bottleneck

### Snowflake Design

```
64-bit ID Structure:
┌─────────────────────────────────────────────────────────────────┐
│  1 bit  │    41 bits     │   10 bits    │      12 bits          │
│  (sign) │  Timestamp     │  Worker ID   │  Sequence Number      │
│  (0)    │  (ms since     │  (1024       │  (4096 per ms per     │
│         │   epoch)       │   workers)   │   worker)             │
└─────────────────────────────────────────────────────────────────┘

Timestamp: 41 bits = 69 years @ ms precision
Worker ID: 10 bits = 1024 workers (datacenter + machine)
Sequence: 12 bits = 4096 IDs/ms/worker = 4M IDs/sec/worker

Total: 1024 workers × 4M = 4B IDs/sec capacity
```

### Implementation

```go
type Snowflake struct {
    epoch     int64    // Custom epoch (e.g., 2024-01-01)
    workerID  int64    // 0-1023
    sequence  int64    // 0-4095
    lastTime  int64
    mu        sync.Mutex
}

func (s *Snowflake) NextID() int64 {
    s.mu.Lock()
    defer s.mu.Unlock()
    
    now := time.Now().UnixMilli() - s.epoch
    
    if now == s.lastTime {
        s.sequence = (s.sequence + 1) & 0xFFF
        if s.sequence == 0 {
            // Sequence exhausted, wait for next ms
            for now <= s.lastTime {
                now = time.Now().UnixMilli() - s.epoch
            }
        }
    } else {
        s.sequence = 0
    }
    
    s.lastTime = now
    
    return (now << 22) | (s.workerID << 12) | s.sequence
}
```

---

## 12.9 Additional Case Studies (Brief)

| System | Key Challenge | Core Pattern |
|--------|---------------|--------------|
| **Search Autocomplete** | Prefix search, ranking | Trie + Redis sorted sets, debouncing |
| **Web Crawler** | Politeness, dedup, scale | Frontier queue (priority), bloom filter, DNS cache |
| **Ad Tech / RTB** | Sub-10ms bidding | In-memory, pre-computed, edge deployment |
| **Metrics/Monitoring** | High cardinality, retention | Downsampling, columnar (ClickHouse), tiered storage |
| **Job Scheduler** | Exactly-once, cron + ad-hoc | Leader election, persistent queue, idempotency |
| **Leaderboard** | Real-time ranking | Redis sorted set, periodic snapshot to DB |

---

## ✅ Exercises

1. **Full Design**: Pick one case study. Do full 25-min design on whiteboard/paper. Time yourself.
2. **Compare**: Design both fan-out-on-write and fan-out-on-read for news feed. Compare at 1B users.
3. **Extend**: Add "Stories" (24h ephemeral) to chat system. What changes?
4. **Optimize**: URL shortener at 1B DAU. What breaks? How to fix?

---

## 🔁 Review Prompts

- For each case study: What's the binding constraint? How did you identify it?
- When do you choose fan-out on write vs read?
- How does WebSocket scaling differ from HTTP scaling?
- What makes ride-sharing matching hard? (Geospatial + real-time + fairness)

---

## 📚 Further Reading

- *System Design Interview Vol. 1 & 2* — Alex Xu (detailed case studies)
- [WhatsApp Architecture](https://www.infoq.com/articles/whatsapp-architecture/) — InfoQ
- [Discord Architecture](https://discord.com/blog/how-discord-stores-trillions-of-messages) — Discord Blog
- [Uber Engineering](https://eng.uber.com/) — Dispatch, Geospatial, Matching
- [Netflix Tech Blog](https://netflixtechblog.com/) — CDN, Transcoding, Chaos
- [Twitter Engineering](https://blog.twitter.com/engineering) — Timeline, Search, Ads