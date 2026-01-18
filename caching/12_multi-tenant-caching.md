# Multi-Tenant Caching

## 1. What is Multi-Tenancy?

**Multi-tenancy** means:

* A **single application instance**
* Serving **multiple tenants (customers/clients)**
* Each tenant’s data must be **isolated**

```
Same Application
 ├── Tenant A
 ├── Tenant B
 └── Tenant C
```

Each tenant feels like they have their **own system**, even though the infrastructure is shared.

---

## 2. What is Multi-Tenant Caching?

**Multi-Tenant Caching** means caching data **without breaking tenant isolation**.

> Cached data for one tenant must **never** be visible to another tenant.

Caching improves performance, but in multi-tenant systems it also introduces **data leakage risks**.

---

## 3. Why Multi-Tenant Caching Is Difficult

Caching is usually **global**, but tenants require **strict separation**.

### Key Challenges

1. **Data Leakage** – Wrong tenant sees cached data
2. **Tenant-Specific Invalidation** – Clear cache for one tenant only
3. **Memory Overuse** – One tenant fills entire cache
4. **Uneven Usage** – Large tenants dominate cache
5. **Operational Complexity** – Monitoring per tenant

---

## 4. Core Problem Example

Without tenant awareness:

```
Cache Key: account:123
```

❌ Problem:

* Tenant A → account:123
* Tenant B → account:123

Same key → **incorrect data returned**

---

## 5. Golden Rule of Multi-Tenant Caching

> **Every cache operation must be tenant-aware**

This is achieved by:

* Including **Tenant ID in cache keys**, or
* Using **isolated cache spaces per tenant**

---

## 6. Multi-Tenant Caching Strategies

---

### Strategy 1: Tenant-Aware Cache Keys (Most Common)

#### Concept

Tenant ID is added to every cache key.

```
Cache Key = tenantId : resourceKey
```

#### Flow

```
Request
  ↓
Resolve Tenant ID
  ↓
Create Tenant-Aware Cache Key
  ↓
Cache Lookup
```

#### Example

```
TenantA:account:123
TenantB:account:123
```

#### Pros

* Simple
* Works with Redis, Caffeine, Hazelcast
* Easy to scale

#### Cons

* Cache grows quickly
* Isolation is logical, not physical

---

### Strategy 2: Cache Namespace per Tenant

#### Concept

Each tenant gets a **logical namespace** inside the cache.

```
Cache
 ├── TenantA Namespace
 └── TenantB Namespace
```

#### Flow

```
Request
  ↓
Identify Tenant
  ↓
Switch Namespace
  ↓
Cache Read/Write
```

#### Pros

* Clear separation
* Easier tenant-level invalidation

#### Cons

* Depends on cache technology
* Still shared infrastructure

---

### Strategy 3: Separate Cache per Tenant

#### Concept

Each tenant has a **dedicated cache instance**.

```
Tenant A → Redis A
Tenant B → Redis B
```

#### Flow

```
Request
  ↓
Resolve Tenant
  ↓
Select Tenant Cache
  ↓
Read / Write
```

#### Pros

* Strong isolation
* No risk of data leakage
* Easy tenant deletion

#### Cons

* High cost
* Operational overhead

> Best suited for **large enterprise tenants**

---

### Strategy 4: Hybrid Model (Recommended)

#### Concept

* Small tenants → Shared cache
* Large tenants → Dedicated cache

```
                ┌── Shared Cache (Small Tenants)
Tenant Request ──┤
                └── Dedicated Cache (Large Tenant)
```

#### Pros

* Cost efficient
* Scales well
* Common in SaaS platforms

---

## 7. Cache Invalidation in Multi-Tenant Systems

### Why It Is Hard

Invalidation must be:

* **Correct**
* **Tenant-specific**
* **Fast**

### Invalidation Flow

```
Tenant Data Change
  ↓
Identify Tenant
  ↓
Evict Only Tenant Cache
```

### Common Techniques

1. **Prefix-Based Eviction**

   ```
   tenantA:*
   ```

2. **Tenant Versioning**

   ```
   tenantA:v3:account:123
   ```

3. **Event-Based Invalidation**

    * Database change → Event → Cache eviction

---

## 8. Cache Fairness & Memory Control

### Problem

One tenant can:

* Flood cache
* Evict other tenants’ data

### Solutions

* Per-tenant cache quota
* Tenant-aware LRU eviction
* Rate limiting cache writes
* Tenant tier-based limits

```
Tenant A → 500MB
Tenant B → 100MB
```

---

## 9. Cache Access Patterns

| Pattern       | Description                    |
| ------------- | ------------------------------ |
| Cache-Aside   | App controls cache explicitly  |
| Read-Through  | Cache loads data automatically |
| Write-Through | Write to DB and cache together |
| Write-Behind  | Async DB writes                |

> **Cache-Aside + Tenant-Aware Keys** is most commonly used

---

## 10. High-Level Architecture

```
Client
  ↓
API Gateway
  ↓
Tenant Resolver
  ↓
Service Layer
  ↓
Tenant-Aware Cache
  ↓
Tenant-Isolated Database
```

---

## 11. Best Practices

- Always propagate tenant context
- Never share mutable cached data across tenants
- Provide tenant-level cache clear APIs
- Monitor cache usage per tenant
- Log tenant ID in cache metrics

---

## 12. What Not to Cache Per Tenant

- Cross-tenant shared mutable data
- Sensitive data without encryption
- Tenant metadata without versioning
- Authentication data without tenant scope

---

## 13. When to Avoid Multi-Tenant Caching

* Very small datasets
* Highly volatile data
* Strong compliance requirements
* Extreme isolation needs

---

## 14. Real-World FinTech Example

```
Tenant = Bank
Data = Account Statements

Cache Key:
bankId:statement:accountId:period
```

* Bank-level isolation
* Tenant-based invalidation
* Controlled cache usage per bank

---

## 15. Key Takeaway

> **Multi-Tenant Caching = Caching + Tenant Isolation + Fair Resource Usage**

Always remember:

**Tenant context must flow through every cache operation**
