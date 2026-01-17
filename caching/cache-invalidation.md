# Cache Invalidation Strategies

## Table of Contents
1. [Introduction](#introduction)
2. [Why Cache Invalidation is Hard](#why-cache-invalidation-is-hard)
3. [Invalidation Strategies](#invalidation-strategies)
4. [Time-Based Invalidation (TTL)](#time-based-invalidation-ttl)
5. [Event-Based Invalidation](#event-based-invalidation)
6. [Write-Through Invalidation](#write-through-invalidation)
7. [Purge/Delete on Update](#purge-delete-on-update)
8. [Tag-Based Invalidation](#tag-based-invalidation)
9. [Version-Based Invalidation](#version-based-invalidation)
10. [Distributed Invalidation](#distributed-invalidation)
11. [Best Practices](#best-practices)
12. [Common Pitfalls](#common-pitfalls)

---

## Introduction

> "There are only two hard things in Computer Science: cache invalidation and naming things."
> — Phil Karlton

Cache invalidation is the process of removing or updating stale data from the cache to maintain consistency with the source of truth (database). Getting it wrong leads to users seeing outdated information, which can have serious consequences in production systems.

---

## Why Cache Invalidation is Hard

### The Fundamental Challenge

```
Time T0: Data written to database
Time T1: Cache doesn't know about the change
Time T2: User reads stale data from cache
Time T3: User makes decision based on stale data
Time T4: Chaos ensues
```

### Common Problems

**1. Timing Issues**
```
Thread 1: UPDATE database SET price = 100
Thread 2: SELECT price FROM cache → Gets 90 (stale)
Thread 1: DELETE cache key
```
Result: Race condition leads to stale data

**2. Partial Failures**
```
✓ Database update succeeds
✗ Cache invalidation fails (network error)
```
Result: Cache contains stale data indefinitely

**3. Complex Dependencies**
```
Update User → Invalidate:
- user:{id}
- user:{id}:profile
- users:list:page:{X}
- feed:user:{id}
- ... 20 more cache keys
```
Missing even one = stale data

**4. Distributed Systems**
```
Instance 1: Updates database, invalidates local cache
Instance 2: Still has stale data in local cache
Instance 3: Still has stale data in local cache
```

---

## Invalidation Strategies

### Strategy Comparison

| Strategy | Complexity | Consistency | Use Case |
|----------|------------|-------------|----------|
| TTL | Low | Eventual | Most common, general purpose |
| Delete on Update | Low | Strong | Simple data, few dependencies |
| Write-Through | Medium | Strong | Write-heavy, consistency critical |
| Event-Based | High | Strong | Complex dependencies |
| Tag-Based | Medium | Strong | Related data groups |
| Version-Based | Low | Strong | Immutable data patterns |

---

## Time-Based Invalidation (TTL)

### Overview

The simplest and most common strategy: set an expiration time on cached data.

```redis
SET product:123 "{...data...}" EX 3600  # Expires in 1 hour
```

### How It Works

```
T=0:    Cache data with TTL=60s
T=30s:  Data still valid
T=60s:  Data expires automatically
T=61s:  Cache miss → Load from database
```

---

### Implementation

**Redis**:
```java
jedis.setex("user:1000", 3600, userData);  // 1 hour TTL
```

**Spring Cache**:
```java
@Configuration
public class CacheConfig {
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration
            .defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30));  // 30 minute TTL
        
        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .build();
    }
}
```

**Caffeine (Local Cache)**:
```java
Cache<String, Product> cache = Caffeine.newBuilder()
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .build();
```

---

### TTL Best Practices

#### 1. **Choose Appropriate TTL Based on Data Volatility**

```java
// Highly volatile data
cache.setex("stock:AAPL:price", 5, price);  // 5 seconds

// Moderate volatility
cache.setex("product:details", 1800, product);  // 30 minutes

// Rarely changes
cache.setex("country:list", 86400, countries);  // 24 hours

// Static data
cache.setex("config:feature-flags", 3600, config);  // 1 hour
```

#### 2. **Add Jitter to Prevent Thundering Herd**

```java
int baseTTL = 3600;  // 1 hour
int jitter = random.nextInt(300);  // 0-5 minutes
int finalTTL = baseTTL + jitter;

jedis.setex(key, finalTTL, value);
```

**Why?** If 1000 keys expire at exactly the same time, all requests hit database simultaneously.

#### 3. **Different TTLs for Different Cache Layers**

```java
// L1 (Local) - Short TTL for consistency
localCache.put(key, value, 5, TimeUnit.MINUTES);

// L2 (Distributed) - Longer TTL
redis.setex(key, 1800, value);  // 30 minutes
```

---

### Advantages

✅ **Simple**: No complex invalidation logic
✅ **Automatic**: No manual invalidation needed
✅ **Safe**: Stale data eventually expires
✅ **Works for all data**: Even when updates unknown

### Disadvantages

❌ **Eventual consistency**: Data can be stale until TTL expires
❌ **Trade-off**: Short TTL = more DB load, Long TTL = stale data longer
❌ **Unpredictable**: Don't know exactly when data becomes fresh

---

## Event-Based Invalidation

### Overview

Invalidate cache when specific events occur (e.g., data update, delete).

```java
public void updateProduct(Product product) {
    // 1. Update database
    repository.save(product);
    
    // 2. Trigger invalidation event
    eventPublisher.publish(new ProductUpdatedEvent(product.getId()));
}

@EventListener
public void onProductUpdated(ProductUpdatedEvent event) {
    // 3. Invalidate affected caches
    cache.delete("product:" + event.getProductId());
    cache.delete("products:category:" + event.getCategoryId());
    cache.delete("products:search:*");  // Invalidate search results
}
```

---

### Implementation with Spring Events

```java
// Event class
public class ProductUpdatedEvent {
    private final String productId;
    private final String categoryId;
    
    // Constructor, getters
}

// Publisher
@Service
public class ProductService {
    @Autowired private ApplicationEventPublisher eventPublisher;
    @Autowired private ProductRepository repository;
    
    public void updateProduct(Product product) {
        repository.save(product);
        eventPublisher.publishEvent(
            new ProductUpdatedEvent(product.getId(), product.getCategoryId())
        );
    }
}

// Listener
@Component
public class CacheInvalidationListener {
    @Autowired private CacheManager cacheManager;
    
    @EventListener
    @Async
    public void handleProductUpdate(ProductUpdatedEvent event) {
        // Invalidate all affected cache entries
        cacheManager.getCache("products")
            .evict("product:" + event.getProductId());
        
        cacheManager.getCache("products")
            .evict("category:" + event.getCategoryId());
        
        // Invalidate search results
        cacheManager.getCache("search").clear();
    }
}
```

---

### Advantages

✅ **Strong consistency**: Cache invalidated immediately on change
✅ **Efficient**: Only invalidate when data changes
✅ **Targeted**: Can invalidate related caches

### Disadvantages

❌ **Complex**: Must track all dependencies
❌ **Error-prone**: Easy to miss invalidations
❌ **Tight coupling**: Application must know cache structure

---

## Purge/Delete on Update

### Overview

Explicitly delete cache entries when data is updated.

```java
public void updateUser(User user) {
    // 1. Update database
    userRepository.save(user);
    
    // 2. Delete from cache
    cache.delete("user:" + user.getId());
    cache.delete("user:" + user.getId() + ":profile");
    cache.delete("users:list");
}
```

---

### Delete vs Update Cache

**Delete (Recommended)**:
```java
public void updateProduct(Product product) {
    database.save(product);
    cache.delete("product:" + product.getId());  // Next read will fetch fresh
}
```

**Update Cache (Risky)**:
```java
public void updateProduct(Product product) {
    database.save(product);
    cache.set("product:" + product.getId(), product);  // Race condition risk
}
```

**Why delete is safer?**
```
Thread 1: Read product (price=90)
Thread 2: Update database (price=100)
Thread 2: Update cache (price=100)
Thread 1: Cache stale read (price=90)  ← Overwrites with stale data!
```

---

### Pattern: Delete-After-Write

```java
@Transactional
public void updateProduct(Product product) {
    try {
        // 1. Update database (in transaction)
        productRepository.save(product);
        
        // 2. Invalidate cache after successful commit
    } catch (Exception e) {
        // Database update failed, don't invalidate cache
        throw e;
    }
    
    // 3. Invalidate after transaction commits
    invalidateProductCache(product.getId());
}

private void invalidateProductCache(String productId) {
    try {
        cache.delete("product:" + productId);
    } catch (Exception e) {
        logger.error("Cache invalidation failed", e);
        // Log but don't fail the request
    }
}
```

---

### Advantages

✅ **Simple to implement**
✅ **No stale data** (if done correctly)
✅ **Immediate consistency**

### Disadvantages

❌ **Must track all related keys**
❌ **Risk of forgetting invalidation**
❌ **Cache miss storm** (if many keys invalidated)

---

## Tag-Based Invalidation

### Overview

Group related cache entries with tags, invalidate by tag.

```java
// Store with tags
cache.setWithTags("product:123", product, 
    Tags.of("product", "category:electronics", "brand:apple"));

cache.setWithTags("product:456", product, 
    Tags.of("product", "category:electronics", "brand:samsung"));

// Invalidate all products in category
cache.invalidateByTag("category:electronics");
```

---

### Implementation with Redis

**Using Sets to Track Tags**:
```java
public void cacheWithTags(String key, Object value, Set<String> tags) {
    // 1. Store the actual data
    redis.setex(key, 3600, serialize(value));
    
    // 2. Add key to each tag set
    for (String tag : tags) {
        redis.sadd("tag:" + tag, key);
        redis.expire("tag:" + tag, 3600);
    }
}

public void invalidateByTag(String tag) {
    // 1. Get all keys with this tag
    Set<String> keys = redis.smembers("tag:" + tag);
    
    // 2. Delete all keys
    if (!keys.isEmpty()) {
        redis.del(keys.toArray(new String[0]));
    }
    
    // 3. Delete the tag set itself
    redis.del("tag:" + tag);
}
```

**Usage**:
```java
// Cache product
Set<String> tags = Set.of(
    "product",
    "category:" + product.getCategoryId(),
    "brand:" + product.getBrand()
);
cacheWithTags("product:" + product.getId(), product, tags);

// Update category → Invalidate all products in category
invalidateByTag("category:" + categoryId);
```

---

### Advantages

✅ **Batch invalidation**: Invalidate related items together
✅ **Flexible**: Dynamic tag assignment
✅ **Maintainable**: Clear relationships

### Disadvantages

❌ **Additional complexity**: Tag tracking overhead
❌ **Memory overhead**: Store tag-to-key mappings
❌ **Potential over-invalidation**: May invalidate more than needed

---

## Version-Based Invalidation

### Overview

Include version number in cache key. New version = new key.

```java
// Version in key
String key = "product:" + productId + ":v" + version;
cache.set(key, product);

// On update, increment version
version++;
String newKey = "product:" + productId + ":v" + version;
cache.set(newKey, updatedProduct);

// Old version naturally expires via TTL
```

---

### Implementation

**Simple Version Counter**:
```java
public class VersionedCache {
    private final AtomicLong version = new AtomicLong(0);
    
    public Product getProduct(String productId) {
        long currentVersion = version.get();
        String key = String.format("product:%s:v%d", productId, currentVersion);
        
        Product cached = cache.get(key);
        if (cached != null) return cached;
        
        Product product = database.findProduct(productId);
        cache.setex(key, 3600, product);
        return product;
    }
    
    public void updateProduct(Product product) {
        database.save(product);
        version.incrementAndGet();  // Increment version, old keys ignored
    }
}
```

**ETags/Content Hashing**:
```java
String contentHash = md5(serialize(product));
String key = "product:" + productId + ":" + contentHash;
```

---

### Advantages

✅ **No invalidation needed**: Old versions ignored
✅ **Simple**: Just increment version
✅ **Immutable keys**: No race conditions

### Disadvantages

❌ **Memory waste**: Old versions take space until TTL
❌ **Requires version tracking**: Additional state
❌ **Not suitable for all data**: Works best with versioned entities

---

## Distributed Invalidation

### Problem

In multi-instance deployments, invalidation must propagate to all instances.

```
Instance 1: Updates data, invalidates local cache
Instance 2: Still has stale data ← Problem!
Instance 3: Still has stale data ← Problem!
```

---

### Solution 1: Redis Pub/Sub

```java
// Publisher (any instance)
public void invalidateCache(String key) {
    // 1. Invalidate local cache
    localCache.invalidate(key);
    
    // 2. Invalidate Redis
    redis.del(key);
    
    // 3. Notify other instances
    redis.publish("cache:invalidate", key);
}

// Subscriber (all instances)
public class CacheInvalidationSubscriber extends JedisPubSub {
    @Override
    public void onMessage(String channel, String message) {
        if ("cache:invalidate".equals(channel)) {
            // Invalidate local cache on this instance
            localCache.invalidate(message);
        }
    }
}

// Setup
jedis.subscribe(new CacheInvalidationSubscriber(), "cache:invalidate");
```

---

### Solution 2: Message Queue (Kafka, RabbitMQ)

```java
// Producer
public void updateProduct(Product product) {
    database.save(product);
    
    CacheInvalidationEvent event = new CacheInvalidationEvent(
        "product:" + product.getId()
    );
    kafka.send("cache-invalidation", event);
}

// Consumer (all instances)
@KafkaListener(topics = "cache-invalidation")
public void handleInvalidation(CacheInvalidationEvent event) {
    localCache.invalidate(event.getKey());
    redis.del(event.getKey());
}
```

---

### Solution 3: Database Triggers + Change Data Capture (CDC)

```sql
-- Database trigger
CREATE TRIGGER product_updated
AFTER UPDATE ON products
FOR EACH ROW
BEGIN
    INSERT INTO cache_invalidation_queue (entity_type, entity_id)
    VALUES ('product', NEW.id);
END;
```

```java
// CDC consumer (Debezium, etc.)
@ChangeDataListener
public void onProductChanged(ProductChangeEvent event) {
    String key = "product:" + event.getProductId();
    
    // Invalidate on all instances
    cache.invalidate(key);
    redis.publish("cache:invalidate", key);
}
```

---

## Best Practices

### 1. **Defense in Depth: Combine Strategies**

```java
public void cacheProduct(Product product) {
    // TTL as fallback
    int ttl = 3600;
    
    // Event-based invalidation as primary
    eventPublisher.publish(new ProductCachedEvent(product.getId()));
    
    redis.setex("product:" + product.getId(), ttl, serialize(product));
}
```

### 2. **Idempotent Invalidation**

```java
// Safe to call multiple times
public void invalidate(String key) {
    cache.delete(key);  // Deleting non-existent key is OK
}
```

### 3. **Async Invalidation (When Possible)**

```java
@Async
public void invalidateCacheAsync(String key) {
    cache.delete(key);
}
```

Don't block user request for cache invalidation.

### 4. **Log Invalidation Events**

```java
public void invalidate(String key) {
    logger.info("Invalidating cache key: {}", key);
    cache.delete(key);
    metrics.incrementCounter("cache.invalidation", "key", key);
}
```

### 5. **Graceful Degradation**

```java
public void updateProduct(Product product) {
    // 1. Critical: Update database
    database.save(product);
    
    // 2. Best effort: Invalidate cache
    try {
        cache.delete("product:" + product.getId());
    } catch (Exception e) {
        logger.error("Cache invalidation failed", e);
        // Don't fail the update
    }
}
```

### 6. **Conditional Invalidation**

```java
public void updateProduct(Product product, Product oldProduct) {
    database.save(product);
    
    // Only invalidate if relevant fields changed
    if (!product.getName().equals(oldProduct.getName()) ||
        !product.getPrice().equals(oldProduct.getPrice())) {
        cache.delete("product:" + product.getId());
    }
}
```

### 7. **Bulkhead Pattern**

```java
// Separate thread pool for cache operations
ExecutorService cacheExecutor = Executors.newFixedThreadPool(10);

public void invalidateAsync(String key) {
    cacheExecutor.submit(() -> {
        try {
            cache.delete(key);
        } catch (Exception e) {
            logger.error("Cache invalidation failed", e);
        }
    });
}
```

---

## Common Pitfalls

### 1. **Forgetting to Invalidate Related Keys**

```java
// BAD: Only invalidate main key
cache.delete("product:" + productId);

// GOOD: Invalidate all related
cache.delete("product:" + productId);
cache.delete("product:" + productId + ":details");
cache.delete("products:category:" + categoryId);
cache.delete("products:search:*");
```

### 2. **Race Conditions**

```java
// BAD: Update cache directly
database.update(product);
cache.set(key, product);  // Race condition!

// GOOD: Delete and let next read populate
database.update(product);
cache.delete(key);
```

### 3. **Not Handling Invalidation Failures**

```java
// BAD: Fail entire operation
database.save(product);
cache.delete(key);  // If this fails, operation fails

// GOOD: Continue even if invalidation fails
database.save(product);
try {
    cache.delete(key);
} catch (Exception e) {
    logger.error("Invalidation failed", e);
}
```

### 4. **Over-Invalidation**

```java
// BAD: Invalidate entire cache on any change
cache.clear();

// GOOD: Targeted invalidation
cache.delete("product:" + productId);
```

### 5. **No TTL Fallback**

```java
// BAD: Only event-based invalidation
cache.set(key, value);  // No TTL

// GOOD: TTL as safety net
cache.setex(key, 3600, value);  // Expires even if invalidation fails
```

---

## Invalidation Decision Tree

```
Q: How often does data change?
├─ Rarely (days/weeks)
│  └─ Use long TTL (hours to days)
├─ Occasionally (hours)
│  └─ Use medium TTL (30-60 minutes)
└─ Frequently (minutes)
   ├─ Q: Can you detect changes?
   │  ├─ Yes → Event-based invalidation + short TTL
   │  └─ No → Very short TTL (1-5 minutes)
   └─ Q: Is consistency critical?
      ├─ Yes → Don't cache OR use write-through
      └─ No → Event-based + short TTL fallback
```

---

## Key Takeaways

1. **Cache invalidation is hard** — plan carefully
2. **Combine strategies** — TTL + event-based for safety
3. **Delete, don't update** — safer from race conditions
4. **Distributed systems need distributed invalidation** — use Pub/Sub or messaging
5. **Always set TTL** — even with event-based invalidation
6. **Log and monitor** — track invalidation events
7. **Graceful degradation** — don't fail requests due to cache issues
8. **Test failure scenarios** — what if invalidation fails?

---

## Next Steps

- [Cache Eviction Policies](./cache-eviction-policies.md)
- [Cache Stampede Prevention](./cache-stampede.md)
- [Write-Through Pattern](./write-through.md)

---

**Last Updated**: 2026-01-17
**Target Audience**: Senior Software Engineers
**Difficulty Level**: Advanced
