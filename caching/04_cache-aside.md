# Cache-Aside Pattern (Lazy Loading)

---

## 1. What is Cache-Aside Pattern?

In this pattern, the **application itself** is responsible for managing the cache — checking it, loading data into it, and clearing it when needed.


**Why "Cache-Aside"?** The cache sits "beside" the main flow. Your application talks to both the cache and the database directly. The cache is not in between — it is on the side.

The application decides:
- When to check the cache
- When to fetch from the database
- When to update or delete from the cache

---

## 2. How it Works

### Reading Data — Step by Step

```
Step 1: User requests data for key "user:123"
Step 2: Application checks cache first

  → CACHE HIT (data found):
    Return data immediately. Fast! (~1ms)

  → CACHE MISS (data not found):
    Step 3: Query the database (~50ms)
    Step 4: If data exists, store it in cache with a TTL
    Step 5: Return data to user
```

### Writing / Updating Data — Step by Step

```
Step 1: Update request arrives for "user:123"
Step 2: Update the database first (database is always the source of truth)
Step 3: Delete (invalidate) the cache key "user:123"
Step 4: Return success to user

Next time someone reads "user:123":
→ Cache miss → Fetches fresh data from DB → Stores in cache
```

**Why DELETE the cache key instead of updating it?**
Deleting is simpler and safer. There is no risk of a race condition. The next read will automatically fetch the latest data from the database.

---


## 3. Advantages

### 1. Resilient — Cache Failure Does Not Break the App
If Redis goes down, application still works! It just falls back to the database directly. Users will experience slightly slower response but the app will not crash.

### 2. Efficient Memory Usage — Lazy Loading
Only data that is actually requested gets cached.

### 3. Database Remains Source of Truth
No data is lost because the database always has the correct data.

---

## 4. Disadvantages

### 1. Cache Miss Penalty — 3 Operations Instead of 1

On a cache miss, three things happen:
```
Check cache (~1ms) + Query DB (~50ms) + Write to cache (~1ms) = ~52ms total

Compare to cache hit: ~1-2ms only
```

Cache miss is 25 to 50 times slower than a cache hit!

### 2. Cold Start Problem
When application restarts or gets freshly deployed, cache is completely empty. All requests for the first 10 to 15 minutes will be cache misses and will hit the database directly. Database load spikes up during this time.

**Solution:** Implement cache warming — pre-load popular data into cache before opening traffic.

### 3. Stale Data Risk
Suppose someone updates a user's name directly in the database using an admin tool (bypassing your application). The cache still has the old name. It will remain stale until the TTL expires.


### 4. Cache Stampede Risk
Suppose a very popular key (like a trending product) expires at 10:00:00 AM. At 10:00:01, 100 users request it at the same time. All 100 find a cache miss and all 100 hit the database simultaneously. Database gets overloaded!

**Solution:** Add TTL jitter — randomize expiry time so many keys do not expire at the same time.

### 5. Race Condition on Write

```
Thread A (READ):              Thread B (WRITE):
1. Check cache → MISS
2. Query DB → gets old value
                              3. Update DB with new value
                              4. Delete cache key
5. Writes OLD value to cache! ← Problem!
```

Now cache has stale data again. This is a real race condition issue in high-traffic systems.

**Solution:** Use shorter TTLs. Or use write-through pattern for critical data.

---

## 6. Best Practices

### 1. Always Set TTL — Never Leave a Key Without Expiry

```
BAD — no expiry, key lives forever:
cache.set("user:123", data)

GOOD — always set TTL:
cache.set("user:123", data, TTL=3600)   # 1 hour

BETTER — add random jitter to prevent stampede:
ttl = 3600 + random(0, 300)             # 3600 to 3900 seconds
cache.set("user:123", data, TTL=ttl)
```

**TTL Guidelines by data type:**
- Frequently changing data (order status): 5 to 15 minutes
- Semi-static data (user profile): 1 to 6 hours
- Rarely changing data (product catalog): 24 hours
- Config/reference data (country list): 12 to 24 hours

---

### 2. Always Handle Cache Failures Gracefully

Cache is an enhancement — not a requirement.
```
function getData(key):
    try:
        data = cache.get(key)
        if data found:
            return data
    catch CacheError:
        log("Cache unavailable, falling back to database")
        # Do NOT crash here — just continue

    data = database.query(key)

    try:
        cache.set(key, data, TTL)
    catch CacheError:
        log("Could not write to cache, continuing anyway")
        # Do NOT crash here either — just log and move on

    return data
```

---

### 3. Invalidate (Delete), Do Not Update Cache on Write

```
RECOMMENDED — Invalidate:
function updateData(id, newData):
    database.save(newData)
    cache.delete("user:" + id)    # Next read will fetch fresh data

RISKY — Update cache:
function updateData(id, newData):
    database.save(newData)
    cache.set("user:" + id, newData)   # Race condition possible!
```

Why deletion is safer: No race condition. No risk of writing wrong data. Next read guarantees fresh data from database.

---

### 4. Use Consistent and Meaningful Key Names

```
BAD:
cache.set("123", data)         # What is this?
cache.set("data", data)        # Too generic
cache.set("u", data)           # Too cryptic

GOOD:
cache.set("user:123", data)
cache.set("product:456:details", data)
cache.set("user:123:orders:recent", data)
```

**Pattern:** `{entity}:{id}:{sub-resource}`

This helps in debugging, monitoring, and doing bulk operations (delete all `user:*` keys).

---

### 5. Cache Null Results — Negative Caching

**Problem:** Someone keeps searching for `product:999999` which does not exist. Every request is a cache miss and hits the database. DB keeps saying "not found" but keeps getting bombarded.

**Solution:** Cache the "not found" result too!

```
function getData(key):
    cached = cache.get(key)

    if cached == "NOT_FOUND":
        return null            # Cache HIT for "not found" — protected!

    if cached found:
        return cached

    data = database.query(key)

    if data found:
        cache.set(key, data, TTL=3600)
    else:
        cache.set(key, "NOT_FOUND", TTL=60)   # Short TTL for nulls!

    return data
```

**Important:** Use a SHORT TTL (60 to 300 seconds) for null/not-found results. You do not want to cache a "not found" result for too long in case the item gets created later.

---

### 6. Prevent Cache Stampede

**Solution 1 — TTL Jitter (Easiest):**
```
ttl = base_ttl + random(0, base_ttl * 0.1)
# Example: 3600 + random(0, 360) = 3600 to 3960 seconds
# Keys expire at different times → no simultaneous stampede
```

**Solution 2 — Locking (For Critical Hot Keys):**
```
function getData(key):
    data = cache.get(key)
    if data found:
        return data

    # Try to acquire a lock for this key
    if acquire_lock(key, timeout=5 seconds):
        try:
            # Double-check cache — another request may have filled it
            data = cache.get(key)
            if data found:
                return data

            # Load from DB and populate cache
            data = database.query(key)
            cache.set(key, data, TTL)
            return data
        finally:
            release_lock(key)
    else:
        # Another request is loading the data, wait and retry
        sleep(100ms)
        return getData(key)   # Retry
```

**Solution 3 — Refresh Before Expiry:**
```
if cache.ttl(key) < 60:   # Less than 1 minute remaining
    background_task.refresh_cache(key)   # Refresh in background
```

---

### 7. Monitor Cache Performance

**Key metrics to watch:**

| Metric        | Target     | Action if bad                                  |
|---------------|------------|------------------------------------------------|
| Hit Rate      | Above 80%  | Increase TTL or cache more data                |
| Miss Rate     | Below 20%  | Check if popular data is being cached          |
| Cache Latency | Below 2ms  | Check Redis server load                        |
| Memory Usage  | 70 to 80%  | Increase cache size or evict more aggressively |
| Error Rate    | Below 0.1% | Investigate Redis connection issues            |

---

## 7. Advanced Patterns

### Single-Flight Pattern — One DB Call for Many Concurrent Requests

When 100 requests for the same key arrive at the same time (all cache misses), only ONE of them should query the database. The other 99 should wait for that result.

```
function getData(id):
    data = cache.get(key)
    if data found:
        return data

    # Check if another request is already loading this data
    if isInFlight(id):
        return waitForResult(id)   # Wait for the in-progress request

    # Mark this key as "loading"
    markInFlight(id)

    try:
        data = database.query(id)
        cache.set(key, data, TTL)
        completeInFlight(id, data)
        return data
    finally:
        clearInFlight(id)
```

**Benefit:** Database gets only 1 query instead of 100 for the same key.

---

### Bulk Loading — Fetch Many Keys at Once

Instead of making 100 separate cache calls for 100 keys, fetch them all in one batch operation.

```
function getBulkData(ids[]):
    # Step 1: Fetch all from cache in one call
    cached_results = cache.multiGet(ids)

    # Step 2: Find which ones were cache misses
    missed_ids = []
    results = {}
    for each id:
        if cached_results[id] found:
            results[id] = cached_results[id]
        else:
            missed_ids.add(id)

    # Step 3: Query database for only the misses (in one batch)
    if missed_ids not empty:
        db_results = database.findMultiple(missed_ids)
        cache.multiSet(db_results, TTL)     # Write to cache in batch
        results.merge(db_results)

    return results
```

**Benefit:** 1 network round trip instead of 100 for bulk reads.

---

## 8. When to Use Cache-Aside

### Use it When:

| Situation                | Why Cache-Aside Works                    |
|--------------------------|------------------------------------------|
| Read-heavy applications  | Most requests served from cache          |
| User profiles / sessions | Frequently read, rarely updated          |
| Product catalogs         | Many reads, occasional price updates     |
| Configuration data       | Read thousands of times, changed rarely  |
| Social media feeds       | High read load, slight staleness is okay |
| API rate limiting        | Simple counter + TTL pattern             |

---

### Do NOT Use it When:

| Situation                                          | Better Alternative                  |
|----------------------------------------------------|-------------------------------------|
| Strong consistency is required (financial data)    | Write-Through Pattern               |
| Very write-heavy workload                          | Write-Behind Pattern                |
| Real-time stock prices or live scores              | Skip caching, read directly from DB |
| Very small dataset that fits entirely in DB memory | No caching needed                   |

---

## 9. Real World Examples

### Example 1: E-Commerce Product Page

Scenario: 10,000 products, 10 lakh daily page views, products updated a few times per day.

- First user visits product 123 → Cache miss → Fetches from DB → Stores in Redis with 1 hour TTL
- Next 999 users visit product 123 → Cache hit every time → DB not touched at all
- Result: 99%+ reduction in DB queries, response time drops from 50ms to 2ms

---

### Example 2: User Session Validation

Scenario: Every API request must validate the user's session token.

- User logs in → Session stored in DB and also in Redis: `session:abc123 → {user_id: 101}` with 30 min TTL
- Every API call → Check Redis first (1ms) instead of DB (50ms)
- If Redis is down → Fall back to DB (slower but still works)
- Result: 99.9% cache hit rate, less than 1ms session validation

---

### Example 3: Social Media User Profiles

Scenario: Profile pages are visited very frequently, but users update their profile rarely.

- Key pattern: `user:profile:{user_id}`, TTL = 1 hour + jitter
- On profile visit: Check cache → Miss on first time → Load from DB → Cache it
- On profile update: Update DB → Delete `user:profile:{user_id}` from cache
- Next visit after update: Cache miss → Fetches fresh data from DB → Caches again
- Result: 95% hit rate, profile page loads in 3ms vs 80ms from DB

---

## 10. Comparison with Other Cache Patterns

| Aspect              | Cache-Aside         | Read-Through | Write-Through            | Write-Behind              |
|---------------------|---------------------|--------------|--------------------------|---------------------------|
| Who manages cache   | Application         | Cache layer  | Cache layer              | Cache layer               |
| Code complexity     | Medium              | Low          | Medium                   | High                      |
| Read performance    | Fast (after warmup) | Fast         | Fast                     | Fast                      |
| Write performance   | Fast                | Fast         | Slower (both cache + DB) | Fastest                   |
| Consistency         | Eventual            | Eventual     | Strong                   | Eventual                  |
| Cache miss handling | Application handles | Automatic    | Automatic                | Automatic                 |
| Risk of data loss   | Low                 | Low          | Very low                 | Medium (if cache crashes) |
| Best for            | Read-heavy apps     | Simple apps  | Consistency needed       | Write-heavy apps          |

---

## 11. Quick Summary

Cache-Aside (Lazy Loading) is the go-to caching pattern for most applications because it is simple, flexible, and resilient.

Key things to remember:
- Application manages the cache — full control, full responsibility.
- Data is loaded into cache **only when requested** (that is why it is called "lazy").
- On cache miss: check cache → query DB → store in cache → return data.
- On write: update DB first → delete cache key (not update — delete!).
- Cache failure must NEVER crash the application — always fall back to DB.
- Always set TTL — never leave keys without expiry.
- Add jitter to TTL to prevent cache stampede.
- Use negative caching to protect DB from "not found" attacks.

---
