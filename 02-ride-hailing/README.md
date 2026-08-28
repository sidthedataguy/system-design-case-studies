
# Ride-Hailing Matching System

## 1. Problem

Design a real-time ride-hailing backend that matches riders with nearby available drivers.

The system must provide low-latency matching while guaranteeing that a ride request is assigned to at most one driver.

---

## 2. Requirements

### Functional

- Rider requests a ride with location
- Find nearby available drivers
- Offer the ride to drivers
- Driver accepts or rejects
- Handle driver timeout/non-response
- Provide live driver location and ETA
- Support ride cancellation
- Handle cancellation racing with driver acceptance

### Non-Functional

- Matching must happen within seconds
- Strong consistency — no double-booking
- Driver locations update continuously
- System remains available during component failures

---

## 3. Key Design Decisions

| Decision | Choice | Why |
|---|---|---|
| Matching strategy | Broadcast to top-N nearest drivers | Sequential offering creates unacceptable latency |
| Double-booking prevention | Atomic DB claim | Application-level check-then-write is racy |
| Losing offers | Explicit "offer revoked" notification | Prevents wasted driver capacity |
| Offer lock | ~15 second soft-lock | Prevents drivers being unavailable unnecessarily |
| Location storage | Redis Geo | Optimized for high-frequency proximity queries |
| Availability | Separate Redis Set | Location and availability have different lifecycles |
| Matching query | Geo results ∩ available drivers | Excludes busy drivers while keeping them trackable |
| Real-time delivery | Push / WebSocket | Polling cannot meet sub-second delivery needs |
| Cancellation race | Atomic conditional updates | Guarantees exactly one outcome |
| Push failure | Retry queue + reconnect fallback | Push delivery is best-effort |
| Redis failure | Secondary instance + automated replacement | Graceful degradation |
| Primary DB failure | Queue requests + failover | Maintains write availability |

---

## 4. Architecture

```mermaid
flowchart TD
    Rider[Rider App]
    Gateway[API Gateway]
    LB[Load Balancer]
    App[Matching Service]

    Geo[(Redis Geo<br/>Driver Locations)]
    Available[(Redis Set<br/>Available Drivers)]

    DB[(Primary DB<br/>Ride State)]
    Replica[(Read Replica)]

    Push[Notification / WebSocket Gateway]
    Queue[Retry Queue]
    Driver1[Driver]
    Driver2[Driver]
    Driver3[Driver]

    Rider --> Gateway
    Gateway --> LB
    LB --> App

    App -->|GEO radius query| Geo
    App -->|Intersect| Available

    App -->|Atomic ride claim| DB
    DB -->|Async replication| Replica

    App -->|Offer ride| Push
    Push --> Driver1
    Push --> Driver2
    Push --> Driver3

    Driver1 -->|Accept / Reject| App
    Driver2 -->|Accept / Reject| App
    Driver3 -->|Accept / Reject| App

    App -->|Location updates| Geo

    App -->|Failed notification| Queue
    Queue --> Push
````

---

## 5. Critical Flow — Matching

```text
Rider requests ride
        ↓
Find nearby drivers using Redis Geo
        ↓
Intersect with available-driver set
        ↓
Select top-N drivers
        ↓
Broadcast offer
        ↓
First valid atomic DB claim wins
        ↓
Revoke remaining offers
        ↓
Notify rider + winning driver
```

---

## 6. Critical Race — Acceptance vs Cancellation

Both operations attempt an atomic conditional update on the same ride record.

```text
              ┌── Driver accepts ──┐
Ride Pending ─┤                    ├─→ Exactly one wins
              └── Rider cancels ──┘
```

The losing operation triggers an explicit compensating flow.

---

## 7. Mistakes & Corrections

1. Sequential driver offering → parallel top-N broadcast
2. No atomic claim → DB-level atomic claim
3. No notification to losing drivers → explicit offer revocation
4. Location + availability conflated → separate data structures
5. SQL used for proximity search → Redis Geo
6. Acceptance/cancellation race unresolved → competing atomic updates
7. Push assumed reliable → retry + reconnect fallback

---

## 8. Core Takeaways

* Broadcasting is safe when the final claim is atomic.
* Data with different lifecycles should be separated.
* Every important state change needs an explicit signal.
* Timeouts should not be used to infer state when an explicit event is possible.

---

## 9. Patterns Learned

* Atomic conditional updates
* Top-N parallel matching
* Geospatial indexing
* Separation of location and availability
* Soft locks
* Compensating flows
* Push + pull fallback
* Graceful degradation

