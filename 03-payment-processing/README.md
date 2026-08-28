
# Payment Processing System

## 1. Problem

Design a payment processing backend for an e-commerce platform integrating with a third-party payment gateway.

The system must reliably track payment state and prevent double-charging or lost transactions, even during timeouts, network failures, or gateway outages.

---

## 2. Requirements

### Functional

- Accept tokenized card details
- Initiate payments with token, amount and order ID
- Handle synchronous gateway responses
- Handle asynchronous webhook confirmations
- Prevent duplicate charges during retries
- Reconcile payments when webhooks fail or responses are ambiguous

### Non-Functional

- ~2,000 transactions/min average
- ~15,000 transactions/min peak
- No double-charging
- No silently lost payments
- Gateway delays must not block application resources
- Resilient to partial/full gateway outages

---

## 3. Key Design Decisions

| Decision | Choice | Why |
|---|---|---|
| Card data | Tokenized data only | Keeps PCI-DSS scope with the gateway |
| Payment initiation | Asynchronous | Avoids holding connections during slow gateway flows |
| Response handling | Sync + webhook | Supports both fast and delayed gateway responses |
| DB ordering | Persist `PENDING` payment before gateway call | Prevents an unrecorded payment attempt |
| Idempotency | `order_id` + UUID reference | Prevents duplicate concurrent requests |
| Duplicate protection | DB unique constraint | Application-level checks are racy |
| Timeout handling | Gateway status check before retry | Timeout does not mean payment failed |
| Reconciliation | Scheduled status-check job | Recovers payments when webhooks are missing |
| Reconciliation failures | Exponential backoff + circuit breaker | Prevents hammering an unavailable gateway |
| Escalation | Manual review only after automated recovery | Humans are the last resort |
| Webhook duplicates | Unique `event_id` | Makes processing idempotent |
| Webhook security | HMAC signature verification | Prevents forged webhook requests |

---

## 4. Architecture

```mermaid
flowchart TD
    Client[Client / E-commerce App]
    API[Payment API]
    DB[(Payment DB<br/>System of Record)]

    Gateway[Payment Gateway]

    Webhook[Webhook Endpoint]
    Queue[Webhook Queue]
    Reconcile[Reconciliation Job]
    Status[Gateway Status API]

    Notify[Notification Service]
    Audit[Audit / Monitoring]

    Client --> API
    API --> DB
    API --> Gateway

    Gateway -->|Sync Response| API
    Gateway -->|Webhook| Webhook

    Webhook -->|Verify HMAC| Queue
    Queue --> DB

    Reconcile -->|Find stale PENDING payments| DB
    Reconcile --> Status
    Status --> Gateway
    Gateway --> Status
    Status --> Reconcile
    Reconcile --> DB

    DB --> Notify
    DB --> Audit
````

---

## 5. Critical Flow — Payment Initiation

```text
Client
  ↓
Create payment record (PENDING)
  ↓
Call payment gateway
  ↓
┌──────────────────┬────────────────────┐
│ Fast response    │ Delayed response   │
│                  │                    │
│ Update payment   │ Wait for webhook   │
│ state            │                    │
└──────────────────┴────────────────────┘
```

The payment record is persisted **before** calling the gateway.

---

## 6. Critical Flow — Ambiguous Timeout

A timeout is **not treated as failure**.

```text
Gateway call
     ↓
Timeout
     ↓
Query gateway status
     ↓
┌─────────────┬─────────────┐
│ Successful  │ Failed      │
│             │             │
│ Mark paid   │ Safe retry  │
└─────────────┴─────────────┘
```

Blindly retrying after a timeout could charge the customer twice.

---

## 7. Webhook & Reconciliation

Webhook is the **fast path**.

Reconciliation is the **guaranteed recovery path**.

```text
Gateway
   ↓
Webhook
   ↓
Process payment

       +

Scheduled Reconciliation
   ↓
Find stale PENDING payments
   ↓
Gateway Status API
   ↓
Correct payment state
```

Webhook events are deduplicated using a unique `event_id`.

Webhook authenticity is verified using HMAC.

---

## 8. Mistakes & Corrections

1. Held connections for up to 3 minutes → asynchronous processing
2. Assumed direct bank communication → gateway is the external dependency
3. Treated DB as audit-only → DB is the payment system of record
4. Treated timeout as failure → status check before retry
5. No webhook recovery → reconciliation job
6. Duplicate detection based on payment status → unique webhook `event_id`
7. No webhook authentication → HMAC verification
8. Immediate manual escalation → backoff + circuit breaker first

---

## 9. Core Takeaways

* **Timeout ≠ failure**, especially when money is involved.
* Persist state before calling external systems.
* Webhooks provide fast asynchronous responses; reconciliation provides recovery.
* Idempotency and atomic database constraints are recurring distributed-system patterns.
* Authentication requires cryptographic verification, not just matching identifiers.
* Automated recovery should be exhausted before human intervention.

---

## 10. Patterns Learned

* Idempotency
* Persist-before-external-call
* Async request/response
* Webhook + reconciliation
* Timeout ambiguity handling
* Exponential backoff
* Circuit breaker
* Cryptographic webhook verification
