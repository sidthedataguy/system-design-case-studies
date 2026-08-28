# System Design — Quick Reference

A one-page summary of the key architectural challenges, decisions, trade-offs, and patterns from the 10 system design case studies.

---

## 1. Case Study Summary

| # | System | Primary Challenge | Key Architectural Decision | Main Pattern |
|---|---|---|---|---|
| 01 | URL Shortener | Extremely read-heavy workload | Redis + read replicas | Caching |
| 02 | Ride-Hailing | Real-time driver matching | Top-N broadcast + atomic claim | Concurrency |
| 03 | Payment Processing | Avoiding double charges | Idempotency + reconciliation | Reliable distributed transactions |
| 04 | Real-Time Chat | Ordering + multi-device delivery | Partition by conversation | Event-driven processing |
| 05 | File Storage | Large-file uploads + permissions | Chunking + resumability | Fault isolation |
| 06 | On-Call Alerting | Reliable escalation | Atomic claiming + watchdog | Distributed workers |
| 07 | Rate Limiter | 100K+ checks/sec | Redis + atomic Lua | Distributed state |
| 08 | Ticket / Inventory | High contention inventory | Atomic reservation | Concurrency control |
| 09 | Social Feed | Celebrity fan-out | Hybrid fan-out | Read/write trade-off |
| 10 | Job Scheduler | Duplicate execution + stuck jobs | Pollers + `SKIP LOCKED` | Distributed scheduling |

---

# 2. What Each System Taught Me

### 01 — URL Shortener

**Problem:** Read-heavy system.

**Decision:** Redis absorbs redirect reads while read replicas scale database reads.

**Key lesson:**

> Optimize the critical path according to the workload ratio.

**Patterns:** Cache-aside, read replicas, async analytics, DB uniqueness.

---

### 02 — Ride-Hailing

**Problem:** Match riders with nearby available drivers without double-booking.

**Decision:** Find top-N drivers using geospatial search, then use an atomic claim for the winning driver.

**Key lesson:**

> Parallelism improves latency; atomicity preserves correctness.

**Patterns:** Geo search, soft locks, atomic claims, WebSockets, push notifications.

---

### 03 — Payment Processing

**Problem:** A timeout does not tell us whether money was actually charged.

**Decision:** Persist payment state, use idempotency, webhooks and reconciliation.

**Key lesson:**

> In distributed systems, an ambiguous outcome must be resolved — not guessed.

**Patterns:** Idempotency, webhook processing, reconciliation, retry, circuit breaker.

---

### 04 — Real-Time Chat

**Problem:** Real-time delivery, ordering, presence and multi-device synchronization.

**Decision:** WebSockets for real-time communication and queue partitioning by `conversation_id`.

**Key lesson:**

> Ordering requirements should determine the partitioning strategy.

**Patterns:** WebSockets, heartbeats, partitioning, idempotency, fan-out.

---

### 05 — File Storage

**Problem:** Large files, resumable uploads, versioning and hierarchical permissions.

**Decision:** Chunk uploads, track upload sessions and cache permission decisions.

**Key lesson:**

> Large operations should be broken into independently retryable units.

**Patterns:** Chunking, resumability, hashing, permission inheritance, cache invalidation.

---

### 06 — On-Call Alerting

**Problem:** Alerts must not silently disappear or be incorrectly deduplicated.

**Decision:** Atomic DB deduplication and distributed worker claiming using `SKIP LOCKED`.

**Key lesson:**

> In reliability-critical systems, choose explicitly which failure is safer.

**Patterns:** Atomic deduplication, worker claiming, escalation, watchdogs, audit trails.

---

### 07 — Distributed Rate Limiter

**Problem:** Very high request volume with extremely low decision latency.

**Decision:** Shard by API key and perform authentication + tier lookup + counter update atomically in Redis.

**Key lesson:**

> The synchronous hot path should contain only what is necessary to make the decision.

**Patterns:** Sharding, Redis Lua, atomic operations, async auditing, per-key failure isolation.

---

### 08 — Ticket / Inventory Booking

**Problem:** Prevent inventory from becoming negative under heavy contention.

**Decision:** Use atomic inventory reservation; Redis for extreme contention and DB conditional updates for moderate contention.

**Key lesson:**

> Similar business problems can require different architectures when their contention profiles differ.

**Patterns:** Atomic decrement, timed holds, idempotency, expiry polling, compensation/refunds.

---

### 09 — Social Media Feed

**Problem:** Normal users and celebrities create very different fan-out costs.

**Decision:** Fan-out-on-write for normal accounts and fan-out-on-read for celebrities.

**Key lesson:**

> Architecture should adapt to asymmetric workload rather than forcing one strategy everywhere.

**Patterns:** Hybrid fan-out, batching, chunking, tombstones, bounded caches.

---

### 10 — Distributed Job Scheduler

**Problem:** Trigger scheduled work while handling overlap, stuck jobs and missed executions.

**Decision:** Separate trigger and watchdog pollers, with distributed claiming using `SKIP LOCKED`.

**Key lesson:**

> Different responsibilities with different semantics should not automatically share the same processing loop.

**Patterns:** Polling, watchdogs, distributed locks, async execution, human-controlled recovery.

---

# 3. Recurring Architecture Patterns

| Pattern | Where It Appears | Why |
|---|---|---|
| **Idempotency** | Payments, Chat, Booking, Alerting | Make retries safe |
| **Atomic Operations** | Ride-Hailing, Payments, Rate Limiter, Booking | Prevent race conditions |
| **`SKIP LOCKED`** | Alerting, Scheduler | Distributed work claiming |
| **Caching** | URL Shortener, Feed, Rate Limiter, File Storage | Reduce latency/load |
| **Async Queues** | Payments, Chat, Alerting, Feed | Decouple workloads |
| **Reconciliation** | Payments, File Storage | Recover from incomplete async operations |
| **Chunking** | File Storage, Feed, Group Chat | Limit failure blast radius |
| **Batching** | Feed, Group Chat | Reduce network/DB overhead |
| **Watchdogs** | Alerting, Scheduler | Detect stuck work |
| **Retry + Backoff** | Payments, Alerting, File Storage | Recover from transient failures |
| **Partitioning** | Chat, Rate Limiter | Scale while preserving locality |
| **Hybrid Architecture** | Inventory, Social Feed | Adapt to workload characteristics |

---

# 4. Distributed-System Lessons

### 1. Timeout ≠ Failure

An external operation can succeed even when the caller times out.

**Example:** Payment processing.

---

### 2. Check-Then-Act Is Dangerous

```text
Check
  ↓
Act
````

Two concurrent requests can both pass the check.

Prefer:

```text
Atomic Check + Act
```

---

### 3. Database Constraints Are Part of the Architecture

Uniqueness and correctness should not depend entirely on application code.

---

### 4. Async Processing Is Not Always Better

Use asynchronous processing when the caller does not need the result immediately.

Keep latency-critical decisions synchronous.

---

### 5. Partitioning Is a Business Decision

The partition key should usually come from the business requirement:

```text
conversation_id
api_key
```

rather than being chosen only for infrastructure convenience.

---

### 6. Failure Handling Is Part of the Design

A good architecture should explicitly answer:

```text
What happens if this component fails?
What happens if the network times out?
What happens if the message is duplicated?
What happens if two operations race?
```

---

### 7. Recovery Should Be Deliberate

Not every failure should be automatically repaired.

Sometimes the correct architecture is:

```text
Detect
  ↓
Stabilize
  ↓
Notify
  ↓
Human decision
```

---

# 5. My System Design Checklist

For a new system, I should ask:

```text
1. What is the business problem?

2. What are the functional requirements?

3. What are the scale assumptions?

4. What is the critical path?

5. Where is the likely bottleneck?

6. What needs strong consistency?

7. What can be asynchronous?

8. What happens when a request is retried?

9. What happens when two requests race?

10. What happens when a dependency times out?

11. What happens when a component fails?

12. How does the system recover?

13. Where does state live?

14. How will the system scale?

15. What trade-off am I consciously making?
```

---

# 6. The Bigger Lesson

Across these 10 systems, the recurring theme is:

> **System design is primarily about managing trade-offs under constraints.**

The technology is the implementation detail.

The architecture comes from:

```text
Requirements
      +
Scale
      +
Constraints
      +
Failure Modes
      +
Consistency Requirements
      ↓
Architectural Decisions
```

---

## Status

**10 system design case studies completed**

This document will evolve as new case studies and architectural patterns are added.

