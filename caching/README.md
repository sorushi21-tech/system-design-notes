# Caching in System Design - Complete Guide

This directory contains comprehensive notes on caching for system design, targeting senior-level engineers.

## 📚 Table of Contents

### Core Concepts
1. **[Caching Overview](./caching.md)** - Introduction, fundamentals, and decision frameworks
2. **[Redis Basics](./redis-basics.md)** - In-depth guide to Redis as a caching solution

### Cache Architecture
3. **[Local vs Distributed Cache](./local-vs-distributed-cache.md)** - Comparing L1 and L2 caching strategies
4. **[Two-Level Cache](./two-level-cache.md)** - Multi-tier caching architecture (Coming Soon)

### Caching Patterns
5. **[Cache-Aside Pattern](./cache-aside.md)** - Lazy loading, most common pattern
6. **[Read-Through Pattern](./read-through.md)** - Cache-managed data loading
7. **[Write-Through Pattern](./write-through.md)** - Synchronous cache and database updates
8. **[Write-Behind Pattern](./write-behind.md)** - Asynchronous database writes

### Cache Management
9. **[Cache Invalidation](./cache-invalidation.md)** - Strategies for keeping cache consistent
10. **[Cache Eviction Policies](./cache-eviction-policies.md)** - LRU, LFU, FIFO, and more
11. **[Cache Warming](./cache-warming.md)** - Preventing cold start problems

### Common Problems & Solutions
12. **[Cache Stampede](./cache-stampede.md)** - Handling thundering herd problem
13. **[Cache Penetration](./cache-penetration.md)** - Preventing bypass attacks
14. **[Multi-Tenant Caching](./multi-tenant-caching.md)** - Caching in multi-tenant systems (Coming Soon)

---

## 🎯 Quick Navigation by Topic

### By Experience Level

**Beginner → Intermediate**:
1. [Caching Overview](./caching.md) - Start here
2. [Cache-Aside Pattern](./cache-aside.md) - Most common pattern
3. [Redis Basics](./redis-basics.md) - Essential tool

**Intermediate → Advanced**:
1. [Cache Invalidation](./cache-invalidation.md) - The hard problem
2. [Cache Eviction Policies](./cache-eviction-policies.md) - Performance optimization
3. [Local vs Distributed Cache](./local-vs-distributed-cache.md) - Architecture decisions

**Advanced**:
1. [Cache Stampede](./cache-stampede.md) - Production-level challenges
2. [Write-Behind Pattern](./write-behind.md) - High-performance writes
3. [Cache Penetration](./cache-penetration.md) - Security considerations

---

### By Problem Type

**Performance Optimization**:
- [Redis Basics](./redis-basics.md) - Fast data structures
- [Local vs Distributed Cache](./local-vs-distributed-cache.md) - Latency optimization
- [Cache Eviction Policies](./cache-eviction-policies.md) - Hit rate optimization

**Consistency & Correctness**:
- [Cache Invalidation](./cache-invalidation.md) - Data freshness
- [Write-Through Pattern](./write-through.md) - Strong consistency
- [Cache-Aside Pattern](./cache-aside.md) - Eventually consistent

**Scalability**:
- [Local vs Distributed Cache](./local-vs-distributed-cache.md) - Scaling strategies
- [Write-Behind Pattern](./write-behind.md) - Write scalability
- [Redis Basics](./redis-basics.md) - Clustering and sharding

**Reliability & Resilience**:
- [Cache Stampede](./cache-stampede.md) - Failure prevention
- [Cache Penetration](./cache-penetration.md) - Attack prevention
- [Cache Warming](./cache-warming.md) - Cold start prevention

---

## 📖 Learning Path

### Path 1: Fundamentals (1-2 weeks)

**Week 1**:
- Day 1-2: [Caching Overview](./caching.md)
- Day 3-4: [Cache-Aside Pattern](./cache-aside.md)
- Day 5-7: [Redis Basics](./redis-basics.md)

**Week 2**:
- Day 1-3: [Cache Eviction Policies](./cache-eviction-policies.md)
- Day 4-5: [Cache Invalidation](./cache-invalidation.md)
- Day 6-7: Practice implementing patterns

### Path 2: Advanced Topics (2-3 weeks)

**Week 1**:
- Day 1-2: [Local vs Distributed Cache](./local-vs-distributed-cache.md)
- Day 3-4: [Write-Through Pattern](./write-through.md)
- Day 5-7: [Read-Through Pattern](./read-through.md)

**Week 2**:
- Day 1-3: [Write-Behind Pattern](./write-behind.md)
- Day 4-5: [Cache Stampede](./cache-stampede.md)
- Day 6-7: [Cache Penetration](./cache-penetration.md)

**Week 3**:
- Day 1-3: [Cache Warming](./cache-warming.md)
- Day 4-7: Build a real caching system

### Path 3: Interview Preparation (1 week)

**Focus Areas**:
1. [Caching Overview](./caching.md) - Key concepts and trade-offs
2. [Cache-Aside Pattern](./cache-aside.md) - Most commonly asked
3. [Cache Invalidation](./cache-invalidation.md) - Famous hard problem
4. [Cache Stampede](./cache-stampede.md) - Production challenges
5. [Redis Basics](./redis-basics.md) - Common technology

**Practice Questions**:
- Design a URL shortener (caching strategy)
- Design a rate limiter (Redis-based)
- Design a leaderboard system (sorted sets)
- Handle cache stampede in e-commerce
- Design multi-level caching for microservices

---

## 🔑 Key Concepts Summary

### When to Use Caching

✅ **Good Candidates**:
- Read-heavy workloads (read:write > 10:1)
- Expensive computations (complex queries, aggregations)
- Frequently accessed data (80/20 rule applies)
- Acceptable staleness (eventual consistency OK)
- Slow data sources (legacy systems, third-party APIs)

❌ **Poor Candidates**:
- Write-heavy workloads (write:read > 1:1)
- Strong consistency requirements (financial transactions)
- Highly dynamic data (changes on every request)
- Large, infrequently accessed data
- Unique, personalized data (low hit rate)

---

### Cache Patterns Comparison

| Pattern | Consistency | Write Speed | Read Speed | Complexity | Use Case |
|---------|-------------|-------------|------------|------------|----------|
| **Cache-Aside** | Eventual | Fast | Fast | Low | General purpose ✅ |
| **Read-Through** | Eventual | Fast | Fast | Low | Standard reads |
| **Write-Through** | Strong | Slow | Fast | Medium | Read-heavy, consistency critical |
| **Write-Behind** | Weak | Very Fast | Fast | High | Write-heavy, analytics |

---

### Common Problems

| Problem | Impact | Solution | Document |
|---------|--------|----------|----------|
| **Cache Stampede** | Database overload on expiration | Locking, jittered TTL | [Cache Stampede](./cache-stampede.md) |
| **Cache Penetration** | Bypass cache with invalid keys | Null caching, Bloom filter | [Cache Penetration](./cache-penetration.md) |
| **Cache Avalanche** | Many keys expire simultaneously | Jittered TTL, warming | [Cache Invalidation](./cache-invalidation.md) |
| **Cold Start** | Empty cache after restart | Cache warming | [Cache Warming](./cache-warming.md) |
| **Stale Data** | Cache out of sync with DB | Invalidation strategy | [Cache Invalidation](./cache-invalidation.md) |
| **Hot Key** | Single key overloaded | Local cache, replication | [Redis Basics](./redis-basics.md) |

---

## 🛠️ Technology Stack

### Distributed Caching
- **Redis** - Most popular, feature-rich ✅ Recommended
- **Memcached** - Simple, high-performance
- **Hazelcast** - Java-native, distributed
- **Apache Ignite** - Distributed database + cache

### Local Caching (Java)
- **Caffeine** - Modern, high-performance ✅ Recommended
- **Guava Cache** - Mature, widely used
- **Ehcache** - Enterprise features
- **ConcurrentHashMap** - Manual implementation

### Caching Frameworks
- **Spring Cache** - Abstraction layer ✅ Recommended for Spring
- **JCache (JSR-107)** - Java standard
- **Hibernate Second-Level Cache** - ORM integration

---

## 📊 Performance Benchmarks

### Typical Latencies

```
L1 (Local Cache):     < 100 nanoseconds
L2 (Redis - same DC): 1-2 milliseconds
Database Query:       50-100 milliseconds
External API:         100-500 milliseconds
```

### Cache Hit Rate Targets

```
Excellent: > 95%
Good:      80-95%
Acceptable: 60-80%
Poor:      < 60% (investigate caching strategy)
```

### Redis Performance

```
Single Instance: 100,000+ ops/sec
Cluster:        1,000,000+ ops/sec (linear scaling)
Latency:        < 1ms (p99)
```

---

## 🎓 Interview Tips

### Common Interview Questions

1. **"Design a caching strategy for [system]"**
   - Start with [Caching Overview](./caching.md)
   - Choose pattern from [Cache-Aside](./cache-aside.md) or [Write-Through](./write-through.md)
   - Discuss trade-offs

2. **"How do you handle cache invalidation?"**
   - Reference [Cache Invalidation](./cache-invalidation.md)
   - Discuss TTL + event-based approaches
   - Mention Phil Karlton's quote

3. **"What happens when cache server fails?"**
   - Graceful degradation to database
   - Circuit breaker pattern
   - Monitoring and alerting

4. **"How do you prevent cache stampede?"**
   - See [Cache Stampede](./cache-stampede.md)
   - Locking, jittered TTL, refresh-ahead
   - Real-world examples

5. **"Local cache vs distributed cache?"**
   - See [Local vs Distributed Cache](./local-vs-distributed-cache.md)
   - Discuss latency, consistency, scalability trade-offs

### Framework for Answering

1. **Clarify Requirements**
   - Read/write ratio?
   - Consistency requirements?
   - Latency SLA?
   - Scale (QPS, data size)?

2. **Propose Solution**
   - Cache type (local vs distributed)
   - Cache pattern (aside, through, etc.)
   - TTL strategy
   - Eviction policy

3. **Address Trade-offs**
   - Consistency vs performance
   - Complexity vs benefits
   - Cost vs value

4. **Handle Edge Cases**
   - Cache stampede
   - Cache penetration
   - Failures and recovery

---

## 🚀 Best Practices Checklist

### Design Phase
- [ ] Identify data suitable for caching (80/20 rule)
- [ ] Choose appropriate cache pattern
- [ ] Define TTL strategy
- [ ] Plan invalidation strategy
- [ ] Consider multi-level caching

### Implementation Phase
- [ ] Set TTL on all cache entries
- [ ] Add jitter to prevent thundering herd
- [ ] Implement graceful degradation
- [ ] Use connection pooling
- [ ] Handle serialization efficiently

### Operations Phase
- [ ] Monitor hit/miss rates
- [ ] Track eviction rates
- [ ] Set up alerting (high miss rate, errors)
- [ ] Plan cache warming strategy
- [ ] Test failure scenarios

### Security Phase
- [ ] Implement rate limiting
- [ ] Protect against cache penetration
- [ ] Encrypt sensitive cached data
- [ ] Implement access control
- [ ] Audit cache access

---

## 📝 Code Examples Repository

All documents include production-ready code examples in:
- **Java** (Spring Boot, Guava, Caffeine)
- **Python** (Redis client)
- **Node.js** (Redis client)
- **Configuration** (Redis, Spring)

### Example Projects

Build these projects to practice:

1. **Beginner**: Simple product catalog with cache-aside
2. **Intermediate**: Multi-level cache for user service
3. **Advanced**: E-commerce system with all patterns
4. **Expert**: Rate limiter with Redis

---

## 🤝 Contributing

These notes are comprehensive but always evolving. Topics to potentially add:
- Multi-tenant caching strategies
- Cache in microservices
- Caching with Kubernetes
- Cache monitoring and observability
- Advanced Redis features (Streams, Pub/Sub)

---

## 📚 Additional Resources

### Books
- "Redis in Action" by Josiah Carlson
- "Designing Data-Intensive Applications" by Martin Kleppmann
- "Web Scalability for Startup Engineers" by Artur Ejsmont

### Online Resources
- Redis Documentation: https://redis.io/documentation
- Caffeine Wiki: https://github.com/ben-manes/caffeine/wiki
- Spring Cache Documentation: https://spring.io/guides/gs/caching

### Papers
- "The Chubby Lock Service for Loosely-Coupled Distributed Systems" (Google)
- "Scaling Memcache at Facebook" (Facebook)

---

## ✅ Completion Checklist

Track your learning progress:

### Core Concepts
- [ ] Understand when to use caching
- [ ] Know different cache types
- [ ] Understand cache hierarchy (L1, L2)

### Patterns
- [ ] Implement cache-aside pattern
- [ ] Understand write-through vs write-behind
- [ ] Know read-through pattern

### Problems
- [ ] Can prevent cache stampede
- [ ] Can handle cache penetration
- [ ] Understand cache invalidation strategies

### Tools
- [ ] Familiar with Redis basics
- [ ] Can choose eviction policy
- [ ] Know how to monitor cache

### Production
- [ ] Can design cache warming strategy
- [ ] Understand failure scenarios
- [ ] Can optimize cache performance

---

**Last Updated**: 2026-01-17
**Maintainer**: System Design Study Plan
**Target Audience**: Senior Software Engineers preparing for system design interviews or building production caching systems
**Difficulty Range**: Beginner to Advanced

---

Happy Caching! 🚀
