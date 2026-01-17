# Caching in System Design

## Table of Contents
1. [Introduction](#introduction)
2. [What is Caching?](#what-is-caching)
3. [Why Caching is Critical](#why-caching-is-critical)
4. [Cache Hierarchy](#cache-hierarchy)
5. [Cache Performance Metrics](#cache-performance-metrics)
6. [When to Use Caching](#when-to-use-caching)
7. [When NOT to Use Caching](#when-not-to-use-caching)
8. [Cache Architecture Patterns](#cache-architecture-patterns)
9. [Common Caching Challenges](#common-caching-challenges)
10. [Best Practices](#best-practices)

---

## Introduction

Caching is one of the most fundamental techniques in system design for improving performance, reducing latency, and scaling applications. As a senior engineer, understanding caching goes beyond just "storing data in memory" — it involves making strategic decisions about cache placement, consistency models, eviction policies, and handling edge cases that can make or break production systems.

This document provides a comprehensive overview of caching concepts, patterns, and practices essential for designing scalable systems.

---

## What is Caching?

**Caching** is a technique that stores copies of frequently accessed data in a fast-access storage layer (cache) to serve future requests more quickly. Instead of repeatedly fetching data from slower sources (databases, APIs, disk), the application retrieves it from the cache.

### Core Concept
```
Client → Cache (fast) → Data Source (slow)
         ↓ Hit          ↓ Miss
       Return          Fetch, Store, Return
```

### Types of Caches by Location
1. **Client-Side Cache**: Browser cache, mobile app cache
2. **CDN Cache**: Edge locations for static content
3. **Application-Level Cache**: In-memory cache within the application (local cache)
4. **Distributed Cache**: Shared cache across multiple application instances (Redis, Memcached)
5. **Database Cache**: Query result cache, buffer pool
6. **CPU Cache**: L1, L2, L3 hardware caches

---

## Why Caching is Critical

### Performance Benefits
- **Reduced Latency**: Memory access (microseconds) vs disk access (milliseconds) or network calls (10-100ms+)
- **Higher Throughput**: Serve more requests with the same resources
- **Better User Experience**: Faster page loads, instant responses

### Cost & Resource Benefits
- **Lower Database Load**: Reduces expensive database queries (80-90% reduction possible)
- **Reduced Backend Calls**: Fewer API calls to microservices or third-party services
- **Lower Infrastructure Costs**: Handle more traffic with fewer servers
- **Network Bandwidth Savings**: Especially with CDN caching

### Scalability Benefits
- **Horizontal Scaling**: Cache helps distribute load before needing to scale databases
- **Traffic Spike Handling**: Absorb sudden traffic spikes without overwhelming backend
- **Bottleneck Prevention**: Database often becomes the bottleneck; caching prevents this

### Real-World Impact
```
Without Cache:
- Request → DB query (50ms)
- 1000 req/s → 50,000ms DB load → Database overload

With Cache (95% hit rate):
- 950 requests → Cache (1ms) = 950ms
- 50 requests → DB (50ms) = 2,500ms
- Total: 3,450ms vs 50,000ms (14x improvement)
```

---

## Cache Hierarchy

Modern systems often employ multiple cache layers working together:

```
┌─────────────────────────────────────┐
│  Browser Cache (Client)             │ ← Fastest
├─────────────────────────────────────┤
│  CDN Edge Cache                     │
├─────────────────────────────────────┤
│  Application Local Cache (L1)       │
├─────────────────────────────────────┤
│  Distributed Cache - Redis (L2)     │
├─────────────────────────────────────┤
│  Database Query Cache               │
├─────────────────────────────────────┤
│  Database (Source of Truth)         │ ← Slowest
└─────────────────────────────────────┘
```

### Multi-Level Cache Strategy
- **L1 (Local Cache)**: In-process memory, fastest, but not shared
- **L2 (Distributed Cache)**: Shared across instances, slightly slower, consistent
- **L3 (Database Cache)**: Last resort before hitting tables

**Why Multiple Levels?**
- Optimize for different access patterns
- Balance speed vs consistency
- Reduce network hops
- Graceful degradation

---

## Cache Performance Metrics

### 1. Hit Rate / Hit Ratio
```
Hit Rate = (Cache Hits / Total Requests) × 100%
```
- **Good**: 80-95%+ for read-heavy workloads
- **Poor**: <50% indicates ineffective caching strategy

### 2. Miss Rate
```
Miss Rate = (Cache Misses / Total Requests) × 100%
Miss Rate = 100% - Hit Rate
```

### 3. Cache Latency
- **Read Latency**: Time to fetch from cache (typically <1ms for in-memory)
- **Write Latency**: Time to update cache

### 4. Eviction Rate
```
Eviction Rate = Items Evicted / Time Period
```
- High eviction rate → Cache too small or poor key design

### 5. Memory Utilization
```
Memory Usage = (Used Memory / Total Cache Size) × 100%
```
- Monitor to prevent cache exhaustion

### 6. Throughput
- Requests per second the cache can handle
- Redis: 100K+ ops/sec on single instance

---

## When to Use Caching

### Ideal Use Cases

1. **Read-Heavy Workloads**
   - Read:Write ratio > 10:1
   - Examples: Product catalogs, user profiles, news feeds

2. **Expensive Computations**
   - Complex aggregations, reports, analytics
   - ML model predictions
   - Recommendation results

3. **Frequently Accessed Data**
   - Hot data (80/20 rule: 80% of requests go to 20% of data)
   - Session data, user preferences

4. **Slow Data Sources**
   - Third-party API calls
   - Legacy systems with high latency
   - Complex database joins

5. **Static or Slowly Changing Data**
   - Configuration data
   - Reference data (countries, currencies)
   - Content that updates hourly/daily

6. **Rate-Limited APIs**
   - Cache responses to avoid hitting rate limits
   - Reduce costs for pay-per-request APIs

### Data Characteristics Suitable for Caching
- **High Read Frequency**: Accessed many times
- **Low Write Frequency**: Doesn't change often
- **Predictable Access Patterns**: Same data requested repeatedly
- **Acceptable Staleness**: Can tolerate slightly outdated data
- **Size**: Reasonable size (KBs to MBs, not GBs per item)

---

## When NOT to Use Caching

### Anti-Patterns

1. **Highly Dynamic Data**
   - Data changes on every request
   - Real-time stock prices (unless microsecond staleness is OK)
   - Live sports scores

2. **Write-Heavy Workloads**
   - Write:Read ratio > 1:1
   - Constant cache invalidation overhead

3. **Large, Infrequently Accessed Data**
   - Wastes cache memory
   - Better served from storage

4. **Strong Consistency Requirements**
   - Financial transactions
   - Inventory in e-commerce (can lead to overselling)
   - When eventual consistency is unacceptable

5. **Personalized, Unique Data**
   - Every user gets different data
   - Low cache hit rate
   - Example: Personalized dashboards with unique metrics

6. **Sensitive Data Without Proper Security**
   - PII, payment info (unless encrypted)
   - Data subject to compliance (GDPR, HIPAA) without proper controls

---

## Cache Architecture Patterns

### 1. Cache-Aside (Lazy Loading)
**Most common pattern**

```
Read:
1. Check cache
2. If miss, query database
3. Store in cache
4. Return data

Write:
1. Update database
2. Invalidate/update cache
```

**Pros**: Simple, data loaded on-demand
**Cons**: Cache miss penalty, potential stale data

---

### 2. Read-Through Cache
```
Application → Cache → Database
```
Cache sits between app and database. On miss, cache loads from DB.

**Pros**: Transparent to app, cache manages loading
**Cons**: Miss penalty, complexity in cache layer

---

### 3. Write-Through Cache
```
Write:
1. Update cache
2. Synchronously update database
3. Return success
```

**Pros**: Cache always consistent with DB
**Cons**: Write latency (must wait for both), increased writes

---

### 4. Write-Behind (Write-Back)
```
Write:
1. Update cache
2. Asynchronously update database (batched)
3. Return success immediately
```

**Pros**: Fast writes, write coalescing
**Cons**: Risk of data loss, complexity, eventual consistency

---

### 5. Refresh-Ahead
Proactively refresh cache before expiration.

**Pros**: Prevents cache misses for hot data
**Cons**: Complexity, may refresh data that won't be accessed

---

## Common Caching Challenges

### 1. Cache Stampede (Thundering Herd)
**Problem**: Many requests hit expired key simultaneously, all query DB

**Solution**:
- Single-flight pattern (only one request loads data)
- Probabilistic early expiration
- Lock-based loading

### 2. Cache Penetration
**Problem**: Queries for non-existent data bypass cache, hit DB

**Solution**:
- Cache negative results (null values) with short TTL
- Bloom filter to check existence
- Request validation

### 3. Cache Avalanche
**Problem**: Many keys expire simultaneously, overwhelming backend

**Solution**:
- Randomize TTLs (base_ttl ± random_jitter)
- Stagger cache warming
- Circuit breaker pattern

### 4. Hot Key Problem
**Problem**: Single key receives too much traffic, overwhelming cache node

**Solution**:
- Local cache for hot keys
- Data replication across nodes
- Request coalescing

### 5. Big Key Problem
**Problem**: Very large cached values cause memory issues, slow operations

**Solution**:
- Split large objects
- Compress data
- Set size limits

### 6. Cache Consistency
**Problem**: Cache and database become out of sync

**Solution**:
- Proper invalidation strategy
- TTL-based expiration
- Event-driven invalidation
- Version tags

### 7. Cold Start Problem
**Problem**: Empty cache leads to poor performance after restart/deployment

**Solution**:
- Cache warming strategies
- Gradual traffic ramp-up
- Persistent cache (Redis persistence)

---

## Best Practices

### Design Principles

1. **Cache Only What Matters**
   - Don't cache everything
   - Focus on hot data (80/20 rule)
   - Monitor hit rates

2. **Set Appropriate TTLs**
   - Short TTL for frequently changing data (1-5 minutes)
   - Long TTL for stable data (hours to days)
   - Add jitter to prevent thundering herd

3. **Use Consistent Key Naming**
   ```
   Pattern: {namespace}:{entity}:{id}:{version}
   Example: user:profile:12345:v2
   ```

4. **Implement Graceful Degradation**
   - System works when cache fails
   - Fall back to database
   - Monitor cache availability

5. **Monitor Cache Health**
   - Hit/miss rates
   - Latency
   - Memory usage
   - Eviction rate
   - Error rates

6. **Size Your Cache Appropriately**
   ```
   Cache Size = Working Set Size × Safety Factor
   Safety Factor = 1.5-2.0 (allow for growth)
   ```

7. **Choose Right Eviction Policy**
   - **LRU (Least Recently Used)**: General purpose
   - **LFU (Least Frequently Used)**: For stable access patterns
   - **TTL**: Time-based expiration
   - **FIFO**: Simple, predictable

8. **Handle Serialization Efficiently**
   - Use efficient formats (Protocol Buffers, MessagePack)
   - Compress large objects
   - Consider serialization overhead

9. **Implement Circuit Breakers**
   - Protect backend when cache fails
   - Prevent cascade failures

10. **Test Cache Failures**
    - Chaos engineering
    - What happens when cache is down?
    - Performance degradation acceptable?

### Security Considerations

1. **Encrypt Sensitive Data**
   - Use encryption at rest for sensitive cached data
   - Consider memory encryption

2. **Access Control**
   - Restrict cache access (network policies, authentication)
   - Separate caches for different tenants

3. **Avoid Caching Secrets**
   - Passwords, API keys, tokens
   - If necessary, use short TTLs and encryption

4. **Data Privacy**
   - GDPR compliance: right to be forgotten
   - Implement cache invalidation for user data deletion

---

## Caching Decision Matrix

| Scenario | Cache Type | Pattern | TTL |
|----------|------------|---------|-----|
| User Profile | Distributed | Cache-Aside | 1 hour |
| Session Data | Distributed | Write-Through | Session lifetime |
| Product Catalog | CDN + Distributed | Read-Through | 24 hours |
| API Rate Limiting | Local | Write-Through | 1 minute |
| Search Results | Distributed | Cache-Aside | 5 minutes |
| Configuration | Local + Distributed | Read-Through | 30 minutes |
| Analytics Results | Distributed | Cache-Aside | 12 hours |
| Real-time Inventory | Don't Cache | N/A | N/A |

---

## Key Takeaways for Senior Engineers

1. **Caching is not a silver bullet** — understand trade-offs between consistency, performance, and complexity

2. **Cache invalidation is hard** — as Phil Karlton said, "There are only two hard things in Computer Science: cache invalidation and naming things"

3. **Design for failure** — caches will fail; systems must degrade gracefully

4. **Monitor everything** — cache effectiveness is measurable; use data to guide decisions

5. **Think in layers** — multi-level caching provides optimal performance

6. **Consider the cost** — caching has memory cost, complexity cost, and consistency cost

7. **Know your data** — access patterns dictate caching strategy

8. **Test cache scenarios** — including failures, stampedes, and cold starts

---

## Further Reading

- [Redis Basics](./redis-basics.md)
- [Local vs Distributed Cache](./local-vs-distributed-cache.md)
- [Cache Patterns Deep Dive](./cache-aside.md)
- [Cache Invalidation Strategies](./cache-invalidation.md)
- [Cache Eviction Policies](./cache-eviction-policies.md)
- [Handling Cache Problems](./cache-stampede.md)

---

**Last Updated**: 2026-01-17
**Target Audience**: Senior Software Engineers
**Difficulty Level**: Intermediate to Advanced
