# Cache Stampede (Thundering Herd) Problem

## Table of Contents
1. [Introduction](#introduction)
2. [The Problem](#the-problem)
3. [Real-World Impact](#real-world-impact)
4. [Root Causes](#root-causes)
5. [Detection and Symptoms](#detection-and-symptoms)
6. [Solution Strategies](#solution-strategies)
7. [Implementation Examples](#implementation-examples)
8. [Best Practices](#best-practices)
9. [Related Problems](#related-problems)

---

## Introduction

**Cache Stampede** (also called **Thundering Herd**, **Dog-Pile Effect**, or **Cache Stampede Problem**) occurs when many requests for the same cached resource hit simultaneously after that resource expires or is invalidated, causing all requests to query the backend database at once.

This can overwhelm the database and cause cascading failures in production systems.

---

## The Problem

### Normal Cache Operation

```
Time: 10:00:00
Request 1 → Cache HIT → Return data (1ms)
Request 2 → Cache HIT → Return data (1ms)
Request 3 → Cache HIT → Return data (1ms)
... (all requests served from cache)

Database: Idle ✓
```

### Cache Stampede Scenario

```
Time: 10:00:00 - Cache entry expires
Time: 10:00:00.001 - 1000 concurrent requests arrive

Request 1   → Cache MISS → Query DB
Request 2   → Cache MISS → Query DB
Request 3   → Cache MISS → Query DB
...
Request 1000 → Cache MISS → Query DB

Database: Receives 1000 simultaneous queries ✗
Result: Database overload, timeouts, cascading failures
```

### Visual Representation

```
Normal Flow:
────────────────────────────────────────
Requests:  ████████████████████████████
Cache:     ✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓
Database:  (idle)

Cache Stampede:
────────────────────────────────────────
                ↓ Cache expires
Requests:  ─────████████████████████────
Cache:     ✓✓✓✓✗✗✗✗✗✗✗✗✗✗✗✗✗✗✗✗✓✓✓✓
Database:  ─────🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥────
                (overloaded)
```

---

## Real-World Impact

### Case Study: E-commerce Flash Sale

```
Scenario: Popular product page, Black Friday sale
Cache TTL: 60 seconds
Traffic: 10,000 requests/second
Cache expires at 11:00:00

Impact:
- 11:00:00.000: Cache expires
- 11:00:00.001-11:00:01.000: 10,000 requests hit database
- Database: 300ms latency → 30s to process all
- Database connections: Exhausted (max 100 connections)
- Result: Site down for 2 minutes

Loss: $100,000+ in sales
```

### Case Study: Social Media Timeline

```
Scenario: Celebrity posts update
Cache invalidated globally
1M followers refresh timeline simultaneously

Impact:
- Database: 1M queries in 1 second
- Normal load: 10K queries/second (100x spike)
- Database crashes
- Cascading failure to other services
- Recovery time: 15 minutes

Loss: User trust, negative press
```

---

## Root Causes

### 1. Synchronized TTL Expiration

```java
// BAD: All keys expire at same time
for (Product product : products) {
    cache.setex("product:" + product.getId(), 3600, product);
}

// Result: At T+3600, all 10,000 products expire simultaneously
```

### 2. Mass Cache Invalidation

```java
// BAD: Invalidate entire cache on single update
public void updateProductPrice(String productId) {
    database.update(productId);
    cache.clear();  // Nuclear option: invalidates EVERYTHING
}
```

### 3. Popular Key Pattern

```java
// Popular key accessed 1000s of times per second
String popularKey = "homepage:featured_products";

// When this expires → stampede
```

### 4. No Request Coalescing

```java
// BAD: Each thread independently queries database
public Product getProduct(String id) {
    Product cached = cache.get(id);
    if (cached == null) {
        // All threads execute this simultaneously
        Product fresh = database.query(id);
        cache.set(id, fresh);
        return fresh;
    }
    return cached;
}
```

---

## Detection and Symptoms

### Metrics to Monitor

```java
// Sudden spike in:
- Database query count (10-100x normal)
- Database connection pool exhaustion
- Cache miss rate (spike to 100%)
- Request latency (10-100x normal)
- Database CPU usage (spike to 100%)
- Error rates (timeouts, connection errors)
```

### Monitoring Code

```java
@Component
public class CacheMonitor {
    
    @Scheduled(fixedRate = 1000)  // Every second
    public void detectStampede() {
        long cacheMisses = metrics.getCacheMisses();
        long dbQueries = metrics.getDatabaseQueries();
        
        // Detect stampede: unusually high miss rate + DB queries
        if (cacheMisses > MISS_THRESHOLD && 
            dbQueries > DB_THRESHOLD) {
            
            logger.error("CACHE STAMPEDE DETECTED! " +
                "Misses: {}, DB Queries: {}", cacheMisses, dbQueries);
            
            alerting.triggerAlert("cache_stampede");
        }
    }
}
```

### Symptoms

**Before Stampede**:
```
Cache hit rate: 95%
DB queries/sec: 100
Avg latency: 50ms
```

**During Stampede**:
```
Cache hit rate: 0% (for affected keys)
DB queries/sec: 10,000+ (100x spike)
Avg latency: 5,000ms+ (100x spike)
Error rate: 50%+ (timeouts)
```

---

## Solution Strategies

### Strategy Comparison

| Solution | Complexity | Effectiveness | Performance Impact |
|----------|------------|---------------|-------------------|
| Locking | Medium | High | Low (single DB query) |
| Probabilistic Early Refresh | Low | Medium | Very Low |
| Request Coalescing | Medium | High | Low |
| External Recomputation | High | High | None |
| Never Expire | Low | High | None (manual refresh) |
| Distributed Locking | High | High | Low-Medium |

---

### Solution 1: Locking (Mutex)

**Concept**: Only allow one request to query database while others wait.

```java
public class CacheWithLocking {
    private final ConcurrentHashMap<String, CompletableFuture<Product>> locks = 
        new ConcurrentHashMap<>();
    
    public Product getProduct(String id) {
        String key = "product:" + id;
        
        // 1. Check cache
        Product cached = redis.get(key);
        if (cached != null) {
            return cached;
        }
        
        // 2. Acquire lock for this key
        CompletableFuture<Product> future = locks.computeIfAbsent(id, k -> {
            return CompletableFuture.supplyAsync(() -> {
                try {
                    // Only ONE thread executes this block
                    
                    // Double-check cache (might be populated by now)
                    Product rechecked = redis.get(key);
                    if (rechecked != null) {
                        return rechecked;
                    }
                    
                    // Query database
                    Product product = database.findProduct(id);
                    
                    // Update cache
                    if (product != null) {
                        redis.setex(key, 3600, product);
                    }
                    
                    return product;
                } finally {
                    // Release lock
                    locks.remove(id);
                }
            });
        });
        
        // 3. Wait for the lock holder to complete
        return future.join();
    }
}
```

**Flow**:
```
Thread 1: Miss → Acquire lock → Query DB → Cache → Release lock
Thread 2: Miss → Wait for Thread 1's lock
Thread 3: Miss → Wait for Thread 1's lock
...
Thread N: Miss → Wait for Thread 1's lock

Result: Only 1 DB query, all threads wait for result
```

**Advantages**:
- ✅ Single database query
- ✅ Works for any key
- ✅ Simple logic

**Disadvantages**:
- ❌ Waiting threads consume resources
- ❌ Single point of contention per key

---

### Solution 2: Distributed Locking (Redis)

For multi-instance deployments:

```java
public class DistributedCacheWithLocking {
    
    public Product getProduct(String id) {
        String cacheKey = "product:" + id;
        String lockKey = "lock:" + id;
        
        // 1. Check cache
        Product cached = redis.get(cacheKey);
        if (cached != null) return cached;
        
        // 2. Try to acquire distributed lock
        String lockValue = UUID.randomUUID().toString();
        boolean locked = redis.set(lockKey, lockValue, "NX", "EX", 10);
        
        if (locked) {
            try {
                // This instance won the lock
                
                // Double-check cache
                cached = redis.get(cacheKey);
                if (cached != null) return cached;
                
                // Query database
                Product product = database.findProduct(id);
                
                // Update cache
                if (product != null) {
                    redis.setex(cacheKey, 3600, product);
                }
                
                return product;
            } finally {
                // Release lock (with Lua script for atomicity)
                releaseLock(lockKey, lockValue);
            }
        } else {
            // Another instance has the lock, wait and retry
            Thread.sleep(100);  // Wait 100ms
            return getProduct(id);  // Recursive retry (with max attempts)
        }
    }
    
    private void releaseLock(String lockKey, String lockValue) {
        String script = 
            "if redis.call('get', KEYS[1]) == ARGV[1] then " +
            "    return redis.call('del', KEYS[1]) " +
            "else " +
            "    return 0 " +
            "end";
        redis.eval(script, 1, lockKey, lockValue);
    }
}
```

---

### Solution 3: Probabilistic Early Expiration

**Concept**: Refresh cache before TTL expires, with increasing probability.

```java
public class ProbabilisticCache {
    private static final int ORIGINAL_TTL = 3600;  // 1 hour
    private final Random random = new Random();
    
    public Product getProduct(String id) {
        String key = "product:" + id;
        
        Product cached = redis.get(key);
        if (cached != null) {
            // Check TTL
            long ttl = redis.ttl(key);
            
            // Calculate refresh probability
            // As TTL decreases, probability increases
            double refreshProbability = 1.0 - (ttl / (double) ORIGINAL_TTL);
            
            // Example: TTL=3000 → 16% chance
            //          TTL=1800 → 50% chance
            //          TTL=600  → 83% chance
            
            if (random.nextDouble() < refreshProbability) {
                // Refresh in background (async)
                executor.submit(() -> {
                    Product fresh = database.findProduct(id);
                    redis.setex(key, ORIGINAL_TTL, fresh);
                });
            }
            
            // Return cached version (may be stale, but prevents stampede)
            return cached;
        }
        
        // Cache miss - normal load
        return loadAndCache(id);
    }
}
```

**Advantages**:
- ✅ Very simple
- ✅ No locking overhead
- ✅ Gradual refresh spreads load

**Disadvantages**:
- ❌ Still allows some stampede (reduced, not eliminated)
- ❌ May return stale data briefly

---

### Solution 4: Request Coalescing (Single Flight)

**Concept**: Merge duplicate in-flight requests.

```java
public class SingleFlightCache {
    private final Map<String, CompletableFuture<Product>> inFlight = 
        new ConcurrentHashMap<>();
    
    public CompletableFuture<Product> getProductAsync(String id) {
        String key = "product:" + id;
        
        // Check cache
        Product cached = redis.get(key);
        if (cached != null) {
            return CompletableFuture.completedFuture(cached);
        }
        
        // Check if request already in-flight
        return inFlight.computeIfAbsent(id, k -> {
            return CompletableFuture.supplyAsync(() -> {
                try {
                    // Only first request executes this
                    Product product = database.findProduct(id);
                    if (product != null) {
                        redis.setex(key, 3600, product);
                    }
                    return product;
                } finally {
                    inFlight.remove(id);
                }
            });
        });
    }
}
```

**Flow**:
```
Time T0:
  Request 1: Miss → Create Future → Query DB (in progress)
  
Time T0+1ms:
  Request 2: Miss → Found Future from Request 1 → Wait
  Request 3: Miss → Found Future from Request 1 → Wait
  
Time T0+50ms:
  Request 1: DB query complete → All waiting requests get result
```

---

### Solution 5: External Recomputation (Refresh-Ahead)

**Concept**: Background job refreshes cache before expiration.

```java
@Component
public class CacheWarmer {
    
    @Scheduled(fixedRate = 1800000)  // Every 30 minutes
    public void refreshPopularProducts() {
        // List of popular product IDs
        List<String> popularIds = getPopularProductIds();
        
        for (String id : popularIds) {
            try {
                // Refresh before expiration
                Product product = database.findProduct(id);
                redis.setex("product:" + id, 3600, product);
            } catch (Exception e) {
                logger.error("Failed to refresh product: {}", id, e);
            }
        }
    }
    
    private List<String> getPopularProductIds() {
        // Get top 1000 most accessed products
        return redis.zrevrange("popular:products", 0, 999);
    }
}
```

**Advantages**:
- ✅ Eliminates stampede for known hot keys
- ✅ Cache always warm
- ✅ Zero user-facing latency

**Disadvantages**:
- ❌ Requires knowing which keys to refresh
- ❌ Additional background job complexity
- ❌ May refresh unnecessary data

---

### Solution 6: Never Expire + Manual Refresh

**Concept**: Set very long TTL, refresh manually when data changes.

```java
public class NeverExpireCache {
    
    public Product getProduct(String id) {
        String key = "product:" + id;
        
        Product cached = redis.get(key);
        if (cached != null) {
            return cached;
        }
        
        Product product = database.findProduct(id);
        if (product != null) {
            // Very long TTL (7 days)
            redis.setex(key, 604800, product);
        }
        
        return product;
    }
    
    public void updateProduct(Product product) {
        database.save(product);
        
        // Manually refresh cache
        redis.setex("product:" + product.getId(), 604800, product);
    }
}
```

**Advantages**:
- ✅ No stampede from expiration
- ✅ Simple

**Disadvantages**:
- ❌ Must manually invalidate/update
- ❌ Stale data if manual refresh fails

---

### Solution 7: Jittered TTL

**Concept**: Add randomness to TTL to prevent synchronized expiration.

```java
public void cacheWithJitter(String key, Product value) {
    int baseTTL = 3600;  // 1 hour
    
    // Add random jitter: ±5 minutes (300 seconds)
    int jitter = random.nextInt(600) - 300;
    int finalTTL = baseTTL + jitter;  // 55-65 minutes
    
    redis.setex(key, finalTTL, value);
}
```

**Result**: Keys expire at different times, spreading load.

```
Without Jitter:
Key 1: Expires at T+3600
Key 2: Expires at T+3600
Key 3: Expires at T+3600
... all 1000 keys expire simultaneously

With Jitter:
Key 1: Expires at T+3400
Key 2: Expires at T+3750
Key 3: Expires at T+3520
... keys expire over 10-minute window
```

---

## Implementation Examples

### Complete Solution (Combining Multiple Strategies)

```java
@Service
public class RobustCacheService {
    private final LoadingCache<String, Product> localCache;  // L1
    private final JedisPool redisPool;  // L2
    private final ProductRepository repository;
    private final Map<String, CompletableFuture<Product>> inFlight = 
        new ConcurrentHashMap<>();
    private final Random random = new Random();
    
    private static final int BASE_TTL = 3600;
    private static final int JITTER = 300;
    
    public Product getProduct(String id) {
        // 1. L1 cache (local)
        Product local = localCache.getIfPresent(id);
        if (local != null) return local;
        
        // 2. L2 cache (Redis) with stampede prevention
        return getFromRedisWithStampedeProtection(id);
    }
    
    private Product getFromRedisWithStampedeProtection(String id) {
        String key = "product:" + id;
        
        try (Jedis jedis = redisPool.getResource()) {
            String cached = jedis.get(key);
            
            if (cached != null) {
                Product product = deserialize(cached);
                
                // Probabilistic early refresh
                long ttl = jedis.ttl(key);
                if (shouldRefreshEarly(ttl)) {
                    refreshAsync(id);
                }
                
                // Populate L1
                localCache.put(id, product);
                return product;
            }
        }
        
        // Cache miss - use request coalescing
        return loadWithCoalescing(id);
    }
    
    private boolean shouldRefreshEarly(long ttl) {
        if (ttl <= 0) return false;
        double refreshProbability = 1.0 - (ttl / (double) BASE_TTL);
        return random.nextDouble() < refreshProbability;
    }
    
    private void refreshAsync(String id) {
        CompletableFuture.runAsync(() -> {
            try {
                Product fresh = repository.findById(id).orElse(null);
                if (fresh != null) {
                    cacheProduct(id, fresh);
                }
            } catch (Exception e) {
                logger.warn("Background refresh failed for: {}", id, e);
            }
        });
    }
    
    private Product loadWithCoalescing(String id) {
        CompletableFuture<Product> future = inFlight.computeIfAbsent(id, k -> {
            return CompletableFuture.supplyAsync(() -> {
                try {
                    // Only one request executes this
                    Product product = repository.findById(id).orElse(null);
                    if (product != null) {
                        cacheProduct(id, product);
                    }
                    return product;
                } finally {
                    inFlight.remove(id);
                }
            });
        });
        
        try {
            return future.get(5, TimeUnit.SECONDS);
        } catch (Exception e) {
            logger.error("Failed to load product: {}", id, e);
            return null;
        }
    }
    
    private void cacheProduct(String id, Product product) {
        // L1
        localCache.put(id, product);
        
        // L2 with jittered TTL
        int ttl = BASE_TTL + random.nextInt(JITTER * 2) - JITTER;
        try (Jedis jedis = redisPool.getResource()) {
            jedis.setex("product:" + id, ttl, serialize(product));
        }
    }
}
```

---

## Best Practices

### 1. **Always Add Jitter to TTL**

```java
// BAD
cache.setex(key, 3600, value);

// GOOD
int ttl = 3600 + random.nextInt(600) - 300;  // 3300-3900 sec
cache.setex(key, ttl, value);
```

### 2. **Implement Request Coalescing for Hot Keys**

```java
// Maintain list of hot keys
Set<String> hotKeys = Set.of("homepage:products", "trending:items");

if (hotKeys.contains(key)) {
    return loadWithCoalescing(key);
} else {
    return loadNormally(key);
}
```

### 3. **Monitor for Stampede Patterns**

```java
@Scheduled(fixedRate = 60000)
public void detectStampedePatterns() {
    Map<String, Long> keyMisses = getKeyMissesLastMinute();
    
    for (Map.Entry<String, Long> entry : keyMisses.entrySet()) {
        if (entry.getValue() > STAMPEDE_THRESHOLD) {
            logger.warn("Potential stampede on key: {}, misses: {}", 
                entry.getKey(), entry.getValue());
        }
    }
}
```

### 4. **Use Circuit Breakers**

```java
@Autowired private CircuitBreaker circuitBreaker;

public Product getProduct(String id) {
    return circuitBreaker.executeSupplier(() -> {
        // Cache logic here
        return fetchFromCacheOrDB(id);
    });
}
```

### 5. **Gradual TTL Reduction**

```java
// Start with long TTL in production
int ttl = isProduction ? 7200 : 3600;
cache.setex(key, ttl, value);

// Gradually reduce after monitoring
```

---

## Related Problems

### Cache Avalanche

**Similar to stampede, but multiple keys expire:**

```
Stampede: Single popular key expires → surge
Avalanche: Many keys expire together → surge
```

**Solution**: Same strategies (jittered TTL, request coalescing)

### Cache Penetration

**Different problem: Queries for non-existent data:**

```
Stampede: Key exists but expired
Penetration: Key never existed, bypass cache every time
```

**Solution**: Cache null results, bloom filters

---

## Key Takeaways

1. **Cache stampede is a serious production issue** — can take down your database
2. **Combine multiple strategies** — jittered TTL + request coalescing + early refresh
3. **Monitor for stampede patterns** — high DB spikes, low cache hit rates
4. **Test under load** — stampede only appears at scale
5. **Don't over-optimize** — simple jittered TTL solves 90% of cases
6. **Plan for failure** — circuit breakers, fallbacks
7. **Hot keys need special handling** — dedicated stampede protection

---

## Next Steps

- [Cache Penetration Prevention](./cache-penetration.md)
- [Cache Warming Strategies](./cache-warming.md)
- [Write-Through Pattern](./write-through.md)

---

**Last Updated**: 2026-01-17
**Target Audience**: Senior Software Engineers
**Difficulty Level**: Advanced
