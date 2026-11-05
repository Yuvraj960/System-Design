# Chapter 5: Caching & CDN

## 5.1 Why Caching Matters

```
┌─────────────────────────────────────────────────────────────────┐
│  THE CACHE HIERARCHY (Latency →)                                │
└─────────────────────────────────────────────────────────────────┘

L1/L2 CPU Cache     ──────►  0.5–7 ns
Main Memory (RAM)   ──────►  100 ns
Redis / Memcached   ──────►  0.5–1 ms (network RTT)
SSD                 ──────►  16 µs – 1 ms
Database (Indexed)  ──────►  1–10 ms
Database (Full Scan)──────►  10–100+ ms
Cross-Region        ──────►  50–150 ms

RULE: Each cache layer is ~100× faster than the next.
      A cache hit at Redis saves ~10 ms vs DB.
      A cache hit at CDN saves ~100 ms vs origin.
```

---

## 5.2 Cache Placement

| Layer | Location | What It Caches | Latency | Scope |
|-------|----------|----------------|---------|-------|
| **Client** | Browser / Mobile app | Static assets, API responses | 0 ms (local) | Per-user |
| **CDN / Edge** | Global PoPs | Static assets, HTML, API (edge) | 10–50 ms | Global |
| **API Gateway** | Ingress | Auth results, rate limit counters | <1 ms | Per-request |
| **Application** | In-process (LRU map) | Hot computed values, config | ~0 ns | Per-instance |
| **Distributed** | Redis Cluster, Memcached | Session, query results, objects | 0.5–1 ms | Cluster-wide |
| **Database** | Query cache, buffer pool | Pages, query plans | In-DB | Per-DB |

---

## 5.3 Caching Patterns

### Cache-Aside (Lazy Loading) — Most Common

```
Application                    Cache                    Database
    │                            │                         │
    ├─ GET key ─────────────────►│                         │
    │◄── MISS ───────────────────┤                         │
    │                            │                         │
    ├─ GET key ───────────────────────────────────────────►│
    │◄── VALUE ────────────────────────────────────────────┤
    │                            │                         │
    ├─ SET key, value ──────────►│                         │
    │                            │                         │
```

```python
def get_user(user_id):
    # 1. Check cache
    user = redis.get(f"user:{user_id}")
    if user:
        return json.loads(user)
    
    # 2. Cache miss → query DB
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    
    # 3. Populate cache
    redis.setex(f"user:{user_id}", ttl=300, value=json.dumps(user))
    return user
```

| Pros | Cons |
|------|------|
| Simple, resilient (DB is source of truth) | First request after miss is slow |
| Cache only what's needed | Stale data until TTL/invalidation |
| Works with any DB | Cache stampede on hot key expiry |

---

### Read-Through

```
Application ──► Cache ──► Database
                │
                └─ Cache loads on miss (transparent to app)
```

- Cache sits "in-line" — application only talks to cache
- Cache library handles DB fallback
- **Use when**: Cache is the primary data access layer

---

### Write-Through

```
Application ──► Cache ──► Database
                │          │
                │          └─ Synchronous write
                └─ Update cache
```

- Strong consistency (cache never stale)
- Write latency = cache + DB
- **Use when**: Consistency critical, write volume moderate

---

### Write-Behind (Write-Back)

```
Application ──► Cache ──► [Async Queue] ──► Database
                │
                └─ Acknowledge immediately
                   Flush to DB in background
```

- Fastest writes
- Risk: Data loss if cache fails before flush
- **Use when**: Write-heavy, can tolerate eventual consistency (analytics, logs)

---

### Write-Around

```
Application ──► Database
                │
                └─ Cache populated on READ (cache-aside)
```

- Writes bypass cache (avoid cache pollution)
- Good for write-once, read-never data

---

## 5.4 Cache Invalidation Strategies

| Strategy | How It Works | Consistency | Complexity |
|----------|--------------|-------------|------------|
| **TTL (Time-To-Live)** | Auto-expire after N seconds | Eventual (stale until expiry) | Lowest |
| **Active Invalidation** | On write, delete/update cache keys | Strong (if done correctly) | Medium |
| **Pub/Sub Invalidation** | Write publishes event → cache subscribers invalidate | Strong | Higher |
| **Versioned Keys** | `key:v{version}` — write new version, old expires | Strong | Medium |

### Cache Stampede (Thundering Herd)

```
Time ──────────────────────────────────────────────────────────►
Hot key expires
       │
       ▼
10,000 requests hit cache MISS simultaneously
       │
       ▼
All 10,000 query database → DB OVERLOAD
```

### Prevention Techniques

```python
# 1. Probabilistic Early Expiration (prevent sync expiry)
def get_with_early_expiry(key, ttl=300):
    value, expiry = redis.get_with_ttl(key)
    if value and (expiry - now()) > ttl * 0.1:  # Not near expiry
        return value
    # Near expiry → probabilistic refresh
    if random.random() < 0.01:  # 1% chance to refresh early
        refresh_cache_async(key)
    return value

# 2. Mutex / Lock (single recomputation)
def get_with_lock(key, ttl=300):
    value = redis.get(key)
    if value:
        return value
    
    # Try to acquire lock
    lock = redis.set(f"lock:{key}", "1", nx=True, ex=10)
    if lock:
        # We won the lock → recompute
        value = compute_value()
        redis.setex(key, ttl, value)
        redis.delete(f"lock:{key}")
        return value
    else:
        # Wait for lock holder to finish
        time.sleep(0.05)
        return get_with_lock(key, ttl)  # Retry

# 3. Lease-based (Redis SETNX with value = computation future)
```

---

## 5.5 Eviction Policies

| Policy | Algorithm | Best For | Memory Overhead |
|--------|-----------|----------|-----------------|
| **LRU** (Least Recently Used) | Doubly-linked list + hash map | General purpose, temporal locality | Medium |
| **LFU** (Least Frequently Used) | Frequency counter + min-heap | Stable popularity (e.g., top items) | High |
| **LRU-K** | Track last K accesses | Better than LRU for scans | Higher |
| **ARC** (Adaptive Replacement) | Dynamic LRU/LFU mix | Mixed workloads | High |
| **TTL-only** | Expire by time | Naturally stale data (config, feeds) | None |
| **Random** | Pick random entry | Memcached default at scale | None |

> **Redis**: `maxmemory-policy` — `allkeys-lru`, `volatile-lru`, `allkeys-lfu`, `noeviction`
> **Memcached**: LRU per slab class (slab allocation)

---

## 5.6 Distributed Caching

### Redis Cluster

```
┌─────────────────────────────────────────────────────────────────┐
│  REDIS CLUSTER: 16384 HASH SLOTS                                │
└─────────────────────────────────────────────────────────────────┘

Slot 0–5460    │    Slot 5461–10922    │    Slot 10923–16383
    │          │          │            │          │
    ▼          ▼          ▼            ▼          ▼
┌───────┐  ┌───────┐  ┌───────┐   ┌───────┐  ┌───────┐
│ Master│  │ Master│  │ Master│   │ Master│  │ Master│
│   A   │  │   B   │  │   C   │   │   D   │  │   E   │
│       │  │       │  │       │   │       │  │       │
│ Repl 1│  │ Repl 1│  │ Repl 1│   │ Repl 1│  │ Repl 1│
└───────┘  └───────┘  └───────┘   └───────┘  └───────┘

Client → hash(key) % 16384 = slot → route to master for slot
```

- **Hash slots** (not consistent hashing) — fixed 16384 slots
- **Redis Cluster** — client-side routing (MOVED/ASK redirection)
- **Replication** — async, 1+ replicas per master
- **Failover** — replica promotion via Raft-like consensus

### Hot Key Problem

```
┌─────────────────────────────────────────────────────────────────┐
│  HOT KEY: One key receives disproportionate traffic            │
└─────────────────────────────────────────────────────────────────┘

Solutions:
  1. Local cache (in-process) + pub/sub invalidation
  2. Key splitting: key → key:1, key:2, key:3 (read from random)
  3. Request collapsing (single flight for same key)
  4. Move hot key to dedicated cache cluster
```

---

## 5.7 CDN (Content Delivery Network)

### Push vs Pull CDN

| Type | How It Works | Control | Best For |
|------|--------------|---------|----------|
| **Pull (Origin Pull)** | CDN fetches on first request | Automatic | Dynamic content, large catalog |
| **Push** | You upload to CDN | Full control | Static assets, pre-positioning |

### CDN Architecture

```
User ──► DNS (GeoDNS) ──► Nearest PoP ──► Cache Hit? ──► Return
                              │
                              ▼ Miss
                         Origin Shield ──► Origin Server
                              │
                              ▼
                         Cache & Return
```

### Cache Keys & Invalidation

```
Cache Key = Host + Path + Query Params + Headers (Vary)

Invalidation:
  1. Purge API (instant, per URL or tag)
  2. TTL (stale-while-revalidate)
  3. Versioned URLs: /asset.v123.js (preferred for static)
```

### Edge Compute

```
User Request ──► CDN Edge ──► [Edge Function] ──► Origin / Cache
                      │
                      ├── A/B testing
                      ├── Auth at edge
                      ├── Image optimization
                      ├── Bot detection
                      └── Geo-routing
```

---

## 5.8 Semantic Caching (AI-Specific)

```
Traditional Cache:     Exact key match → "What is 2+2?" ≠ "What is two plus two?"

Semantic Cache:        Embedding similarity → "What is 2+2?" ≈ "What is two plus two?"
                       Vector search in cache → Return cached answer if similarity > 0.95
```

- **Use case**: LLM responses, RAG queries
- **Implementation**: Store (embedding, response) in vector DB; on query, search cache first
- **Benefit**: 30–70% cache hit rate for similar prompts

---

## ✅ Exercises

1. **Pattern Choice**: User profile service — read-heavy, writes on profile update. Which pattern?
2. **Stampede**: Your Redis cache expires a hot key (homepage config) at midnight. 50K req/s hit DB. Fix it.
3. **Eviction**: 10GB cache, 100M keys. Access pattern: 80% of requests hit 20% of keys. LRU or LFU?
4. **CDN**: You deploy a React SPA. How do you cache HTML vs JS/CSS vs API responses?

---

## 🔁 Review Prompts

- Compare cache-aside vs read-through vs write-through. When does each shine?
- What causes cache stampede? Name 3 prevention techniques.
- How does Redis Cluster distribute data? How does it differ from consistent hashing?
- What's the difference between push and pull CDN? When to use each?

---

## 📚 Further Reading

- [Redis Documentation — Caching Patterns](https://redis.io/docs/manual/patterns/)
- *Designing Data-Intensive Applications*, Ch 4 — Kleppmann
- [High Performance Browser Networking](https://hpbn.co/) — Ilya Grigorik
- [Cloudflare Caching Docs](https://developers.cloudflare.com/cache/)