# Cache Invalidation


## 1. What Is Cache Invalidation? (Pure Concept)

**Cache invalidation** is the act of ensuring cached data does **not lie**.

A cache is a *copy* of data. The moment the original data changes, the copy becomes suspicious.

```
Truth (DB) changes
        ↓
Cache becomes potentially wrong
        ↓
System must react
```

If the system fails to react correctly, users see **outdated reality**.

---

## 2. The Core Mental Model

> Cache is an **optimization**, not a source of truth.

Think of cache as:

* A *hint* to speed things up
* A *temporary shortcut*
* Something that is **allowed to be wrong briefly**

Never design systems that *depend on cache correctness*.

---

## 3. Why Cache Invalidation Is Fundamentally Hard

### The Time Gap Problem

The hardest part is the **gap between change and awareness**.

```mermaid
flowchart LR
    A[Data Updated]
    B[Cache Still Old]
    C[User Reads Cache]
    D[Wrong Decision]

    A --> B --> C --> D
```

Even a very small gap can cause damage.

---

### Four Root Causes (Conceptual)

#### 1. Time & Ordering

Operations do not happen in a clean sequence.

```
Read happens
Update happens
Invalidate happens
```

But **not always in that order**.

---

#### 2. Partial Failure

```mermaid
flowchart TD
    DB[Database Update]
    Cache[Cache Invalidation]

    DB -->|Success| Cache
    Cache -->|Failure| Stale[Stale Cache Persists]
```

Databases and caches fail **independently**.

---

#### 3. Hidden Relationships

One data change affects many views of reality.

```mermaid
flowchart TD
    User[User Updated]
    P1[Profile Cache]
    P2[List Cache]
    P3[Search Cache]
    P4[Feed Cache]

    User --> P1
    User --> P2
    User --> P3
    User --> P4
```

Missing *one* relationship breaks correctness.

---

#### 4. Distribution

Multiple machines hold different memories.

```mermaid
flowchart LR
    DB[(Database)]
    A[App A]
    B[App B]
    C[App C]

    DB --> A
    DB --> B
    DB --> C

    A --> CacheA[Cache]
    B --> CacheB[Cache]
    C --> CacheC[Cache]
```

Invalidation must **travel**, not just happen.

---

## 4. Strategy Landscape (Concept-Only View)

| Strategy         | Core Idea                       | Key Trade-off          |
| ---------------- | ------------------------------- | ---------------------- |
| TTL              | Let time fix the problem        | Temporary wrongness    |
| Delete-on-Change | Remove copy after truth changes | Must know all copies   |
| Event-Based      | React to change events          | High coordination cost |
| Tag-Based        | Invalidate groups               | Over-invalidation risk |
| Version-Based    | Ignore old reality              | Memory overhead        |

---

## 5. Time-Based Invalidation (TTL)

### Concept

Data expires **whether or not it was updated**.

```mermaid
flowchart LR
    CacheWrite[Cache Written]
    Valid[Temporarily Valid]
    Expire[Auto Expire]
    Reload[Reload from Truth]

    CacheWrite --> Valid --> Expire --> Reload
```

### Why TTL Exists

TTL is not about freshness — it is about **safety**.

> TTL guarantees that *no lie lives forever*.

### Core Trade-off

```mermaid
flowchart TD
    ShortTTL[Short TTL]
    Fresh[Fresh Data]
    Load[High DB Load]

    LongTTL[Long TTL]
    Stale[Stale Data]
    LowLoad[Low DB Load]

    ShortTTL --> Fresh --> Load
    LongTTL --> Stale --> LowLoad
```

---

## 6. Delete-on-Change (Purge Model)

### Concept

When truth changes, **destroy the copy**.

```mermaid
sequenceDiagram
    participant DB as Database
    participant C as Cache
    participant U as User

    DB->>DB: Update
    DB->>C: Invalidate
    U->>C: Read (miss)
    U->>DB: Fetch fresh
```

### Key Insight

Deleting is safer than updating because it avoids **late-arriving stale writes**.

---

## 7. Event-Based Invalidation

### Concept

Changes emit signals. Caches react.

```mermaid
flowchart LR
    Change[State Change]
    Event[Event Emitted]
    Invalidate[Invalidate Caches]

    Change --> Event --> Invalidate
```

### Why It’s Powerful

* Targeted
* Immediate
* Precise

### Why It’s Dangerous

* Requires perfect knowledge
* Missed events = silent corruption

---

## 8. Tag-Based Invalidation

### Concept

Cache entries belong to **groups**.

```mermaid
flowchart TD
    Tag[Tag: Category]
    K1[Cache A]
    K2[Cache B]
    K3[Cache C]

    Tag --> K1
    Tag --> K2
    Tag --> K3
```

Invalidate the group, not individuals.

---

## 9. Version-Based Invalidation

### Concept

Reality moves forward. Old reality is ignored.

```mermaid
flowchart LR
    V1[Version 1]
    V2[Version 2]
    V3[Version 3]

    V1 -->|Ignored| X1[Stale]
    V2 -->|Ignored| X2[Stale]
    V3 --> Valid[Current Truth]
```

No deletion. No coordination. Just progression.

---

## 10. Distributed Invalidation (Big Picture)

### Core Problem

```mermaid
flowchart LR
    Update[Update Happens]
    Node1[Node 1]
    Node2[Node 2]
    Node3[Node 3]

    Update --> Node1
    Node2 -->|Still stale| Node2
    Node3 -->|Still stale| Node3
```

### Conceptual Solution

```mermaid
flowchart LR
    Update --> Signal
    Signal --> AllNodes[All Nodes]
    AllNodes --> InvalidateAll[Invalidate Local Cache]
```

---

## 11. Strategy Selection (Thinking Flow)

```mermaid
flowchart TD
    A[Data Changes?]
    B[Can You Detect Change?]
    C[Strong Consistency Needed?]

    A -->|Rare| TTL[Long TTL]
    A -->|Often| B
    B -->|No| ShortTTL[Short TTL]
    B -->|Yes| C
    C -->|Yes| Event[Event-Based or No Cache]
    C -->|No| Hybrid[Event + TTL]
```

---

## 12. Golden Rules (Conceptual)

1. Cache must **never define truth**
2. TTL is mandatory — always
3. Deleting copies is safer than fixing copies
4. Invalidation failures must not break business logic
5. Distributed systems need distributed signals
6. Expect failure — design for it

---

## 13. Final Mental Model

> Cache invalidation is not about perfection.

It is about:

* Limiting damage
* Bounding inconsistency
* Recovering automatically

A good system assumes cache will lie — and survives anyway.
