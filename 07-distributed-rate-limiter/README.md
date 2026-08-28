
# Distributed Rate Limiter

## 1. Problem

Design a shared, synchronous rate-limiting service used by multiple internal APIs.

The service enforces per-API-key limits based on subscription tier while also validating authorization.

Target: ~100k checks/sec with sub-5ms decision latency.

---

## 2. Requirements

### Functional

- Free tier: 10 requests/minute
- Paid tier: 1,000 requests/minute
- Validate API key as part of the same check
- Return `429` + `Retry-After` when rejected
- Log allowed requests asynchronously

### Non-Functional

- 100,000 active API keys
- ~110,000 checks/sec peak
- Single-digit millisecond decision latency
- Minor fixed-window boundary overshoot is acceptable
- Fail closed only for keys whose shard is unavailable

---

## 3. Key Design Decisions

| Decision | Choice | Why |
|---|---|---|
| Algorithm | Fixed/tumbling window | Simple and cost-effective |
| State store | Redis Cluster | High-throughput, low-latency counter operations |
| Sharding | `api_key` hash | Keeps each key's state on one shard |
| Atomicity | Redis Lua script | Auth + tier + counter become one operation |
| Redis failure | Replica promotion | Automatic infrastructure-level failover |
| Total shard failure | Fail closed per key | Preserves rate-limit correctness |
| Audit | Async queue | Keeps audit writes out of hot path |
| Tier changes | Active Redis update | Avoids stale authorization/rate-limit state |

---

## 4. Architecture

```mermaid
flowchart TD
    Client[API Client]
    Gateway[API Gateway / API Service]
    RateLimiter[Rate Limiter Service]

    Redis[(Redis Cluster<br/>Sharded by API Key)]
    Billing[Billing / Account Service]

    Queue[Audit Queue]
    Audit[(Audit / Analytics DB)]

    Client --> Gateway
    Gateway --> RateLimiter

    RateLimiter -->|Single atomic Lua call| Redis

    Billing -->|Tier update| Redis

    RateLimiter -->|Allowed request event| Queue
    Queue --> Audit
````

---

## 5. Critical Flow

```text
API Request
    ↓
Rate Limiter
    ↓
Redis Lua Script
    ├── Validate API key
    ├── Read subscription tier
    ├── Check/reset window
    ├── Increment counter
    └── Return ALLOW / REJECT
```

The entire decision is completed in **one Redis round trip**.

---

## 6. Redis State

Each API key has one Redis hash:

```text
apikey:{api_key}

auth_hash
tier
current_count
window_start
TTL = 60 seconds
```

Keys are distributed across the Redis Cluster using the API key hash.

---

## 7. Failure Handling

### Redis shard failure

```text
Primary shard fails
       ↓
Redis replica promoted
       ↓
Application retries
       ↓
Continue serving
```

If the shard **and all replicas** are unreachable:

```text
Affected API keys → 503 / Retry-After
Other API keys   → Continue normally
```

The system does **not** reroute the key to another shard because that could reset its counter and violate the rate limit.

---

## 8. Async Audit Path

Allowed requests are sent asynchronously:

```text
Rate Limit Decision
       ↓
   Audit Queue
       ↓
Analytics / Audit DB
```

Audit processing can lag without affecting the synchronous decision.

---

## 9. Mistakes & Corrections

1. Separate auth + tier + rate-check calls → one atomic Lua operation
2. Application-level shard rerouting → replica promotion
3. Global fail-closed → per-key fail-closed
4. Synchronous audit DB write → asynchronous queue
5. Passive tier-cache expiry → active tier updates
6. Treating every service interaction as asynchronous → keep latency-critical decision path synchronous

---

## 10. Core Takeaways

* Not every cross-service interaction should use a queue.
* Atomicity can improve both correctness **and** latency.
* Sharding strategy can become part of the correctness model.
* Global consistency has a latency and complexity cost.
* Capacity assumptions should be quantified before choosing infrastructure.
* Application failure handling and infrastructure observability are separate concerns.

---

## 11. Patterns Learned

* Fixed-window rate limiting
* Redis Lua atomic operations
* Key-based sharding
* Per-key failure isolation
* Active cache/state invalidation
* Async audit pipelines
* Capacity-driven architecture

