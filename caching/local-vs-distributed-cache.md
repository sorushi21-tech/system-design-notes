# Local vs Distributed Cache

## Table of Contents
1. [Introduction](#introduction)
2. [Local Cache (L1 Cache)](#local-cache-l1-cache)
3. [Distributed Cache (L2 Cache)](#distributed-cache-l2-cache)
4. [Comparison Matrix](#comparison-matrix)
5. [When to Use Which](#when-to-use-which)
6. [Multi-Level Caching (L1 + L2)](#multi-level-caching-l1--l2)
7. [Implementation Examples](#implementation-examples)
8. [Common Pitfalls](#common-pitfalls)
9. [Best Practices](#best-practices)

---

## Introduction

In system design, choosing between local and distributed caching — or combining both — is a fundamental architectural decision that impacts performance, consistency, scalability, and complexity.

**Key Question**: Should cache data be stored locally in each application instance, or in a shared external cache server?

The answer depends on your consistency requirements, traffic patterns, deployment architecture, and performance goals.

---

## Local Cache (L1 Cache)

### What is Local Cache?

**Local cache** (also called **in-process cache** or **L1 cache**) stores data in the application's own memory space, within the same JVM/process/container.

```
┌─────────────────────────┐
│   Application Instance  │
│  ┌─────────────────┐    │
│  │  Local Cache    │    │
│  │  (In-Memory)    │    │
│  └─────────────────┘    │
│         │               │
│  ┌──────▼──────┐        │
│  │ Application │        │
│  │    Logic    │        │
│  └─────────────┘        │
└─────────────────────────┘
```

### Characteristics

**Storage Location**: Application's heap memory
**Scope**: Single application instance
**Access Speed**: Nanoseconds (direct memory access)
**Network Overhead**: None
**Consistency**: Per-instance (not shared)

---

### Advantages

#### 1. **Ultra-Low Latency**
- Direct memory access (no network calls)
- Typically <100 nanoseconds
- 10-100x faster than distributed cache

#### 2. **No Network Overhead**
- Zero network latency
- No serialization/deserialization
- No connection pool management

#### 3. **High Throughput**
- Limited only by CPU and memory
- Can serve millions of requests per second
- No external dependency bottleneck

#### 4. **Simple Implementation**
```java
// Simple local cache with Guava
LoadingCache<String, User> cache = CacheBuilder.newBuilder()
    .maximumSize(10000)
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .build(new CacheLoader<String, User>() {
        public User load(String userId) {
            return database.findUser(userId);
        }
    });

User user = cache.get("user-123");
```

#### 5. **No External Dependencies**
- Works offline
- No Redis/Memcached to manage
- Simpler deployment

#### 6. **Cost-Effective**
- Uses existing application memory
- No additional infrastructure costs

---

### Disadvantages

#### 1. **Memory Constraints**
- Limited to JVM heap size
- Competes with application memory
- Risk of OutOfMemoryError

#### 2. **Inconsistency Across Instances**
```
Instance 1: user:1000 → {name: "Alice"}
Instance 2: user:1000 → {name: "Bob"}  ← Stale data!
```

Each instance has its own cache, can become stale independently.

#### 3. **Cache Warm-Up Per Instance**
- Each instance starts with cold cache
- Deployment/restart = empty cache
- Initial latency spike

#### 4. **Inefficient for Large Datasets**
- Data duplicated across instances
- Memory waste (N instances × cache size)

#### 5. **No Sharing Between Instances**
- Instance 1 caches data, Instance 2 misses
- Load not balanced across cache

#### 6. **Difficult Invalidation**
```
Problem: User updates profile
Instance 1: Invalidates cache ✓
Instance 2: Still has stale data ✗
```

---

### Popular Local Cache Libraries

#### Java
- **Caffeine** ✅ Recommended (high performance, modern API)
- **Guava Cache** (widely used, mature)
- **Ehcache** (enterprise features)
- **ConcurrentHashMap** (manual implementation)

#### .NET
- **MemoryCache** (built-in)
- **LazyCache** (wrapper with async support)

#### Node.js
- **node-cache**
- **lru-cache**

#### Python
- **functools.lru_cache** (built-in)
- **cachetools**

---

### Local Cache Implementation Example (Java)

```java
import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import java.util.concurrent.TimeUnit;

public class LocalCacheService {
    private final Cache<String, Product> productCache;
    
    public LocalCacheService() {
        this.productCache = Caffeine.newBuilder()
            .maximumSize(10_000)                        // Max 10K entries
            .expireAfterWrite(30, TimeUnit.MINUTES)     // TTL: 30 min
            .expireAfterAccess(15, TimeUnit.MINUTES)    // Idle timeout
            .recordStats()                               // Enable metrics
            .build();
    }
    
    public Product getProduct(String productId) {
        return productCache.get(productId, id -> {
            // Called on cache miss
            return database.loadProduct(id);
        });
    }
    
    public void invalidate(String productId) {
        productCache.invalidate(productId);
    }
    
    public CacheStats getStats() {
        return productCache.stats();
    }
}
```

---

## Distributed Cache (L2 Cache)

### What is Distributed Cache?

**Distributed cache** (L2 cache) is a shared caching layer external to application instances, typically Redis or Memcached.

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Instance 1 │  │  Instance 2 │  │  Instance 3 │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │
       └────────────────┼────────────────┘
                        │ Network
                ┌───────▼────────┐
                │ Distributed    │
                │    Cache       │
                │  (Redis)       │
                └────────────────┘
```

### Characteristics

**Storage Location**: External cache server (Redis, Memcached)
**Scope**: Shared across all instances
**Access Speed**: 1-5 milliseconds (network call)
**Network Overhead**: Serialization + network latency
**Consistency**: Shared state

---

### Advantages

#### 1. **Shared Across Instances**
- All instances see same cached data
- Efficient memory usage (single copy)
- One instance caches = all benefit

#### 2. **Consistency**
- Single source of truth
- Invalidation affects all instances immediately
- Reduces stale data issues

#### 3. **Scalable Storage**
- Not limited by application memory
- Can cache GBs or TBs of data
- Separate scaling of cache and application

#### 4. **Persistence Options**
- Redis supports RDB/AOF persistence
- Survive application restarts
- No cold start problem

#### 5. **Advanced Features**
- Complex data structures (Redis)
- Pub/Sub messaging
- Distributed locking
- Atomic operations across instances

#### 6. **Centralized Management**
- Monitor from single place
- Invalidation strategies easier
- Metrics and observability

---

### Disadvantages

#### 1. **Network Latency**
- 1-5ms per request (vs nanoseconds for local)
- Serialization/deserialization overhead
- Network can become bottleneck

#### 2. **Additional Infrastructure**
- Need to deploy and manage Redis/Memcached
- Operational complexity
- Additional costs

#### 3. **Single Point of Failure**
- If cache server down, all instances affected
- Requires high availability setup (Sentinel, Cluster)

#### 4. **Network Bandwidth**
- Large values consume network bandwidth
- Can saturate network with high traffic

#### 5. **Serialization Overhead**
```java
Object → Serialize → Network → Deserialize → Object
         ~100μs                    ~100μs
```

#### 6. **Connection Management**
- Connection pooling required
- Connection limits
- Connection failures to handle

---

### Distributed Cache Implementation Example (Java + Redis)

```java
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.Jedis;
import com.fasterxml.jackson.databind.ObjectMapper;

public class DistributedCacheService {
    private final JedisPool jedisPool;
    private final ObjectMapper objectMapper;
    
    public DistributedCacheService(JedisPool pool) {
        this.jedisPool = pool;
        this.objectMapper = new ObjectMapper();
    }
    
    public Product getProduct(String productId) {
        String key = "product:" + productId;
        
        try (Jedis jedis = jedisPool.getResource()) {
            String cached = jedis.get(key);
            
            if (cached != null) {
                return objectMapper.readValue(cached, Product.class);
            }
            
            Product product = database.loadProduct(productId);
            
            if (product != null) {
                String json = objectMapper.writeValueAsString(product);
                jedis.setex(key, 3600, json);  // 1 hour TTL
            }
            
            return product;
        } catch (Exception e) {
            // Fallback to database on cache failure
            logger.error("Cache error, falling back to DB", e);
            return database.loadProduct(productId);
        }
    }
    
    public void invalidate(String productId) {
        try (Jedis jedis = jedisPool.getResource()) {
            jedis.del("product:" + productId);
        }
    }
}
```

---

## Comparison Matrix

| Aspect | Local Cache | Distributed Cache |
|--------|-------------|-------------------|
| **Latency** | <100ns | 1-5ms |
| **Throughput** | Millions/sec | 100K-1M/sec |
| **Network** | None | Required |
| **Consistency** | Per-instance | Shared |
| **Memory** | Limited (heap) | Scalable (GBs) |
| **Complexity** | Low | Medium-High |
| **Cost** | Free (uses app memory) | Infrastructure cost |
| **Invalidation** | Per-instance | Global |
| **Cold Start** | Every restart | Persistent |
| **Data Sharing** | No | Yes |
| **Best For** | Hot, stable data | Shared state, large datasets |

---

## When to Use Which

### Use Local Cache When:

✅ **Read-heavy, rarely changing data**
- Configuration settings
- Reference data (country codes, currencies)
- Feature flags

✅ **Very high read frequency**
- 1000s of reads per second
- Sub-millisecond latency required

✅ **Small dataset**
- Can fit in application memory (MBs, not GBs)
- Won't cause memory pressure

✅ **Tolerate some inconsistency**
- Eventual consistency acceptable
- Stale data for short period OK

✅ **Minimal infrastructure**
- Want to avoid external dependencies
- Simple deployment

**Examples**:
- Product catalog (updated daily)
- User roles/permissions
- Geolocation data
- Currency exchange rates
- Translation strings

---

### Use Distributed Cache When:

✅ **Need consistency across instances**
- Shopping cart
- User sessions
- Inventory counts

✅ **Large dataset**
- Doesn't fit in single instance memory
- GBs of cached data

✅ **Shared state**
- Data accessed by multiple services
- Microservices architecture

✅ **Frequent updates**
- Data changes often
- Invalidation must affect all instances

✅ **Persistence required**
- Survive application restarts
- Avoid cold start issues

✅ **Complex data structures needed**
- Sorted sets, pub/sub, atomic operations
- Redis-specific features

**Examples**:
- User sessions (distributed)
- Rate limiting counters
- Real-time leaderboards
- Shopping carts
- API response caching
- User authentication tokens

---

## Multi-Level Caching (L1 + L2)

### The Best of Both Worlds

Combine local cache (L1) with distributed cache (L2) for optimal performance.

```
Request Flow:
1. Check Local Cache (L1)     ← 100ns
   ↓ Miss
2. Check Distributed Cache (L2) ← 2ms
   ↓ Miss
3. Query Database               ← 50ms
4. Store in L2
5. Store in L1
6. Return
```

### Architecture

```
┌─────────────────────────────┐
│      Application Instance   │
│  ┌───────────────────────┐  │
│  │ L1: Local Cache       │  │ ← Fastest
│  │ (Caffeine/Guava)      │  │
│  └───────────┬───────────┘  │
│              ↓ Miss          │
└──────────────┼──────────────┘
               │
        ┌──────▼──────┐
        │ L2: Redis   │ ← Shared, Consistent
        └──────┬──────┘
               ↓ Miss
        ┌──────▼──────┐
        │  Database   │ ← Source of Truth
        └─────────────┘
```

---

### Benefits of Multi-Level Caching

1. **Performance**: L1 serves hot data at nanosecond speed
2. **Consistency**: L2 provides shared state
3. **Scalability**: L2 handles large datasets
4. **Resilience**: Layers provide fallback
5. **Cost Efficiency**: Reduce L2 traffic with L1

---

### Implementation Strategy

```java
public class MultiLevelCacheService {
    private final Cache<String, Product> localCache;  // L1
    private final JedisPool redisPool;                // L2
    
    public Product getProduct(String productId) {
        String key = "product:" + productId;
        
        // 1. Check L1 (local cache)
        Product product = localCache.getIfPresent(key);
        if (product != null) {
            return product;  // L1 hit
        }
        
        // 2. Check L2 (Redis)
        try (Jedis jedis = redisPool.getResource()) {
            String cached = jedis.get(key);
            if (cached != null) {
                product = deserialize(cached);
                localCache.put(key, product);  // Populate L1
                return product;  // L2 hit
            }
        }
        
        // 3. Database query
        product = database.loadProduct(productId);
        
        if (product != null) {
            // Populate both caches
            try (Jedis jedis = redisPool.getResource()) {
                jedis.setex(key, 3600, serialize(product));  // L2
            }
            localCache.put(key, product);  // L1
        }
        
        return product;
    }
    
    public void invalidate(String productId) {
        String key = "product:" + productId;
        localCache.invalidate(key);  // Invalidate L1
        
        try (Jedis jedis = redisPool.getResource()) {
            jedis.del(key);  // Invalidate L2
            
            // Publish invalidation event for other instances
            jedis.publish("cache:invalidate", key);
        }
    }
}
```

---

### L1 + L2 Invalidation with Pub/Sub

**Challenge**: When one instance invalidates, others must clear their L1 cache.

**Solution**: Use Redis Pub/Sub

```java
public class CacheInvalidationListener {
    private final Cache<String, Object> localCache;
    private final JedisPool redisPool;
    
    public void startListening() {
        new Thread(() -> {
            try (Jedis jedis = redisPool.getResource()) {
                jedis.subscribe(new JedisPubSub() {
                    @Override
                    public void onMessage(String channel, String message) {
                        // Invalidate local cache when message received
                        localCache.invalidate(message);
                    }
                }, "cache:invalidate");
            }
        }).start();
    }
    
    public void invalidate(String key) {
        // 1. Invalidate local
        localCache.invalidate(key);
        
        // 2. Invalidate Redis
        try (Jedis jedis = redisPool.getResource()) {
            jedis.del(key);
            
            // 3. Notify other instances
            jedis.publish("cache:invalidate", key);
        }
    }
}
```

---

### Configuration Guidelines

**L1 Cache (Local)**:
- Small size (1K-100K entries)
- Short TTL (1-10 minutes)
- Very hot data only

**L2 Cache (Distributed)**:
- Larger size (100K-millions of entries)
- Longer TTL (10 minutes to hours)
- Shared data

**Example**:
```
L1: 10K entries, 5-minute TTL
L2: 1M entries, 1-hour TTL
```

---

## Implementation Examples

### Spring Boot with Caffeine (Local) + Redis (Distributed)

```java
@Configuration
public class CacheConfig {
    
    @Bean
    public Cache<String, Object> localCache() {
        return Caffeine.newBuilder()
            .maximumSize(10000)
            .expireAfterWrite(5, TimeUnit.MINUTES)
            .recordStats()
            .build();
    }
    
    @Bean
    public JedisPool redisPool() {
        JedisPoolConfig config = new JedisPoolConfig();
        config.setMaxTotal(128);
        return new JedisPool(config, "localhost", 6379);
    }
}

@Service
public class ProductService {
    @Autowired private Cache<String, Product> localCache;
    @Autowired private JedisPool redisPool;
    @Autowired private ProductRepository repository;
    @Autowired private ObjectMapper objectMapper;
    
    public Product getProduct(String id) {
        // L1 hit
        Product product = localCache.getIfPresent(id);
        if (product != null) return product;
        
        // L2 hit
        try (Jedis jedis = redisPool.getResource()) {
            String json = jedis.get("product:" + id);
            if (json != null) {
                product = objectMapper.readValue(json, Product.class);
                localCache.put(id, product);
                return product;
            }
        } catch (Exception e) {
            logger.warn("Redis error", e);
        }
        
        // Database
        product = repository.findById(id).orElse(null);
        if (product != null) {
            localCache.put(id, product);
            try (Jedis jedis = redisPool.getResource()) {
                jedis.setex("product:" + id, 3600, 
                    objectMapper.writeValueAsString(product));
            } catch (Exception e) {
                logger.warn("Redis error", e);
            }
        }
        
        return product;
    }
}
```

---

## Common Pitfalls

### 1. Over-Caching with Local Cache
**Problem**: Caching too much data locally causes OutOfMemoryError

**Solution**:
- Set reasonable size limits
- Monitor heap usage
- Use bounded caches (Caffeine with `maximumSize`)

---

### 2. Inconsistent L1 Caches
**Problem**: Different instances have different cached values

**Solution**:
- Short TTLs on L1
- Pub/Sub invalidation
- Accept eventual consistency

---

### 3. Not Handling Distributed Cache Failures
**Problem**: Redis down = all requests fail

**Solution**:
```java
try {
    return redisCache.get(key);
} catch (RedisException e) {
    logger.error("Redis down, falling back to database", e);
    return database.get(key);
}
```

---

### 4. Serialization Overhead
**Problem**: Slow serialization/deserialization to Redis

**Solution**:
- Use efficient formats (Protocol Buffers, MessagePack)
- Keep objects small
- Consider compression for large objects

---

### 5. Cache Stampede on L2 Miss
**Problem**: Many instances hit database when L2 key expires

**Solution**:
- Distributed locking
- Single-flight pattern (only one instance loads)

---

### 6. Not Setting TTLs
**Problem**: Cache grows indefinitely, never cleans up

**Solution**:
- Always set TTL on distributed cache
- Use eviction policies

---

## Best Practices

### 1. Layer by Access Pattern
```
L1: Ultra-hot, stable data (config, reference)
L2: Hot, shared data (products, users)
DB: Source of truth
```

### 2. Different TTLs per Layer
```
L1 TTL: 1-5 minutes (shorter)
L2 TTL: 30-60 minutes (longer)
```

### 3. Monitor Both Layers
```java
// L1 metrics
CacheStats stats = localCache.stats();
logger.info("L1 Hit Rate: {}", stats.hitRate());

// L2 metrics
RedisInfo info = redis.info();
logger.info("L2 Hit Rate: {}", info.getHitRate());
```

### 4. Graceful Degradation
```
L1 miss → L2
L2 miss → Database
L2 error → Database
Database error → Return error
```

### 5. Size Appropriately
```
L1: 1-10% of L2 size
L2: 10-30% of database working set
```

### 6. Invalidation Strategy
```
Immediate: Delete from both L1 and L2
Pub/Sub: Notify other instances
TTL: Let it expire naturally
```

### 7. Test Failure Scenarios
- L2 cache down
- Network partition
- Serialization errors
- Memory pressure

---

## Decision Tree

```
Start: Do I need caching?
├─ Yes
│  ├─ Is data shared across instances?
│  │  ├─ Yes → Use Distributed Cache (L2)
│  │  │  └─ Is it ultra-hot data?
│  │  │     └─ Yes → Add Local Cache (L1 + L2)
│  │  └─ No → Can it fit in memory?
│  │     ├─ Yes → Use Local Cache (L1)
│  │     └─ No → Use Distributed Cache (L2)
│  └─ Is consistency critical?
│     ├─ Yes → Use Distributed Cache (L2) only
│     └─ No → Consider Local Cache (L1) or Both
└─ No → Direct database access
```

---

## Key Takeaways

1. **Local cache** is fastest but inconsistent across instances
2. **Distributed cache** is slower but provides consistency
3. **Multi-level caching** combines benefits of both
4. **Choose based on**: consistency needs, dataset size, access patterns
5. **L1 for**: hot, stable data with high read frequency
6. **L2 for**: shared state, large datasets, consistency requirements
7. **Monitor metrics** to validate caching effectiveness
8. **Plan for failures** at each layer

---

## Next Steps

- [Two-Level Cache Architecture](./two-level-cache.md)
- [Cache-Aside Pattern](./cache-aside.md)
- [Cache Invalidation Strategies](./cache-invalidation.md)

---

**Last Updated**: 2026-01-17
**Target Audience**: Senior Software Engineers
**Difficulty Level**: Intermediate
