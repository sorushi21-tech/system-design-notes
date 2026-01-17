# Write-Through Cache Pattern

## Table of Contents
1. [Introduction](#introduction)
2. [Pattern Overview](#pattern-overview)
3. [How It Works](#how-it-works)
4. [Implementation](#implementation)
5. [Advantages](#advantages)
6. [Disadvantages](#disadvantages)
7. [vs Other Patterns](#vs-other-patterns)
8. [Use Cases](#use-cases)
9. [Best Practices](#best-practices)

---

## Introduction

**Write-Through Cache** is a caching pattern where writes go through the cache to the database. Every write operation updates both the cache and the database synchronously, ensuring the cache is always consistent with the database.

---

## Pattern Overview

### Architecture

```
┌──────────────┐
│ Application  │
└──────┬───────┘
       │
       │ Write
       ▼
   ┌────────┐
   │ Cache  │ ← Write updates cache
   └────┬───┘
        │
        │ Synchronous write
        ▼
   ┌──────────┐
   │ Database │ ← Then updates database
   └──────────┘
```

### Key Characteristics
- **Synchronous writes**: Cache and database updated together
- **Consistency**: Cache always matches database
- **Write latency**: Higher (must wait for both updates)
- **Read-optimized**: Subsequent reads always hit cache

---

## How It Works

### Write Flow

```
1. Application issues write request
2. Update cache
3. Update database (synchronously)
4. Both succeed → Return success
5. Either fails → Rollback, return error
```

### Read Flow

```
1. Application issues read request
2. Check cache
3. If HIT: Return from cache (always fresh)
4. If MISS: 
   ├─ Query database
   ├─ Update cache
   └─ Return data
```

---

## Implementation

### Basic Implementation (Java)

```java
@Service
public class WriteThroughCacheService {
    @Autowired private JedisPool redisPool;
    @Autowired private ProductRepository repository;
    @Autowired private ObjectMapper objectMapper;
    
    private static final int TTL = 3600;  // 1 hour
    
    public Product getProduct(String productId) {
        String key = "product:" + productId;
        
        // 1. Try cache first
        try (Jedis jedis = redisPool.getResource()) {
            String cached = jedis.get(key);
            if (cached != null) {
                return objectMapper.readValue(cached, Product.class);
            }
        } catch (Exception e) {
            logger.warn("Cache read error", e);
        }
        
        // 2. Cache miss - query database
        Product product = repository.findById(productId).orElse(null);
        
        // 3. Update cache
        if (product != null) {
            try (Jedis jedis = redisPool.getResource()) {
                String json = objectMapper.writeValueAsString(product);
                jedis.setex(key, TTL, json);
            } catch (Exception e) {
                logger.warn("Cache write error", e);
            }
        }
        
        return product;
    }
    
    public Product updateProduct(Product product) {
        String key = "product:" + product.getId();
        
        try {
            // 1. Update cache first
            try (Jedis jedis = redisPool.getResource()) {
                String json = objectMapper.writeValueAsString(product);
                jedis.setex(key, TTL, json);
            }
            
            // 2. Update database
            Product saved = repository.save(product);
            
            return saved;
        } catch (Exception e) {
            // On error, invalidate cache to prevent inconsistency
            try (Jedis jedis = redisPool.getResource()) {
                jedis.del(key);
            } catch (Exception ex) {
                logger.error("Failed to invalidate cache after error", ex);
            }
            throw e;
        }
    }
    
    public void deleteProduct(String productId) {
        String key = "product:" + productId;
        
        // 1. Delete from cache
        try (Jedis jedis = redisPool.getResource()) {
            jedis.del(key);
        }
        
        // 2. Delete from database
        repository.deleteById(productId);
    }
}
```

---

### Transactional Write-Through

```java
@Service
public class TransactionalWriteThroughCache {
    
    @Transactional
    public Product updateProduct(Product product) {
        String key = "product:" + product.getId();
        
        // 1. Update database within transaction
        Product saved = repository.save(product);
        
        // 2. Update cache after successful DB commit
        // (Use @TransactionalEventListener for post-commit)
        return saved;
    }
    
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void updateCacheAfterCommit(ProductUpdatedEvent event) {
        String key = "product:" + event.getProductId();
        
        try (Jedis jedis = redisPool.getResource()) {
            Product product = repository.findById(event.getProductId())
                .orElse(null);
            
            if (product != null) {
                String json = objectMapper.writeValueAsString(product);
                jedis.setex(key, TTL, json);
            }
        } catch (Exception e) {
            logger.error("Failed to update cache after commit", e);
        }
    }
}
```

---

### Spring Cache with Write-Through

```java
@Service
public class ProductService {
    
    @Autowired private ProductRepository repository;
    
    @Cacheable(value = "products", key = "#productId")
    public Product getProduct(String productId) {
        return repository.findById(productId).orElse(null);
    }
    
    @CachePut(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        // @CachePut updates cache with return value
        return repository.save(product);
    }
    
    @CacheEvict(value = "products", key = "#productId")
    public void deleteProduct(String productId) {
        repository.deleteById(productId);
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
        RedisCacheConfiguration config = RedisCacheConfiguration
            .defaultCacheConfig()
            .entryTtl(Duration.ofHours(1));
        
        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .build();
    }
}
```

---

## Advantages

### 1. **Strong Consistency**

```java
// Cache always reflects database state
Product cached = cache.get("product:123");
Product fromDB = database.get("product:123");

assert cached.equals(fromDB);  // Always true
```

Cache and database never diverge.

### 2. **Simple Read Logic**

```java
// Reads are simple - cache is always fresh
public Product getProduct(String id) {
    Product cached = cache.get(id);
    if (cached != null) {
        return cached;  // Always up-to-date
    }
    
    Product fromDB = database.get(id);
    cache.set(id, fromDB);
    return fromDB;
}
```

### 3. **No Cache Invalidation Needed**

```java
// No need to invalidate on update - cache already updated
public void updateProduct(Product product) {
    cache.set(product.getId(), product);  // Already done
    database.save(product);
}
```

### 4. **Read Performance**

Subsequent reads after write are fast (cached).

### 5. **Durability**

Data written to both cache and database — survives cache failures.

---

## Disadvantages

### 1. **Write Latency**

```java
Write-Through:
├─ Update cache: 2ms
├─ Update database: 50ms
└─ Total: 52ms

vs

Cache-Aside (write):
├─ Update database: 50ms
├─ Invalidate cache: 1ms
└─ Total: 51ms (slightly faster)
```

Must wait for both cache and database updates.

### 2. **Write Amplification**

```java
// Every write updates both cache and database
for (int i = 0; i < 1000; i++) {
    updateProduct(product);  // 1000 cache writes + 1000 DB writes
}

// Even if data isn't read, it's cached
```

### 3. **Complexity**

```java
// Must handle partial failures
try {
    cache.update(key, value);
    database.update(key, value);
} catch (CacheException e) {
    // Cache updated, database failed - inconsistency!
    cache.delete(key);  // Rollback cache
    throw e;
} catch (DatabaseException e) {
    // Database updated, cache failed - also inconsistency!
    cache.delete(key);  // Invalidate cache
    throw e;
}
```

### 4. **Not Suitable for Write-Heavy Workloads**

```java
Write:Read ratio = 10:1

Result:
- 10 writes → 10 cache updates + 10 DB updates
- 1 read → Fast (cache hit)

But: Wasted effort caching rarely-read data
```

### 5. **Cache Becomes Bottleneck**

```java
// High write throughput can overwhelm cache
1000 writes/sec → Cache must handle:
- 1000 writes/sec to cache
- 1000 writes/sec to database
- All read traffic

Result: Cache becomes performance bottleneck
```

---

## vs Other Patterns

### Write-Through vs Cache-Aside

| Aspect | Write-Through | Cache-Aside |
|--------|---------------|-------------|
| **Write Path** | Cache → Database | Database → Invalidate Cache |
| **Consistency** | Always consistent | Eventually consistent |
| **Write Latency** | Higher (both updates) | Lower (DB only) |
| **Read Latency** | Lower (always cached) | Higher (cache misses) |
| **Complexity** | Higher | Lower |
| **Use Case** | Read-heavy, consistency critical | General purpose |

```java
// Write-Through
public void update(Product p) {
    cache.set(p.getId(), p);     // 1. Cache
    database.save(p);             // 2. Database
}

// Cache-Aside
public void update(Product p) {
    database.save(p);             // 1. Database
    cache.delete(p.getId());      // 2. Invalidate
}
```

---

### Write-Through vs Write-Behind

| Aspect | Write-Through | Write-Behind |
|--------|---------------|--------------|
| **Synchronous** | Yes | No |
| **Write Latency** | Higher | Lower |
| **Durability** | Strong | Weak (risk of data loss) |
| **Complexity** | Medium | High |
| **Database Load** | Higher | Lower (batching) |

```java
// Write-Through
public void update(Product p) {
    cache.set(p.getId(), p);
    database.save(p);  // Wait for DB
    return;  // Return after both complete
}

// Write-Behind
public void update(Product p) {
    cache.set(p.getId(), p);
    queue.add(p);  // Async DB write
    return;  // Return immediately
}
```

---

### Write-Through vs Read-Through

**Write-Through**: Writes go through cache
**Read-Through**: Reads go through cache

**Combined Pattern**: Both reads and writes managed by cache layer

```java
// Write-Through + Read-Through
public class ThroughCache {
    
    // Read-Through
    public Product get(String id) {
        return cache.get(id, key -> database.get(key));
    }
    
    // Write-Through
    public void update(Product p) {
        cache.set(p.getId(), p);
        database.save(p);
    }
}
```

---

## Use Cases

### ✅ Good Fit

**1. Read-Heavy Workloads**
```java
Ratio: Read:Write = 100:1

Benefit:
- Writes slow (acceptable)
- Reads fast (from cache)
- Net: Excellent performance
```

**2. Strong Consistency Requirements**
```java
Example: Financial data, inventory counts

Benefit: Cache always matches database
```

**3. Small, Frequently Accessed Data**
```java
Examples:
- User sessions
- User profiles
- Configuration data
- Product catalogs (read-heavy)
```

**4. When Cache Miss is Expensive**
```java
// Complex database query
Query: 5-table join + aggregation (500ms)

Write-Through: Cache always populated, miss rare
Cache-Aside: Cache misses expensive
```

---

### ❌ Poor Fit

**1. Write-Heavy Workloads**
```java
Ratio: Write:Read = 10:1

Problem: Every write updates cache unnecessarily
```

**2. Large Data Sets**
```java
Dataset: 1TB of data
Cache: 10GB capacity

Problem: Can't cache everything, wasted writes
```

**3. Time-Sensitive Data**
```java
Example: Stock prices, real-time sensor data

Problem: Data stale before next read
```

**4. High Write Latency Requirements**
```java
SLA: <10ms write latency

Write-Through: 50ms+ (cache + database)
Problem: Can't meet SLA
```

---

## Best Practices

### 1. **Handle Partial Failures**

```java
public Product updateProduct(Product product) {
    String key = "product:" + product.getId();
    boolean cacheUpdated = false;
    
    try {
        // 1. Update cache
        cache.set(key, product);
        cacheUpdated = true;
        
        // 2. Update database
        Product saved = repository.save(product);
        
        return saved;
    } catch (Exception e) {
        // Rollback cache if database fails
        if (cacheUpdated) {
            try {
                cache.delete(key);
            } catch (Exception ex) {
                logger.error("Failed to rollback cache", ex);
            }
        }
        throw e;
    }
}
```

### 2. **Use TTL as Safety Net**

```java
// Even with write-through, set TTL
cache.setex(key, 3600, value);  // 1 hour TTL

// Why? Protection against:
// - Cache inconsistency bugs
// - Missed invalidations
// - External database updates
```

### 3. **Implement Circuit Breaker**

```java
@Autowired private CircuitBreaker cacheCircuitBreaker;

public void updateProduct(Product product) {
    // Update database (always)
    repository.save(product);
    
    // Update cache (with circuit breaker)
    cacheCircuitBreaker.run(() -> {
        cache.set(product.getId(), product);
    }, () -> {
        // Fallback: Just log, don't fail request
        logger.warn("Cache update failed, skipping");
    });
}
```

### 4. **Monitor Write Performance**

```java
@Aspect
public class CacheMonitoringAspect {
    
    @Around("@annotation(CachePut)")
    public Object monitorCachePut(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        
        try {
            Object result = pjp.proceed();
            long duration = System.currentTimeMillis() - start;
            
            metrics.histogram("cache.write.duration").update(duration);
            
            if (duration > 100) {
                logger.warn("Slow cache write: {}ms", duration);
            }
            
            return result;
        } catch (Exception e) {
            metrics.counter("cache.write.error").inc();
            throw e;
        }
    }
}
```

### 5. **Batch Writes When Possible**

```java
public void updateProducts(List<Product> products) {
    // Batch database update
    repository.saveAll(products);
    
    // Batch cache update
    Map<String, String> cacheUpdates = products.stream()
        .collect(Collectors.toMap(
            p -> "product:" + p.getId(),
            p -> serialize(p)
        ));
    
    redis.mset(cacheUpdates);
}
```

### 6. **Consider Async Write-Through**

```java
public CompletableFuture<Product> updateProductAsync(Product product) {
    return CompletableFuture.supplyAsync(() -> {
        // Update cache
        cache.set(product.getId(), product);
        
        // Update database
        return repository.save(product);
    });
}
```

### 7. **Version/Timestamp Tracking**

```java
public class VersionedProduct {
    private String id;
    private String name;
    private long version;  // Incremented on each update
    
    // Or use timestamp
    private Instant lastModified;
}

public void updateProduct(Product product) {
    product.setVersion(product.getVersion() + 1);
    product.setLastModified(Instant.now());
    
    cache.set(product.getId(), product);
    repository.save(product);
}
```

---

## Key Takeaways

1. **Write-Through ensures cache consistency** — cache always matches database
2. **Higher write latency** — must update both cache and database
3. **Best for read-heavy workloads** — writes are slower but reads are fast
4. **Handles partial failures carefully** — rollback cache on database error
5. **Set TTL even with write-through** — safety net for bugs
6. **Not suitable for write-heavy** — wastes cache updates
7. **Combine with circuit breakers** — graceful degradation on cache failures

---

## Next Steps

- [Write-Behind Pattern](./write-behind.md)
- [Read-Through Pattern](./read-through.md)
- [Cache-Aside Pattern](./cache-aside.md)
- [Cache Invalidation Strategies](./cache-invalidation.md)

---

**Last Updated**: 2026-01-17
**Target Audience**: Senior Software Engineers
**Difficulty Level**: Intermediate
