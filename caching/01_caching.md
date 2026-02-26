# Caching in System Design

---

## 1. What is Caching?

Caching means storing a copy of data in a fast place so that next time someone asks for it, we can give it quickly — without going to the original slow source (like a database).

**Main benefits:** Faster responses, less load on database, and lower server costs.

---

## 2. Cache Hit vs Cache Miss

- **Cache HIT:** Data is found in the cache. Response is very fast (~1ms). No need to go to database.
- **Cache MISS:** Data is NOT in the cache. We have to go to database (~50ms). After fetching, we save it in cache for next time.
- **Hit Rate:** Out of 100 requests, how many were served from cache. Higher hit rate = better performance.
---

## 3. Types of Caches

| Cache Type           | Where it sits                                    | Example                        |
|----------------------|--------------------------------------------------|--------------------------------|
| Browser Cache        | User's browser or mobile app                     | Chrome storing images locally  |
| CDN Cache            | Edge servers near the user                       | Cloudflare, Akamai             |
| L1 Local Cache       | Inside the application (same server)             | Caffeine, Guava (Java)         |
| L2 Distributed Cache | Separate cache server, shared by all app servers | Redis, Memcached               |
| Database Cache       | Inside the database itself                       | MySQL Query Cache, Buffer Pool |
| CPU Cache            | Hardware level inside processor                  | L1/L2/L3 CPU cache             |

---

## 4. Why Caching is So Important

### Performance Benefits
- Memory is much faster than disk. 
  - Memory access = nanoseconds. 
  - Disk = milliseconds. 
  - Network = 10–100ms.
- More requests can be handled per server. Less pressure on database.

### Cost Benefits
- Database gets less load — so you need fewer DB servers.
- Fewer calls to other microservices and third-party APIs.
- Lower infrastructure cost overall.

### Real World Example

```
Without Cache:
  Every request --> DB query (50ms)
  1000 req/sec  --> 50,000ms DB load --> Database crashes!

With Cache (95% hit rate):
  950 requests  --> Cache (1ms)   = 950ms total
   50 requests  --> DB (50ms)     = 2,500ms total
  Grand total: 3,450ms  (vs 50,000ms without cache)
  That is 14x improvement!
```

### Scalability Benefits
- Caching helps scale app servers horizontally. No need to keep scaling the database.
- Handles sudden traffic spikes easily — like during Diwali sale or IPL match ticket booking!

---

## 5. Cache Hierarchy ( Multiple Layers )

Modern systems use many cache layers. Data travels from fastest to slowest:

1. **Browser Cache** — Fastest (0ms). Only for that one user.
2. **CDN/Edge Cache** — Very fast (10–50ms). Covers a whole geographic region.
3. **L1 Local Cache** — Fast (0.1–1ms). Lives inside one app server only.
4. **L2 Distributed Cache** — Medium fast (1–5ms). Shared across all app servers.
5. **Database** — Slowest (50–100ms). The final source of truth.

**Why use so many layers?** Because the closer the data is to the user, the faster it gets delivered. If one layer fails, the next layer takes over.

---

## 6. How to Measure Cache Performance

| Metric        | What it means                        | Good Target                         |
|---------------|--------------------------------------|-------------------------------------|
| Hit Rate      | % of requests served from cache      | 80–95%+                             |
| Miss Rate     | % of requests that missed cache      | Lower is better                     |
| Cache Latency | Time to read/write from cache        | <1ms for in-memory                  |
| Eviction Rate | How often data is removed from cache | Low (means cache is sized properly) |
| Memory Usage  | How full the cache is                | Not more than 80–90%                |
| Throughput    | Requests per second cache can handle | Redis handles 100K+ ops/sec         |

---

## 7. When to Use Caching

### Good Situations to Cache

1. **Read-heavy workloads** — When read data 10x more than write (e.g., product catalog, user profiles, news feed).
2. **Expensive calculations** — Like running complex reports, ML model predictions, analytics.
3. **Frequently accessed data** — The famous 80/20 rule: 20% of data is accessed 80% of the time.
4. **Slow third-party APIs** — If some external API takes 2–3 seconds, cache its response.
5. **Static or rarely changing data** — Like city/state lists, configuration settings, public holidays.
6. **Rate-limited APIs** — To avoid hitting their request limits.

### When NOT to Cache

- Real-time data that changes every second — like live stock prices, live cricket scores.
- Write-heavy systems — If  writing data more than reading, cache becomes useless.
- Data needing strict consistency — Like banking transactions, inventory count.
- Very large but rarely accessed data — Wastes memory.
- Sensitive data (PII, passwords, payment info) — Unless encrypted properly.

---

## 8. Cache Architecture Patterns

### 1. Cache-Aside (Lazy Loading)

The application itself is responsible for loading data into cache. Cache does NOT automatically fetch from DB.

- Step 1: Application checks cache.
- Step 2: If MISS, application fetches from DB.
- Step 3: Application stores result in cache.
- Step 4: Returns data to user.

**Good for:** Most microservices, simple read-heavy APIs.  
**Drawback:** First request is always slow (miss penalty). Data can become stale.

---

### 2. Read-Through Cache

Cache sits in front of DB. When there is a MISS, the cache itself fetches data from DB automatically. Application does not need to know.

**Good for:** When you want the caching logic hidden from your application code.  
**Drawback:** First request still slow. Cache layer becomes more complex.

---

### 3. Write-Through Cache

Every write goes to cache AND database at the same time (synchronously).

**Good for:** When consistency between cache and DB is very important.  
**Drawback:** Every write is slower because it must update both cache and DB.

---

### 4. Write-Behind (Write-Back)

Write updates the cache immediately. Database is updated later, asynchronously in background.

**Good for:** High write throughput use cases. When eventual consistency is okay.  
**Risk:** If cache crashes before DB update, you lose data!

---

### 5. Refresh-Ahead

Cache proactively refreshes popular data before it expires. No one has to wait for a miss.

**Good for:** Predictable access patterns (like dashboard data refreshed every 5 minutes).  
**Drawback:** Wastes resources if refreshing data that no one actually needs.

---

## 9. Common Caching Problems

### Problem 1: Cache Stampede

Suppose a very popular data item expires from cache. At that exact moment, 1000 users request it. All 1000 requests find a MISS and hit the database at the same time. Database gets overloaded!

**Solution 1 — Request Coalescing:** Only the first request goes to DB. Other requests wait for it to complete.  
**Solution 2 — Probabilistic Refresh:** Start refreshing data slightly before it expires, not after.  
**Solution 3 — Never Expire Hot Keys:** Keep refreshing the most popular data continuously.

---

### Problem 2: Cache Penetration

User keeps requesting data that does not exist at all (like a product ID that was never created). Every request misses cache and hits DB. DB keeps saying "not found" but gets hammered anyway.

**Solution 1 — Cache Null Result:** Store "null" or "not found" in cache with short TTL.  
**Solution 2 — Bloom Filter:** A data structure that quickly says whether a key MIGHT exist. If Bloom Filter says no, don't even go to DB.  
**Solution 3 — Validate Input:** Check if the ID format is valid before querying.

---

### Problem 3: Cache Avalanche

Many cache keys expire at the SAME TIME. All those requests suddenly hit the database together.

**Solution:** Add random jitter (random extra seconds) to TTL so keys don't all expire together.

```
TTL = 3600 + random(0, 300) seconds
```

---

### Problem 4: Hot Key Problem

One particular key is so popular (like a trending news article) that one cache node gets all the traffic. That node gets overloaded.

**Solution:** Copy the hot key to multiple cache nodes. Or cache it locally inside the application memory as well.

---

### Problem 5: Cache Consistency (Stale Data)

Cache has old data. Database has new data. User sees wrong information.

**Solution 1 — Short TTL:** Data expires quickly. Accept small delay in freshness.  
**Solution 2 — Invalidation on Write:** Whenever data is updated in DB, delete or update the cache key immediately.  
**Solution 3 — Event-Driven Updates:** Use database change streams to detect changes and update cache automatically.

---

### Problem 6: Cold Start Problem

After cache server restarts, cache is empty. Suddenly all requests are MISS. Database gets 100% load.

**Solution:** Warm up the cache by pre-loading frequently accessed data before opening traffic. Use Redis persistence so cache survives restarts.

---

## 10. Best Practices

### 1. Cache Only What is Needed
Focus on data that is requested frequently. Monitor hit rate regularly and aim for 80% or more.

### 2. Set Proper TTL
TTL (Time To Live) is how long data stays in cache before it expires.

- Fast-changing data (like product availability): Short TTL — a few minutes.
- Slow-changing data (like user profile): Medium TTL — 1 hour.
- Rarely changing data (like country list): Long TTL — 24 hours or more.
- Always add random jitter to TTL to prevent cache avalanche.

### 3. Use Good Key Names

```
Format:  {namespace}:{entity}:{id}:{version}
Example: user:profile:12345:v2
         product:details:9876:v1
         order:summary:4567:v3
```

### 4. Graceful Degradation
System should work even if cache goes down. Always have a fallback — if cache is not available, go directly to database. Log cache failures so you can monitor and fix quickly.

### 5. Choose the Right Eviction Policy

| Policy                      | What it does                                | Best for                           |
|-----------------------------|---------------------------------------------|------------------------------------|
| LRU (Least Recently Used)   | Removes data that was not accessed recently | Most general use cases             |
| LFU (Least Frequently Used) | Removes data that is rarely accessed        | When access frequency matters more |
| TTL (Time To Live)          | Removes data after a fixed time             | Data that becomes stale over time  |
| FIFO (First In First Out)   | Removes oldest data first                   | Rarely used in practice            |

### 6. Cache Size

```
Cache Size = Working Set x Safety Factor
Working Set  = All data that is actively used
Safety Factor = 1.5 to 2.0

Too small = high miss rate
Too large = wasted memory and cost
```
