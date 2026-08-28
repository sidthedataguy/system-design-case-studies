
# System Design Case Studies

A collection of system design case studies focused on scalable architectures, distributed systems, failure handling, trade-offs, and architectural decision-making.

## Why This Repository?

I am using these case studies to strengthen my system design and architecture skills.

The goal is not to memorize technologies or architecture diagrams.

The goal is to practice:

- Understanding business requirements
- Identifying constraints and bottlenecks
- Making architectural decisions
- Reasoning about scale and performance
- Designing for failure
- Handling concurrency and consistency
- Understanding trade-offs
- Recognizing reusable distributed-system patterns

Each case study represents one system-design exercise.

---

## Case Studies

| # | Case Study | Key Concepts |
|---|---|---|
| 01 | [URL Shortener](./01-url-shortener/) | Caching, read replicas, idempotency |
| 02 | [Ride-Hailing](./02-ride-hailing/) | Geospatial search, atomic claims, real-time systems |
| 03 | [Payment Processing](./03-payment-processing/) | Idempotency, reconciliation, webhooks |
| 04 | [Real-Time Chat](./04-realtime-chat/) | WebSockets, ordering, presence, fan-out |
| 05 | [File Storage](./05-file-storage/) | Chunking, resumable uploads, permissions |
| 06 | [On-Call Alerting](./06-on-call-alerting/) | Deduplication, escalation, distributed workers |
| 07 | [Distributed Rate Limiter](./07-distributed-rate-limiter/) | Redis, sharding, atomic operations |
| 08 | [Ticket / Inventory Booking](./08-ticket-booking/) | Concurrency, timed holds, inventory |
| 09 | [Social Media Feed](./09-social-media-feed/) | Fan-out, caching, celebrity problem |
| 10 | [Distributed Job Scheduler](./10-job-scheduler/) | Scheduling, polling, watchdogs, recovery |

---

## How I Approach System Design

My approach is:

```text
Business Problem
      ↓
Requirements
      ↓
Scale & Constraints
      ↓
Identify Bottlenecks
      ↓
Architecture
      ↓
Critical Flows
      ↓
Failure Scenarios
      ↓
Trade-offs
      ↓
Patterns & Lessons
````

I try to answer **"Why this design?"** before asking **"Which technology?"**

---

## Patterns Emerging Across the Cases

Some patterns have repeatedly appeared across different problems:

* Atomic database operations
* Idempotency
* `FOR UPDATE SKIP LOCKED`
* Asynchronous processing
* Caching
* Partitioning and sharding
* Chunking and batching
* Reconciliation
* Retry and backoff
* Cache invalidation
* Graceful degradation
* Explicit failure handling

An important part of this exercise is recognizing when the **same underlying problem shape appears in a completely different system**.

---

## What I Am Learning

The main lesson so far:

> **Good system design is less about choosing technologies and more about understanding constraints, failure modes, and trade-offs.**

The case studies will continue to evolve as my understanding of system design improves.

---

## Status

**10 case studies completed**

This repository is an ongoing learning and architecture practice journal.

