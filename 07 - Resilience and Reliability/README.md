# Chapter 7: Resilience & Reliability Patterns

> **Prerequisite**: Chapters 1-6

---

## 7.1 Design for Failure — The Mindset

```
┌─────────────────────────────────────────────────────────────────┐
│  "EVERYTHING FAILS, ALL THE TIME"                               │
│                                                                 │
│  • Network partitions (switch fails, cable cut, GC pause)       │
│  • Hardware fails (disk, NIC, power, memory)                    │
│  • Software bugs (memory leaks, deadlocks, infinite loops)      │
│  • Dependency failures (downstream service, DB, DNS)            │
│  • Human error (bad deploy, config typo, capacity miscalc)      │
│  • Natural disasters (datacenter flood, power outage)           │
│                                                                 │
│  YOUR JOB: Define what happens WHEN (not if) each fails.        │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7.2 Timeout — The Foundation

```
┌─────────────────────────────────────────────────────────────────┐
│  NEVER WAIT FOREVER                                             │
└─────────────────────────────────────────────────────────────────┘

Timeout = max time you're willing to wait for a response

┌───────────────────────────────────────────────────────────────────┐
│  TIMEOUT HIERARCHY                                                │
├───────────────────────────────────────────────────────────────────┤
│  Client timeout        │  5–30s    │ User-facing request          │
│  API Gateway timeout   │  30–60s   │ Upstream aggregate           │
│  Service-to-service    │  1–5s     │ Internal RPC                 │
│  Database query        │  100ms–5s │ Per-query                    │
│  Cache (Redis)         │  10–100ms │ Fast path                    │
│  External API          │  5–30s    │ Third-party dependency       │
└───────────────────────────────────────────────────────────────────┘

RULE: Timeout at each layer should be LESS than caller's timeout.
      Client(30s) > Gateway(25s) > Service(10s) > DB(5s)
```

### Timeout Types

| Type | Description |
|------|-------------|
| **Connection Timeout** | Time to establish TCP connection |
| **Request Timeout** | Time from request sent to response received |
| **Idle Timeout** | Max time with no data transferred |
| **Global Timeout** | Total budget for entire request chain |

---

## 7.3 Retry with Exponential Backoff

```
┌─────────────────────────────────────────────────────────────────┐
│  RETRY STORM: 100 clients × 3 retries = 300× load on failing    │
│  service. Exponential backoff + jitter prevents this.           │
└─────────────────────────────────────────────────────────────────┘

Algorithm:
  delay = base_delay × 2^attempt + random_jitter
  max_delay = cap (e.g., 30s)
  max_attempts = 3–5

Example (base=100ms, jitter=±25%):
  Attempt 1: 100ms  ± 25ms
  Attempt 2: 200ms  ± 50ms
  Attempt 3: 400ms  ± 100ms
  Attempt 4: 800ms  ± 200ms
  Attempt 5: 1600ms ± 400ms (capped at max_delay)
```

### Retry Best Practices

```python
# ✅ GOOD: Retry only on transient errors
def call_with_retry(func, max_attempts=3, base_delay=0.1):
    for attempt in range(max_attempts):
        try:
            return func()
        except TransientError as e:  # Timeout, 5xx, network error
            if attempt == max_attempts - 1:
                raise
            delay = base_delay * (2 ** attempt) + random.uniform(0, 0.1)
            time.sleep(delay)
        except PermanentError as e:  # 4xx, validation, auth
            raise  # Don't retry!

# ❌ BAD: Retry on everything
# ❌ BAD: No jitter (thundering herd)
# ❌ BAD: No max attempts (infinite loop)
# ❌ BAD: Retry non-idempotent operations without idempotency key
```

---

## 7.4 Circuit Breaker

```
┌─────────────────────────────────────────────────────────────────┐
│  CIRCUIT BREAKER STATE MACHINE                                  │
└─────────────────────────────────────────────────────────────────┘

         ┌─────────────┐     Failure threshold      ┌─────────┐
         │   CLOSED    │ ─────────────────────────► │  OPEN   │
         │ (Normal)    │   (e.g., 5 failures in 10s)│ (Fail   │
         │ Requests    │                            │  Fast)  │
         │ pass through│                            │ Returns │
         └─────────────┘                            │ error   │
              ▲                                     │ immed.  │
              │                                     └────┬────┘
              │                                          │
              │              Success threshold           │
              │              (e.g., 3 successes)         │
              │                                          ▼
              │                                  ┌─────────────┐
              └──────────────────────────────────│  HALF-OPEN  │
                                                 │ (Test probe)│
                                                 │ 1 request   │
                                                 │ passes?     │
                                                 └─────────────┘
```

### Implementation

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, success_threshold=3, timeout=30):
        self.failure_threshold = failure_threshold
        self.success_threshold = success_threshold
        self.timeout = timeout
        self.state = "CLOSED"
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None
    
    def call(self, func):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.timeout:
                self.state = "HALF_OPEN"
            else:
                raise CircuitOpenError()
        
        try:
            result = func()
            self.on_success()
            return result
        except Exception as e:
            self.on_failure()
            raise
    
    def on_success(self):
        self.failure_count = 0
        if self.state == "HALF_OPEN":
            self.success_count += 1
            if self.success_count >= self.success_threshold:
                self.state = "CLOSED"
                self.success_count = 0
    
    def on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = "OPEN"
            self.success_count = 0
```

### When to Trip

| Metric | Threshold | Window |
|--------|-----------|--------|
| **Error Rate** | > 50% | 10s rolling |
| **Latency p99** | > 5s | 10s rolling |
| **Timeout Rate** | > 20% | 10s rolling |
| **Concurrent Failures** | > N | Immediate |

---

## 7.5 Bulkhead (Isolation)

```
┌─────────────────────────────────────────────────────────────────┐
│  BULKHEAD: ISOLATE FAILURE DOMAINS                              │
└─────────────────────────────────────────────────────────────────┘

WITHOUT BULKHEAD:                    WITH BULKHEAD:
                                   
All requests share               ┌─────────┐ ┌─────────┐ ┌─────────┐
thread pool:                     │ Pool A  │ │ Pool B  │ │ Pool C  │
  100 threads                    │ (10)    │ │ (30)    │ │ (60)    │
    │                            │ Service │ │ Service │ │ Service │
    ▼                            │   A     │ │   B     │ │   C     │
Service A fails ──►              └─────────┘ └─────────┘ └─────────┘
all threads blocked                 │           │           │
Service B, C also fail              ▼           ▼           ▼
                                    OK          OK          OK
```

### Bulkhead Types

| Type | Isolation | Example |
|------|-----------|---------|
| **Thread Pool** | Per-service thread pool | Hystrix, Resilience4j |
| **Connection Pool** | Per-downstream pool | HTTP client pools |
| **Process/Container** | Separate deployments | Microservices |
| **Cluster/Region** | Separate infrastructure | Multi-AZ, multi-region |

---

## 7.6 Rate Limiting

### Algorithms

| Algorithm | Behavior | Burst Handling | Implementation |
|-----------|----------|----------------|----------------|
| **Token Bucket** | Tokens refill at rate, bucket = burst capacity | ✅ Allows bursts up to bucket size | Redis `INCR` + Lua / `token_bucket` lib |
| **Leaky Bucket** | Fixed-rate output queue | ❌ Smooths bursts | Queue + fixed drain |
| **Fixed Window** | Count per time window | ❌ Burst at boundaries | Redis `INCR` + TTL |
| **Sliding Window** | Rolling window count | ✅ Accurate | Sorted set / `sliding_window` lib |
| **Sliding Log** | Exact request timestamps | ✅ Most accurate | High memory |

### Token Bucket (Recommended)

```
Bucket: capacity = 100, refill = 10/sec

Client A (idle 10s): 100 tokens accumulated
  → Burst 100 requests allowed instantly
  → Then 10/sec sustained

Client B (steady 5/sec): Never hits limit
```

### Placement

```
┌─────────────────────────────────────────────────────────────────┐
│  RATE LIMIT AT THE EDGE (API Gateway)                           │
│                                                                 │
│  Client ──► [API Gateway: Rate Limit] ──► Services              │
│                    │                                            │
│                    ├── Per API Key                              │
│                    ├── Per User ID                              │
│                    ├── Per IP (anonymous)                       │
│                    └── Per Endpoint (tiered)                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7.7 Idempotency

```
┌─────────────────────────────────────────────────────────────────┐
│  IDEMPOTENCY: SAME REQUEST × N = SAME RESULT AS × 1             │
└─────────────────────────────────────────────────────────────────┘

Client generates: idempotency_key = UUID per user action
                   (stored in localStorage, retried on failure)

Server:
  1. Check Redis: EXISTS idempotency_key
  2. If YES → return cached response
  3. If NO → execute, store (key → response) with TTL 24h
  4. Return response

POST /payments
Idempotency-Key: abc-123
{ "amount": 100, "currency": "USD" }

→ Response cached for key "abc-123"
→ Retry with same key → same response, no double-charge
```

### Idempotency Key Design

| Scenario | Key Generation |
|----------|----------------|
| **User-initiated** | `user_id:action:timestamp` (client generates) |
| **Webhook callback** | Provider's event ID (Stripe: `evt_xxx`) |
| **Internal retry** | Correlation ID passed through chain |

---

## 7.8 Graceful Degradation

```
┌─────────────────────────────────────────────────────────────────┐
│  DEGRADE GRACEFULLY: Core works, non-core fails softly          │
└─────────────────────────────────────────────────────────────────┘

FEATURE TIERING:
  TIER 1 (Critical):  Payments, Auth, Core API     → Never degrade
  TIER 2 (Important): Recommendations, Search      → Fallback: cached/popular
  TIER 3 (Nice-to-have): Analytics, A/B tests, ML  → Disable first

FALLBACK STRATEGIES:
  • Cache stale data (serve expired cache)
  • Default values (empty list, generic response)
  • Reduced functionality (no personalization)
  • Static fallback (maintenance page)
  • Queue for later (async processing)
```

---

## 7.9 Chaos Engineering

```
┌─────────────────────────────────────────────────────────────────┐
│  CHAOS ENGINEERING: EXPERIMENT ON PRODUCTION                    │
└─────────────────────────────────────────────────────────────────┘

PRINCIPLES:
  1. Define steady state (metrics: latency, error rate, throughput)
  2. Hypothesize: "If X fails, system behaves Y"
  3. Inject failure: Kill pod, add latency, drop packets, revoke cert
  4. Observe: Does system recover? Alert fire? Degradation acceptable?
  5. Fix: Improve resilience based on findings

TOOLS:
  • Chaos Mesh (Kubernetes)
  • Gremlin (SaaS)
  • LitmusChaos
  • AWS Fault Injection Simulator

EXPERIMENTS:
  • Kill random pod → verify auto-recovery
  • Add 500ms latency to DB → verify circuit breaker
  • Lose AZ → verify multi-AZ failover
  • Expire certs → verify rotation
  • Fill disk → verify alerts + cleanup
```

---

## 7.10 Reliability Checklist

```
┌─────────────────────────────────────────────────────────────────┐
│  PRODUCTION READINESS CHECKLIST                                 │
├─────────────────────────────────────────────────────────────────┤
│  TIMEOUTS                                                       │
│  □ Every external call has timeout                              │
│  □ Timeout hierarchy: client > gateway > service > DB           │
│  □ Connection timeout separate from request timeout             │
│                                                                 │
│  RETRIES                                                        │
│  □ Exponential backoff + jitter                                 │
│  □ Max attempts (3–5)                                           │
│  □ Retry only on transient errors (5xx, timeout, network)       │
│  □ Idempotency keys for non-idempotent operations               │
│                                                                 │
│  CIRCUIT BREAKER                                                │
│  □ Per-dependency circuit breaker                               │
│  □ Failure threshold, success threshold, timeout configured     │
│  □ Fallback responses defined                                   │
│                                                                 │
│  BULKHEADS                                                      │
│  □ Thread pools isolated per downstream                         │
│  □ Connection pools isolated per downstream                     │
│  □ Critical paths on dedicated infrastructure                   │
│                                                                 │
│  RATE LIMITING                                                  │
│  □ Edge rate limiting (API Gateway)                             │
│  □ Per-tenant / per-user quotas                                 │
│  □ Graceful 429 with Retry-After header                         │
│                                                                 │
│  OBSERVABILITY                                                  │
│  □ RED metrics (Rate, Errors, Duration) per service             │
│  □ Distributed tracing (100% sampling for errors)               │
│  □ Structured logs (JSON, correlation IDs)                      │
│  □ Alerts on SLO burn rate                                      │
│                                                                 │
│  FAILURE TESTING                                                │
│  □ Chaos experiments scheduled                                  │
│  □ Game days quarterly                                          │
│  □ Runbook for each alert                                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## ✅ Exercises

1. **Timeout Hierarchy**: Design timeout values for: Mobile App → API Gateway → Auth Service → User DB → Redis Cache.
2. **Circuit Breaker**: Payment service calls Stripe (external). Stripe has 2% error rate, 200ms p99. Configure circuit breaker.
3. **Bulkhead**: You have 3 downstream services with different criticality. Design thread pool allocation for 100 total threads.
4. **Idempotency**: Design idempotency for "send email" (not naturally idempotent).

---

## 🔁 Review Prompts

- Why is timeout hierarchy important? What happens if service timeout > client timeout?
- What's the difference between retry and circuit breaker? When does each trigger?
- How does bulkhead prevent cascade failures?
- What makes an operation idempotent? How do you make a non-idempotent operation idempotent?

---

## 📚 Further Reading

- *Release It!* — Michael Nygard (stability patterns)
- *Site Reliability Engineering* — Google (Ch on Reliability)
- [Resilience4j Documentation](https://resilience4j.readme.io/)
- [Hystrix (archived) / Resilience4j Patterns](https://github.com/Netflix/Hystrix/wiki)