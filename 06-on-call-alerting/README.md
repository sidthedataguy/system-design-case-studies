
# On-Call Alerting System

## 1. Problem

Design a system that ingests alerts from monitoring tools, deduplicates them into incidents, notifies the correct on-call engineer, and automatically escalates unacknowledged incidents according to priority-based SLAs.

Scale: ~1,000–10,000 alerts/day.

The primary design goal is **reliability over efficiency** — never silently lose a real escalation.

---

## 2. Requirements

### Functional

- Ingest alerts through API, email, and manual trigger
- Authenticate external alert sources
- Deduplicate repeated signals into one incident
- Notify on-call engineer through SMS + email
- Escalate to voice if unacknowledged
- Support priority-based escalation SLAs
- Capture acknowledgments
- Stop further notifications after acknowledgment
- Rate-limit new incident creation per monitoring source
- Maintain a complete audit trail

### Non-Functional

- Resist spoofing and replay
- Resolve the current on-call roster at send-time
- Prevent duplicate escalation processing
- Favor over-notification over silently missing an escalation

---

## 3. Key Design Decisions

| Decision | Choice | Why |
|---|---|---|
| Ingestion | Common schema → queue | Normalizes multiple input channels |
| Authentication | HMAC / SPF-DKIM / timestamp validation | Prevents spoofing and replay |
| Deduplication | DB unique constraint + `ON CONFLICT` | Durable concurrency-safe enforcement |
| Escalation engine | Stateless pollers | Simple and horizontally scalable |
| Multi-instance safety | `FOR UPDATE SKIP LOCKED` | Prevents duplicate claims |
| Notification dispatch | Separate queue + workers | Third-party latency doesn't block escalation |
| Roster lookup | Resolve at send-time | Handles on-call rotation |
| Acknowledgment | Conditional `UPDATE ... WHERE status='OPEN'` | Race-safe |
| Ack/escalation race | Favor escalation | Missing a real alert is worse than a redundant page |
| Rate limiting | New incidents only | Doesn't suppress legitimate escalations |
| Audit | Every state transition | Provides traceability |

---

## 4. Architecture

```mermaid
flowchart TD
    API[Monitoring API]
    Email[Alert Email]
    Manual[Manual Trigger]

    Ingest[Ingestion Service]
    Queue[Ingestion Queue]
    DB[(Incident DB)]
    
    Poller[Escalation Pollers]
    NotifyQ[Notification Queue]
    Workers[Notification Workers]

    SMS[SMS Provider]
    Mail[Email Provider]
    Voice[Voice Provider]

    Roster[On-Call Roster]
    Audit[(Audit Log)]

    API --> Ingest
    Email --> Ingest
    Manual --> Ingest

    Ingest --> Queue
    Queue --> DB

    DB --> Poller
    Poller -->|Resolve current roster| Roster
    Poller --> NotifyQ

    NotifyQ --> Workers
    Workers --> SMS
    Workers --> Mail
    Workers --> Voice

    Workers --> DB
    DB --> Audit
````

---

## 5. Critical Flow — Alert to Escalation

```text
Alert
  ↓
Authenticate
  ↓
Normalize
  ↓
Queue
  ↓
Atomic deduplication
  ↓
Create / update incident
  ↓
Escalation poller
  ↓
Claim incident using SKIP LOCKED
  ↓
Resolve current on-call roster
  ↓
Notification queue
  ↓
SMS + Email
  ↓
Escalate if unacknowledged
```

---

## 6. Deduplication

Repeated alerts for the same underlying problem are collapsed using:

```text
alert_key = hash(alert_rule + host/resource)
```

The database enforces uniqueness for open incidents.

```sql
INSERT ... 
ON CONFLICT (alert_key)
WHERE status = 'OPEN'
DO UPDATE SET count = count + 1;
```

The database is the source of truth for deduplication.

---

## 7. Escalation

Pollers periodically claim eligible incidents:

```sql
SELECT *
FROM incidents
WHERE status = 'OPEN'
  AND next_action_at <= NOW()
FOR UPDATE SKIP LOCKED
LIMIT 100;
```

This allows multiple poller instances without processing the same incident simultaneously.

---

## 8. Acknowledgment Race

Acknowledgment and escalation can happen almost simultaneously.

The design deliberately favors **over-notification**.

```text
Incident OPEN
      │
      ├── ACK
      │
      └── Escalation fires
```

If both race, the escalation may still be delivered.

Reason:

> One redundant notification is preferable to silently losing a real escalation.

---

## 9. Rate Limiting

Rate limiting applies only to **new incident creation**.

```text
800 alerts/hour
      ↓
Heads-up notification

1000 alerts/hour
      ↓
Reject new alert keys
```

Existing open incidents can still retry and escalate.

This prevents a malfunctioning monitoring source from being suppressed without blocking legitimate incident handling.

---

## 10. Mistakes & Corrections

1. Queue-native deduplication → DB constraint as durable source of truth
2. Check-then-insert → atomic `ON CONFLICT`
3. "Escalation service checks and acts" → explicit polling + locking + state fields
4. Suppressed escalation during ack race → favor over-notification
5. Cached roster at incident creation → resolve at send-time
6. Blanket source rate limit → limit only new incident creation
7. Email sender string matching → SPF/DKIM verification
8. No replay protection → timestamp validation

---

## 11. Core Takeaways

* Deduplication is not the same as suppression.
* Check-then-insert is a concurrency race.
* A distributed worker needs an explicit claiming mechanism.
* Dynamic state should be resolved when the action occurs.
* Security verification must be cryptographically grounded.
* When two correctness guarantees race, explicitly decide which failure is safer.

---

## 12. Patterns Learned

* Atomic deduplication
* `ON CONFLICT`
* `FOR UPDATE SKIP LOCKED`
* Queue-backed notification dispatch
* Dynamic state resolution
* Priority-based escalation
* Race-aware design
* Source-level rate limiting
* Cryptographic request verification
* Full audit trails


```
