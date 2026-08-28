
# Real-Time Chat / Messaging System

## 1. Problem

Design a real-time messaging backend supporting 1-to-1 and group conversations, offline users, multi-device presence, message ordering, and delivery/read receipts.

---

## 2. Requirements

### Functional

- 1-to-1 and group messaging
- Real-time delivery to online users
- Reliable delivery to offline users after reconnect
- Ordering per conversation
- Delivery states: sent → delivered → read
- Multi-device support
- End-to-end encrypted messages

### Non-Functional

- Millions of concurrent users
- Near-instant delivery
- No message loss or duplicates
- Scalable group fan-out

---

## 3. Key Design Decisions

| Decision | Choice | Why |
|---|---|---|
| Presence | WebSocket + Redis | Real-time presence without polling |
| Presence liveness | Heartbeat every ~20s + 60s TTL | Detects ungraceful disconnects |
| Message durability | Persist as `SENT` before delivery | Prevents message loss |
| Online delivery | Targeted push to WebSocket server | Avoids generic broadcast |
| Offline delivery | Fetch on WebSocket reconnect | Event-driven recovery |
| Ordering | Queue partitioned by `conversation_id` | Guarantees per-conversation ordering |
| Idempotency | Client-generated UUID + DB unique constraint | Retries reuse the same ID |
| Consumer recovery | Queue ack/redelivery | Failed messages are automatically retried |
| Multi-device | Redis set of sessions per user | Supports multiple active devices |
| Status sync | Push status changes to all sessions | Keeps devices consistent |
| Dead server detection | Remove Redis entry on push failure | Faster failure detection |
| Group fan-out | Batched Redis lookups + bulk inserts | Avoids sequential per-member processing |
| Fault isolation | Chunked bulk inserts | Limits blast radius |
| Analytics | Separate OLAP store | Keeps analytics away from OLTP workload |

---

## 4. Architecture

```mermaid
flowchart TD
    Client[Client Apps]
    WS[WebSocket Gateway]
    Chat[Chat Service]

    Redis[(Redis<br/>Presence + Sessions)]
    Queue[Message Queue<br/>Partition: conversation_id]

    DB[(Operational DB<br/>Messages + Delivery Status)]

    Fanout[Fan-out Consumer]
    Analytics[Analytics Pipeline]
    OLAP[(Analytics Store)]

    Client <-->|WebSocket| WS
    WS --> Chat

    Chat -->|Presence / Sessions| Redis
    Chat -->|Persist message| DB
    Chat -->|Publish message| Queue

    Queue --> Fanout
    Fanout -->|Lookup active sessions| Redis
    Fanout -->|Push to online devices| WS
    Fanout -->|Delivery status| DB

    Chat -->|Message events| Analytics
    Analytics --> OLAP
````

---

## 5. Critical Flow — Send Message

```text
Sender
  ↓
WebSocket Gateway
  ↓
Chat Service
  ↓
Persist message as SENT
  ↓
Message Queue
  ↓
Consumer
  ↓
Lookup recipient sessions
  ↓
Push to active devices
  ↓
Update delivery status
```

The message is persisted **before** delivery is attempted.

---

## 6. Offline Delivery

No continuous polling is required.

```text
User offline
     ↓
Message remains SENT in DB
     ↓
User reconnects
     ↓
Fetch undelivered messages
     ↓
Deliver
```

---

## 7. Ordering & Idempotency

### Ordering

Messages are partitioned by:

```text
conversation_id
```

All messages for a conversation are processed sequentially by the same queue partition.

### Idempotency

The client generates the message UUID.

Retries therefore reuse the same ID, while a DB unique constraint prevents duplicates.

---

## 8. Group Fan-Out

Instead of processing members one-by-one:

```text
Fetch members
    ↓
Bulk DB insert delivery records
    ↓
Batch Redis lookup
    ↓
Push only to online members
```

Large operations are chunked to limit failure blast radius.

---

## 9. Mistakes & Corrections

1. Polling for offline delivery → reconnect-triggered fetch
2. Single Redis presence entry → multiple sessions per user
3. First device receiving message marked delivered → push to all active sessions
4. Silent DB status update → push status changes to other devices
5. FIFO queue assumed sufficient → partition by `conversation_id`
6. Server-generated message ID → client-generated UUID
7. Polling for crashed consumers → queue ack/redelivery
8. Passive TTL for dead servers → immediate Redis cleanup on push failure
9. Sequential group fan-out → batching
10. All-or-nothing fan-out transaction → chunked inserts
11. Analytics store as operational DB → separate OLTP and OLAP

---

## 10. Core Takeaways

* Real-time systems generally benefit from push-based architecture rather than polling.
* Ordering requires partitioning around a logical key.
* Client-generated IDs are important when retries must be recognized.
* Multi-device systems must model users as multiple sessions.
* Chunking is a useful fault-isolation technique.
* Datastores should match the access pattern: OLTP for operational state, OLAP for analytics.

---

## 11. Patterns Learned

* Heartbeat-based presence
* Persist-before-deliver
* Partitioning for ordered processing
* Client-generated identifiers
* Event-driven recovery
* Multi-device state synchronization
* Batched fan-out
* Fault isolation through chunking

