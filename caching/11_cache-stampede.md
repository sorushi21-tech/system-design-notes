# Cache Stampede (Thundering Herd) 

---

## 1. What Is Cache Stampede? (Pure Concept)

**Cache Stampede** occurs when **many requests simultaneously miss the cache for the same key** and all attempt to rebuild it at once.

Key condition:

> The data **exists**, but the cache is temporarily empty.

```
Requests → Cache (MISS for same key) → Backend overload
```

---

## 2. The Core Invariant That Breaks

### Normal Cache Invariant

> *Only a small, controlled number of requests should ever hit the database.*

Cache stampede **breaks this invariant** by allowing *unbounded concurrency* to reach the backend.

```mermaid
flowchart LR
    R[Many Requests]
    C[Cache]
    D[Database]

    R -->|HIT| C
    C -->|Serve| R
    D -.->|Idle| D
```

---

### Stampede Invariant Violation

```mermaid
flowchart LR
    R[Many Requests]
    C[Cache]
    D[Database]

    R -->|MISS| C -->|MISS| D
    R -->|MISS| C -->|MISS| D
    R -->|MISS| C -->|MISS| D
```

The cache stops acting as a *gatekeeper*.

---

## 3. Why Cache Stampede Is Dangerous

### Amplification Effect

One expired key can create:

```
1 cache miss
→ 100s or 1000s of backend calls
```

This is **load amplification**, not just high traffic.

---

### Cascading Failure Chain

```mermaid
flowchart TD
    Expiry[Cache Expiry]
    Surge[Request Surge]
    DB[Database Saturation]
    Timeouts[Timeouts]
    Retries[Client Retries]
    Collapse[System Collapse]

    Expiry --> Surge --> DB --> Timeouts --> Retries --> Collapse
```

Stampedes rarely fail alone — they trigger **system‑wide collapse**.

---

## 4. When Cache Stampede Happens

### 1️⃣ Time‑Based Expiration

All requests hit immediately after TTL expiry.

```
Time T: Cache valid
Time T+1ms: Cache expired
Time T+2ms: Thousands of requests arrive
```

---

### 2️⃣ Event‑Based Invalidation

Mass invalidation creates an *instant cold cache*.

```mermaid
flowchart LR
    Update[Data Update]
    Invalidate[Cache Cleared]
    Requests[Concurrent Requests]

    Update --> Invalidate --> Requests
```

---

### 3️⃣ Hot Key Access

A **single highly popular key** dominates traffic.

> The hotter the key, the more destructive its expiry.

---

## 5. Cache Stampede vs Other Cache Failures

| Problem     | Does Data Exist? | Failure Cause       |
| ----------- | ---------------- | ------------------- |
| Penetration | ❌ No             | Non‑existence       |
| Stampede    | ✅ Yes            | Concurrent rebuild  |
| Avalanche   | ✅ Yes (many)     | Synchronized expiry |
| Hot Key     | ✅ Yes            | Skewed access       |

Correct identification determines the correct fix.

---

## 6. Core Insight: Stampede Is a *Concurrency* Problem

> Cache stampede is not about caching — it is about **coordination**.

Multiple actors try to rebuild the *same thing* at the *same time*.

---

## 7. Conceptual Defense Strategies

### Strategy Landscape (No Implementation)

| Strategy      | What It Controls | Core Idea                     |
| ------------- | ---------------- | ----------------------------- |
| TTL Jitter    | Time             | Avoid synchronized expiry     |
| Single‑Flight | Concurrency      | One rebuild, many wait        |
| Early Refresh | Probability      | Spread rebuilds over time     |
| Refresh‑Ahead | Scheduling       | Rebuild before expiry         |
| Never‑Expire  | Control          | Manual ownership of freshness |
| Rate Limiting | Damage           | Cap backend pressure          |

---

## 8. TTL Jitter — Breaking Time Alignment

### Mental Model

The problem is **many keys expiring at the same moment**.

The fix: *make time fuzzy*.

```mermaid
flowchart TD
    Uniform[Same Expiry]
    Jittered[Distributed Expiry]

    Uniform -->|Add randomness| Jittered
```

Instead of one spike → many small ripples.

---

## 9. Single‑Flight (Request Coalescing)

### Core Idea

> For a given key, **only one rebuild is allowed at a time**.

```mermaid
flowchart TD
    R1[Request 1]
    R2[Request 2]
    R3[Request 3]
    Build[Rebuild Data]

    R1 --> Build
    R2 -->|Wait| Build
    R3 -->|Wait| Build
```

Everyone shares the result.

---

### Key Insight

Waiting is *cheaper* than rebuilding.

---

## 10. Early Refresh (Probabilistic Thinking)

### Concept

Instead of refreshing **after** expiry, refresh **before**, randomly.

```mermaid
flowchart LR
    Cache[Cached Data]
    TTL[TTL Decreases]
    Chance[Refresh Probability ↑]

    Cache --> TTL --> Chance
```

Load is smoothed across time.

---

### Trade‑off

* Slight staleness
* Massive stability gain

---

## 11. Refresh‑Ahead (Predictive Defense)

### Concept

Known hot data is refreshed *proactively*.

```mermaid
flowchart TD
    Scheduler[Background Refresh]
    Cache
    Requests

    Scheduler --> Cache
    Requests -->|Always hit| Cache
```

Users never trigger rebuilds.

---

## 12. Never‑Expire Strategy

### Concept

Expiration is removed as a failure trigger.

> Freshness becomes a *business responsibility*, not a cache side‑effect.

Used when:

* Data is critical
* Updates are explicit

---

## 13. Layered Mental Model (Correct Architecture)

```mermaid
flowchart TD
    Req[Request]
    L1[L1 Cache]
    L2[L2 Cache]
    Guard[Concurrency Guard]
    DB[Database]

    Req --> L1 --> L2
    L2 -->|Hit| Req
    L2 -->|Miss| Guard --> DB
```

Each layer reduces:

* Cost
* Concurrency
* Blast radius

---

## 14. Observability: Detecting Stampedes Early

### Signals That Matter

```mermaid
flowchart TD
    MissRate[Cache Miss Spike]
    DBQ[DB Query Spike]
    Latency[Latency Surge]

    MissRate --> Alert
    DBQ --> Alert
    Latency --> Alert
```

**Single‑key miss concentration** is the strongest indicator.

---

## 15. Key Takeaways

1. Cache stampede is a **coordination failure**, not a cache failure
2. Expiration is a *dangerous event*, not a convenience
3. Time synchronization creates risk
4. One rebuild per key is the golden rule
5. Waiting is cheaper than rebuilding
6. Layered defenses beat clever tricks
7. Stability > freshness under load

---

## Final Mental Model

> A resilient cache does not answer requests faster — it decides *who is allowed to rebuild reality*.

If your database is the first responder to traffic spikes, the system is already broken.
