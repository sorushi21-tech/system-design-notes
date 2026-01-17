# Cache-Aside Pattern (Lazy Loading)

## Table of Contents
1. [Introduction](#introduction)
2. [Pattern Overview](#pattern-overview)
3. [How It Works](#how-it-works)
4. [Implementation](#implementation)
5. [Advantages](#advantages)
6. [Disadvantages](#disadvantages)
7. [Best Practices](#best-practices)
8. [Common Patterns and Variations](#common-patterns-and-variations)
9. [Handling Failures](#handling-failures)
10. [Performance Considerations](#performance-considerations)

---

## Introduction

**Cache-Aside** (also called **Lazy Loading** or **Look-Aside**) is the most common caching pattern. The application is responsible for managing cache reads and writes explicitly.

Unlike other patterns where the cache sits transparently between the application and database, in cache-aside the application talks to both cache and database directly.

---

## Pattern Overview

### Architecture

```
┌──────────────┐
│ Application  │
└───┬──────┬───┘
    │      │
    │      │ 1. Check cache
    │      ▼
    │   ┌──────┐
    │   │Cache │
    │   └──────┘
    │      │
    │      │ 2. On miss, query DB
    ▼      ▼
┌──────────────┐
│  Database    │
└──────────────┘
    │
    │ 3. Store in cache
    │ 4. Return to app
```

### Key Characteristics
- **Application-managed**: App controls cache logic
- **Lazy**: Data loaded only when requested
- **Cache-aside**: Cache is "beside" the main data flow
- **Most common**: Used in 80%+ of caching scenarios

---

## How It Works

### Read Flow

```
1. Application receives request
2. Check cache for key
3. If FOUND (cache hit):
   └─ Return cached data
4. If NOT FOUND (cache miss):
   ├─ Query database
   ├─ Store result in cache
   └─ Return data to caller
```

### Write Flow

```
1. Application receives write request
2. Update database
3. Invalidate cache (or update it)
4. Return success
```

---

## Implementation

### Basic Implementation (Java + Redis)

```java
public class CacheAsideService {
    private final JedisPool redisPool;
    private final ProductRepository repository;
    private final ObjectMapper objectMapper;
    
    public Product getProduct(String productId) {
        String cacheKey = "product:" + productId;
        
        // 1. Try to get from cache
        try (Jedis jedis = redisPool.getResource()) {
            String cached = jedis.get(cacheKey);
            
            // 2. Cache hit - return immediately
            if (cached != null) {
                return objectMapper.readValue(cached, Product.class);
            }
        } catch (Exception e) {
            logger.warn("Cache read error, falling back to DB", e);
        }
        
        // 3. Cache miss - query database
        Product product = repository.findById(productId);
        
        if (product == null) {
            return null;  // Or throw NotFoundException
        }
        
        // 4. Store in cache for next time
        try (Jedis jedis = redisPool.getResource()) {
            String json = objectMapper.writeValueAsString(product);
            jedis.setex(cacheKey, 3600, json);  // 1 hour TTL
        } catch (Exception e) {
            logger.warn("Cache write error", e);
            // Continue - cache failure shouldn't break the app
        }
        
        // 5. Return data
        return product;
    }
    
    public void updateProduct(String productId, Product product) {
        // 1. Update database first (source of truth)
        repository.save(product);
        
        // 2. Invalidate cache
        String cacheKey = "product:" + productId;
        try (Jedis jedis = redisPool.getResource()) {
            jedis.del(cacheKey);
        } catch (Exception e) {
            logger.warn("Cache invalidation error", e);
        }
    }
}
```

---

### Spring Boot with @Cacheable

```java
@Service
public class ProductService {
    
    @Autowired
    private ProductRepository repository;
    
    @Cacheable(value = "products", key = "#productId", unless = "#result == null")
    public Product getProduct(String productId) {
        // This method only called on cache miss
        return repository.findById(productId).orElse(null);
    }
    
    @CachePut(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        return repository.save(product);
    }
    
    @CacheEvict(value = "products", key = "#productId")
    public void deleteProduct(String productId) {
        repository.deleteById(productId);
    }
    
    @CacheEvict(value = "products", allEntries = true)
    public void clearCache() {
        // Invalidate all product cache entries
    }
}
```

**Configuration**:
```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(60))
            .serializeValuesWith(SerializationPair.fromSerializer(
                new GenericJackson2JsonRedisSerializer()));
        
        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .build();
    }
}
```

---

### Python Implementation

```python
import redis
import json
from typing import Optional

class CacheAsideService:
    def __init__(self, redis_client: redis.Redis, db_client):
        self.redis = redis_client
        self.db = db_client
    
    def get_product(self, product_id: str) -> Optional[dict]:
        cache_key = f"product:{product_id}"
        
        # 1. Try cache
        try:
            cached = self.redis.get(cache_key)
            if cached:
                return json.loads(cached)
        except redis.RedisError as e:
            print(f"Cache error: {e}")
        
        # 2. Cache miss - query database
        product = self.db.find_product(product_id)
        
        if not product:
            return None
        
        # 3. Store in cache
        try:
            self.redis.setex(
                cache_key,
                3600,  # 1 hour
                json.dumps(product)
            )
        except redis.RedisError as e:
            print(f"Cache write error: {e}")
        
        return product
    
    def update_product(self, product_id: str, product: dict):
        # Update database
        self.db.save_product(product)
        
        # Invalidate cache
        cache_key = f"product:{product_id}"
        try:
            self.redis.delete(cache_key)
        except redis.RedisError as e:
            print(f"Cache invalidation error: {e}")
```

---

### Node.js Implementation

```javascript
const redis = require('redis');

class CacheAsideService {
    constructor(redisClient, database) {
        this.redis = redisClient;
        this.db = database;
    }
    
    async getProduct(productId) {
        const cacheKey = `product:${productId}`;
        
        // 1. Try cache
        try {
            const cached = await this.redis.get(cacheKey);
            if (cached) {
                return JSON.parse(cached);
            }
        } catch (error) {
            console.warn('Cache read error:', error);
        }
        
        // 2. Query database
        const product = await this.db.findProduct(productId);
        
        if (!product) {
            return null;
        }
        
        // 3. Store in cache
        try {
            await this.redis.setEx(
                cacheKey,
                3600,  // 1 hour
                JSON.stringify(product)
            );
        } catch (error) {
            console.warn('Cache write error:', error);
        }
        
        return product;
    }
    
    async updateProduct(productId, product) {
        // Update database
        await this.db.saveProduct(product);
        
        // Invalidate cache
        const cacheKey = `product:${productId}`;
        try {
            await this.redis.del(cacheKey);
        } catch (error) {
            console.warn('Cache invalidation error:', error);
        }
    }
}
```

---

## Advantages

### 1. **Simple and Intuitive**
- Easy to understand and implement
- Clear control flow
- Explicit cache management

### 2. **Flexible**
- Application decides what to cache
- Can implement custom caching logic
- Easy to add business rules

### 3. **Resilient to Cache Failures**
```java
// Cache failure doesn't break the app
try {
    return cache.get(key);
} catch (Exception e) {
    // Fall back to database
    return database.get(key);
}
```

### 4. **Only Requested Data is Cached**
- Lazy loading = efficient memory usage
- Don't cache unused data
- Cache grows organically based on traffic

### 5. **Database Remains Source of Truth**
- Cache can be completely cleared without data loss
- Database consistency maintained
- Easy to rebuild cache

### 6. **Cache Miss Performance**
- Only one additional cache check
- No write-through overhead
- Fast for write-heavy workloads

---

## Disadvantages

### 1. **Cache Miss Penalty**
```
Cache hit:  1-2ms (cache latency)
Cache miss: 50-100ms (cache check + DB query + cache write)
```

Three operations on miss:
- Check cache (1ms)
- Query database (50ms)
- Write to cache (1ms)

### 2. **Initial Cold Start**
- Empty cache after restart/deployment
- All initial requests are cache misses
- Poor performance until cache warms up

### 3. **Potential Stale Data**
```
Time 0: User reads data (cached)
Time 1: User updates data (cache invalidated)
Time 2: External system updates DB directly
Time 3: User reads data (stale cached version if TTL not expired)
```

### 4. **Cache Stampede Risk**
```
Popular key expires at 10:00:00
100 concurrent requests at 10:00:01
All 100 requests:
1. Miss cache
2. Query database
3. Write to cache

Result: Database overwhelmed
```

### 5. **Consistency Challenges**
- Application must handle invalidation
- Race conditions between read and write
- Multiple instances can have different cache states

### 6. **Code Complexity**
- Cache logic mixed with business logic
- Error handling for cache operations
- More code paths to test

---

## Best Practices

### 1. **Always Set TTL**

```java
// BAD: No TTL
jedis.set(key, value);

// GOOD: Set TTL
jedis.setex(key, 3600, value);  // 1 hour

// BETTER: Add jitter to prevent thundering herd
int ttl = 3600 + random.nextInt(300);  // 3600-3900 seconds
jedis.setex(key, ttl, value);
```

### 2. **Handle Cache Failures Gracefully**

```java
public Product getProduct(String id) {
    try {
        String cached = redis.get(key);
        if (cached != null) return deserialize(cached);
    } catch (RedisConnectionException e) {
        logger.error("Cache unavailable, using database", e);
        metrics.incrementCacheError();
        // Fall through to database
    }
    
    Product product = database.findProduct(id);
    
    try {
        redis.setex(key, ttl, serialize(product));
    } catch (RedisConnectionException e) {
        logger.warn("Failed to update cache", e);
        // Don't fail the request
    }
    
    return product;
}
```

### 3. **Invalidate, Don't Update on Write**

```java
// RECOMMENDED: Invalidate
public void updateProduct(Product product) {
    database.save(product);
    cache.delete("product:" + product.getId());
}

// AVOID: Update cache
// Problem: Race condition between DB write and cache update
public void updateProduct(Product product) {
    database.save(product);
    cache.set("product:" + product.getId(), product);  // Risk of inconsistency
}
```

**Why?** Invalidation is safer:
- Avoids race conditions
- Next read will fetch latest from DB
- Simpler logic

### 4. **Use Consistent Key Naming**

```java
// Pattern: {namespace}:{entity}:{id}
String key = String.format("product:%s", productId);

// For collections/lists
String key = String.format("products:category:%s:page:%d", categoryId, page);

// With versioning
String key = String.format("user:%s:profile:v2", userId);
```

### 5. **Cache Null Results (with Short TTL)**

```java
public Product getProduct(String id) {
    String cached = redis.get(key);
    if (cached != null) {
        if (cached.equals("NULL")) return null;  // Cached miss
        return deserialize(cached);
    }
    
    Product product = database.findProduct(id);
    
    if (product == null) {
        // Cache the "not found" result to prevent repeated DB queries
        redis.setex(key, 60, "NULL");  // Short TTL for negative cache
        return null;
    }
    
    redis.setex(key, 3600, serialize(product));
    return product;
}
```

Prevents **cache penetration** attacks.

### 6. **Use Monitoring and Metrics**

```java
public Product getProduct(String id) {
    Timer.Context timer = metrics.timer("cache.get").time();
    
    try {
        String cached = redis.get(key);
        if (cached != null) {
            metrics.incrementCounter("cache.hit");
            return deserialize(cached);
        }
        
        metrics.incrementCounter("cache.miss");
        Product product = database.findProduct(id);
        
        if (product != null) {
            redis.setex(key, ttl, serialize(product));
        }
        
        return product;
    } finally {
        timer.stop();
    }
}
```

**Track**:
- Hit rate / miss rate
- Latency (cache vs database)
- Error rate
- Eviction rate

---

## Common Patterns and Variations

### 1. **Single-Flight Pattern (Prevent Cache Stampede)**

```java
private final Map<String, CompletableFuture<Product>> inFlightRequests = 
    new ConcurrentHashMap<>();

public Product getProduct(String id) {
    String cacheKey = "product:" + id;
    
    // Check cache
    String cached = redis.get(cacheKey);
    if (cached != null) return deserialize(cached);
    
    // Check if request is already in-flight
    CompletableFuture<Product> future = inFlightRequests.computeIfAbsent(id, key -> {
        return CompletableFuture.supplyAsync(() -> {
            try {
                // Only one thread executes this
                Product product = database.findProduct(id);
                if (product != null) {
                    redis.setex(cacheKey, 3600, serialize(product));
                }
                return product;
            } finally {
                inFlightRequests.remove(id);
            }
        });
    });
    
    return future.join();  // Wait for in-flight request
}
```

### 2. **Probabilistic Early Expiration**

Prevents stampede by refreshing before expiration:

```java
public Product getProduct(String id) {
    String cached = redis.get(key);
    if (cached != null) {
        long ttl = redis.ttl(key);
        
        // Probabilistically refresh before expiration
        // Higher probability as TTL decreases
        double refreshProbability = 1.0 - (ttl / (double) ORIGINAL_TTL);
        if (Math.random() < refreshProbability) {
            // Refresh in background
            executor.submit(() -> refreshCache(id));
        }
        
        return deserialize(cached);
    }
    
    // Cache miss - load normally
    return loadAndCache(id);
}
```

### 3. **Bulk/Batch Loading**

```java
public Map<String, Product> getProducts(List<String> ids) {
    // 1. Bulk read from cache
    List<String> keys = ids.stream()
        .map(id -> "product:" + id)
        .collect(Collectors.toList());
    
    List<String> cached = redis.mget(keys.toArray(new String[0]));
    
    // 2. Find cache misses
    Map<String, Product> results = new HashMap<>();
    List<String> missedIds = new ArrayList<>();
    
    for (int i = 0; i < ids.size(); i++) {
        if (cached.get(i) != null) {
            results.put(ids.get(i), deserialize(cached.get(i)));
        } else {
            missedIds.add(ids.get(i));
        }
    }
    
    // 3. Batch query database for misses
    if (!missedIds.isEmpty()) {
        List<Product> products = database.findProducts(missedIds);
        
        // 4. Bulk write to cache
        Map<String, String> toCache = new HashMap<>();
        for (Product p : products) {
            results.put(p.getId(), p);
            toCache.put("product:" + p.getId(), serialize(p));
        }
        
        if (!toCache.isEmpty()) {
            redis.mset(toCache);
        }
    }
    
    return results;
}
```

---

## Handling Failures

### Cache Failure Scenarios

#### 1. **Cache Read Failure**
```java
try {
    cached = redis.get(key);
} catch (JedisConnectionException e) {
    logger.error("Cache unavailable", e);
    cached = null;  // Treat as cache miss
}
```

#### 2. **Cache Write Failure**
```java
try {
    redis.setex(key, ttl, value);
} catch (JedisConnectionException e) {
    logger.warn("Failed to cache", e);
    // Continue - don't fail the request
}
```

#### 3. **Partial Cache Failure**
```java
public List<Product> getProducts(List<String> ids) {
    try {
        return getFromCache(ids);
    } catch (PartialFailureException e) {
        // Some keys retrieved, some failed
        return mergeWithDatabase(e.getPartialResults(), e.getFailedIds());
    }
}
```

### Circuit Breaker Pattern

```java
@Slf4j
public class CacheAsideWithCircuitBreaker {
    private final CircuitBreaker circuitBreaker;
    
    public Product getProduct(String id) {
        if (circuitBreaker.isOpen()) {
            // Cache unavailable, go directly to database
            return database.findProduct(id);
        }
        
        try {
            return getWithCache(id);
        } catch (RedisException e) {
            circuitBreaker.recordFailure();
            return database.findProduct(id);
        }
    }
}
```

---

## Performance Considerations

### Latency Breakdown

```
Cache Hit:
├─ Network to Redis: 1ms
├─ Redis lookup: 0.1ms
├─ Network from Redis: 1ms
└─ Deserialization: 0.5ms
Total: ~2.5ms

Cache Miss:
├─ Network to Redis: 1ms
├─ Redis lookup (miss): 0.1ms
├─ Network from Redis: 1ms
├─ Database query: 50ms
├─ Network to Redis: 1ms
├─ Redis write: 0.5ms
├─ Network from Redis: 1ms
├─ Serialization: 0.5ms
└─ Deserialization: 0.5ms
Total: ~56ms
```

### Optimization Strategies

**1. Reduce Serialization Overhead**
```java
// Use efficient serialization formats
Protocol Buffers < MessagePack < JSON < Java Serialization

// Compress large objects
byte[] compressed = compress(serialize(object));
```

**2. Use Connection Pooling**
```java
JedisPoolConfig config = new JedisPoolConfig();
config.setMaxTotal(128);
config.setMaxIdle(64);
config.setMinIdle(16);
config.setTestOnBorrow(true);
```

**3. Pipeline Multiple Operations**
```java
Pipeline pipeline = jedis.pipelined();
pipeline.get("key1");
pipeline.get("key2");
pipeline.get("key3");
List<Object> results = pipeline.syncAndReturnAll();
```

---

## Key Takeaways

1. **Cache-Aside is the most common caching pattern** — application explicitly manages cache
2. **Lazy loading** — data cached only when requested
3. **Resilient** — cache failures don't break application (fallback to database)
4. **Cache miss penalty** — requires cache check + DB query + cache write
5. **Always set TTL** — prevent infinite stale data
6. **Invalidate on write** — safer than updating cache
7. **Handle cache failures gracefully** — don't let cache outage break your app
8. **Monitor hit rates** — validate effectiveness of caching strategy

---

## Next Steps

- [Read-Through Pattern](./read-through.md)
- [Write-Through Pattern](./write-through.md)
- [Cache Invalidation Strategies](./cache-invalidation.md)
- [Handling Cache Stampede](./cache-stampede.md)

---

**Last Updated**: 2026-01-17
**Target Audience**: Senior Software Engineers
**Difficulty Level**: Intermediate
