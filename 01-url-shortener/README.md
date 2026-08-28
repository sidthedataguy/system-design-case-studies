
# URL Shortener

## 1. Problem

Design a URL shortening service for a mid-sized B2B company.

Internal teams and external partners can shorten URLs through an API and track usage through analytics.

---

## 2. Requirements

### Functional

- Shorten long URLs
- Support custom aliases
- Redirect short URLs to original URLs
- Support time-based and click-based expiration
- Provide per-client analytics:
  - Click count
  - Geography
  - Referrer
- API-key based access
- No anonymous usage

### Non-Functional

- ~50M new URLs/month
- Read/write ratio ≈ 100:1
- Traffic can peak at 5× average during campaigns
- Per-client rate limiting
- Low-latency redirects

---

## 3. Scale Characteristics

The system is **heavily read-oriented**.

URL creation is relatively infrequent, while redirects can be significantly higher.

Therefore:

- Optimize the redirect path for latency.
- Keep writes strongly consistent.
- Push non-critical work out of the request path.

---

## 4. Architecture

```mermaid
flowchart TD
    Client[Client / Partner API]

    Gateway[API Gateway<br/>API Key + Rate Limit]
    LB[Load Balancer]
    App[Stateless App Servers]

    Primary[(Primary DB<br/>URL + Alias)]
    Cache[(Redis Cache<br/>Short Code → URL)]
    Replicas[(Read Replicas)]

    Queue[Message Queue<br/>Kafka / SQS]
    Analytics[Analytics Pipeline]
    Warehouse[(Data Warehouse)]
    BI[Power BI / AI-ML]

    Client --> Gateway
    Gateway --> LB
    LB --> App

    App -->|Create URL / Alias| Primary
    Primary -->|Async Replication| Replicas

    App -->|Read| Cache
    Cache -->|Cache Miss| Replicas
    Replicas -->|Response| App

    App -->|Async Analytics Event| Queue
    Queue --> Analytics
    Analytics --> Warehouse
    Warehouse --> BI
````

---

## 5. Key Design Decisions

| Decision                | Choice                                    | Reason                                            |
| ----------------------- | ----------------------------------------- | ------------------------------------------------- |
| Short-code generation   | Snowflake-style distributed ID + Base62   | Avoids centralized counter bottleneck             |
| Custom alias uniqueness | DB unique constraint + atomic INSERT      | Prevents race conditions                          |
| Authentication          | Stateless API-key validation              | Suitable for B2B machine-to-machine access        |
| Redirect path           | Redis cache-aside + read replica fallback | Optimized for 100:1 read-heavy workload           |
| URL creation            | Primary DB                                | Correctness cannot depend on replica state        |
| Replication             | Primary → read replicas, async            | Separates write correctness from read scalability |
| Analytics               | Async message queue                       | Keeps analytics off the critical redirect path    |
| Rate limiting           | API Gateway                               | Reject excessive traffic early                    |

---

## 6. Critical Flows

### Create Short URL

```text
Client
  ↓
API Gateway
  ↓
Load Balancer
  ↓
Application
  ↓
Generate ID / Validate Alias
  ↓
Primary DB
  ↓
Return Short URL
```

Custom aliases rely on a database uniqueness constraint rather than:

```text
Check alias → If available → Insert
```

because that approach is vulnerable to concurrent requests.

### Redirect

```text
Client
  ↓
API Gateway
  ↓
Application
  ↓
Redis
  │
  ├── Hit → Return original URL
  │
  └── Miss → Read Replica → Populate Cache
```

This is the critical low-latency path.

### Analytics

```text
Redirect
   ↓
Message Queue
   ↓
Analytics Pipeline
   ↓
Data Warehouse
   ↓
Reporting / AI-ML
```

Analytics processing does not block the redirect response.

---

## 7. Consistency & Failure Considerations

* Writes always go to the primary database.
* Read replicas are used for scalable reads.
* Redis absorbs the majority of redirect traffic.
* Database constraints enforce correctness for aliases.
* Analytics failure should not prevent URL redirection.

---

## 8. Core Takeaways

* Statelessness does not mean no authentication.
* Read-heavy and write-heavy paths should be designed differently.
* Correctness-critical operations belong at the data layer.
* Non-critical side effects should be asynchronous.

---

## 9. Patterns Learned

* Cache + read replicas for read-heavy systems
* Database-enforced uniqueness
* Asynchronous side-effect processing
* Separate critical and non-critical request paths



I converted your original architecture diagram into **Mermaid**, while keeping the same major components and flows shown in your document. :contentReference[oaicite:1]{index=1}

**Next:** you can paste this into GitHub and preview it. Then we'll do a quick review of Day 1 before moving to Day 2.
```
