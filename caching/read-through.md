# Read-Through Cache Pattern

## Table of Contents
1. [Introduction](#introduction)
2. [Pattern Overview](#pattern-overview)
3. [How It Works](#how-it-works)
4. [Implementation](#implementation)
5. [Advantages](#advantages)
6. [Disadvantages](#disadvantages)
7. [vs Cache-Aside](#vs-cache-aside)
8. [Use Cases](#use-cases)
9. [Best Practices](#best-practices)

---

## Introduction

**Read-Through Cache** is a caching pattern where the cache sits transparently between the application and the database. On a cache miss, the cache itself is responsible for loading data from the database, not the application.

This differs from cache-aside where the application explicitly manages cache loading.

---

## Pattern Overview

### Architecture

```
┌──────────────┐
│ Application  │
└──────┬───────┘
       │
       │ Read request
       ▼
   ┌────────┐
   │ Cache  │ ← Cache manages loading
   └────┬───┘
        │
        │ On miss, cache loads from database
        ▼
   ┌──────────┐
   │ Database │
   └──────────┘
```

### Key Characteristics
- **Transparent**: Application doesn't know about database
- **Cache-managed loading**: Cache loads data on miss
- **Simpler application code**: No explicit cache logic
- **Lazy loading**: Data loaded on first access

---

## How It Works

### Read Flow

```
1. Application requests data from cache
2. Cache checks if data exists
3. If HIT: Return data
4. If MISS:
   ├─ Cache queries database (not application)
   ├─ Cache stores result
   └─ Cache returns data to application
5. Application receives data (unaware of cache miss)
```

### Visual Flow

```
Application perspective:
┌─────────┐
│  App    │ ──── Request ────→ Cache ──── Data ────→ App
└─────────┘                     │
                                │ (miss handling hidden)
                                
Cache perspective (on miss):
┌─────────┐
│ Cache   │ ──── Query ────→ Database
│         │ ←─── Data ─────── Database
│         │ Store data
│         │ ──── Data ────→ App
└─────────┘
```

---

## Implementation

### Basic Read-Through (Java)

```java
public interface CacheLoader<K, V> {
    V load(K key);
}

public class ReadThroughCache<K, V> {
    private final Map<K, V> cache = new ConcurrentHashMap<>();
    private final CacheLoader<K, V> loader;
    private final int ttl;
    
    public ReadThroughCache(CacheLoader<K, V> loader, int ttl) {
        this.loader = loader;
        this.ttl = ttl;
    }
    
    public V get(K key) {
        // Check cache
        V cached = cache.get(key);
        if (cached != null) {
            return cached;
        }
        
        // Cache miss - loader handles database access
        synchronized (this) {
            // Double-check after acquiring lock
            cached = cache.get(key);
            if (cached != null) {
                return cached;
            }
            
            // Load from database (via loader)
            V value = loader.load(key);
            
            if (value != null) {
                cache.put(key, value);
                // Schedule expiration
                scheduleExpiration(key, ttl);
            }
            
            return value;
        }
    }
}
```

---

### Guava LoadingCache (Read-Through)

```java
import com.google.common.cache.CacheBuilder;
import com.google.common.cache.CacheLoader;
import com.google.common.cache.LoadingCache;

@Service
public class ProductCacheService {
    private final LoadingCache<String, Product> productCache;
    
    public ProductCacheService(ProductRepository repository) {
        this.productCache = CacheBuilder.newBuilder()
            .maximumSize(10000)
            .expireAfterWrite(30, TimeUnit.MINUTES)
            .recordStats()
            .build(new CacheLoader<String, Product>() {
                @Override
                public Product load(String productId) {
                    // Cache automatically calls this on miss
                    return repository.findById(productId)
                        .orElseThrow(() -> new ProductNotFoundException(productId));
                }
            });
    }
    
    public Product getProduct(String productId) {
        try {
            // Application just calls get()
            // Cache handles loading automatically
            return productCache.get(productId);
        } catch (ExecutionException e) {
            throw new RuntimeException("Failed to load product", e);
        }
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

### Caffeine LoadingCache (Modern, High-Performance)

```java
import com.github.benmanes.caffeine.cache.Caffeine;
import com.github.benmanes.caffeine.cache.LoadingCache;

@Service
public class UserCacheService {
    private final LoadingCache<String, User> userCache;
    
    public UserCacheService(UserRepository repository) {
        this.userCache = Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(10, TimeUnit.MINUTES)
            .refreshAfterWrite(5, TimeUnit.MINUTES)  // Proactive refresh
            .recordStats()
            .build(userId -> {
                // Auto-loading function
                return repository.findById(userId).orElse(null);
            });
    }
    
    public User getUser(String userId) {
        return userCache.get(userId);
    }
    
    // Async read-through
    public CompletableFuture<User> getUserAsync(String userId) {
        return userCache.asMap().computeIfAbsent(userId, key -> {
            return CompletableFuture.supplyAsync(() -> 
                repository.findById(key).orElse(null)
            );
        });
    }
}
```

---

### Spring Cache with @Cacheable (Read-Through)

```java
@Service
public class ProductService {
    @Autowired private ProductRepository repository;
    
    @Cacheable(
        value = "products",
        key = "#productId",
        unless = "#result == null"
    )
    public Product getProduct(String productId) {
        // Spring cache automatically:
        // 1. Checks cache
        // 2. On miss, calls this method
        // 3. Stores result in cache
        // 4. Returns cached value on next call
        
        return repository.findById(productId).orElse(null);
    }
    
    @Cacheable(value = "products", key = "#categoryId + '_' + #page")
    public List<Product> getProductsByCategory(
        String categoryId, 
        int page
    ) {
        return repository.findByCategory(categoryId, 
            PageRequest.of(page, 20));
    }
}

@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration
            .defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30));
        
        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .build();
    }
}
```

---

### Redis-Based Read-Through

```java
@Service
public class RedisReadThroughCache {
    @Autowired private JedisPool redisPool;
    @Autowired private ProductRepository repository;
    @Autowired private ObjectMapper objectMapper;
    
    public Product getProduct(String productId) {
        String key = "product:" + productId;
        
        try (Jedis jedis = redisPool.getResource()) {
            String cached = jedis.get(key);
            
            if (cached != null) {
                // Cache hit
                return objectMapper.readValue(cached, Product.class);
            }
            
            // Cache miss - read-through logic
            Product product = loadFromDatabase(productId);
            
            if (product != null) {
                // Store in cache
                String json = objectMapper.writeValueAsString(product);
                jedis.setex(key, 3600, json);
            }
            
            return product;
        } catch (Exception e) {
            logger.error("Cache error, falling back to database", e);
            return loadFromDatabase(productId);
        }
    }
    
    private Product loadFromDatabase(String productId) {
        return repository.findById(productId).orElse(null);
    }
}
```

---

## Advantages

### 1. **Simplified Application Code**

**Cache-Aside (Application manages loading)**:
```java
public Product getProduct(String id) {
    // Application must:
    // 1. Check cache
    Product cached = cache.get(id);
    if (cached != null) return cached;
    
    // 2. Load from DB
    Product product = database.get(id);
    
    // 3. Update cache
    if (product != null) {
        cache.set(id, product);
    }
    
    return product;
}
```

**Read-Through (Cache manages loading)**:
```java
public Product getProduct(String id) {
    // Application just requests data
    return cache.get(id);  // Cache handles loading
}
```

**Benefit**: Simpler, fewer lines of code, less room for errors.

---

### 2. **Consistent Caching Logic**

```java
// Cache loading logic centralized in one place
LoadingCache<String, Product> cache = Caffeine.newBuilder()
    .build(key -> database.loadProduct(key));

// All gets use same logic
Product p1 = cache.get("product1");
Product p2 = cache.get("product2");
```

No risk of different code paths loading data differently.

---

### 3. **Transparent to Application**

```java
// Application doesn't know about cache misses
Product product = cacheService.getProduct(id);

// Works the same whether cache hit or miss
// Application code unchanged
```

---

### 4. **Built-in Libraries**

```java
// Many libraries support read-through pattern
- Guava LoadingCache
- Caffeine LoadingCache
- Spring @Cacheable
- Ehcache
- JCache (JSR-107)
```

Well-tested, production-ready implementations available.

---

## Disadvantages

### 1. **Cache Miss Penalty**

```java
Same as cache-aside:
- Check cache: 1ms
- Load from database: 50ms
- Store in cache: 1ms
Total: 52ms on miss

Not inherently slower, just not faster than cache-aside
```

---

### 2. **Initial Cold Start**

```java
// Empty cache after restart
First request: 52ms (cache miss)
Second request: 1ms (cache hit)

// All users experience initial slow requests
```

**Solution**: Cache warming

---

### 3. **Less Control**

```java
// Cache-Aside: Application controls loading
public Product get(String id) {
    // Can add custom logic:
    // - Fallbacks
    // - Enrichment
    // - Validation
    return customLoad(id);
}

// Read-Through: Cache controls loading
cache.get(id);  // Less flexibility
```

---

### 4. **Error Handling Complexity**

```java
// What if database is down?
try {
    return cache.get(id);  // Throws exception on load failure
} catch (ExecutionException e) {
    // How to handle?
    // - Return null?
    // - Default value?
    // - Re-throw?
}
```

---

## vs Cache-Aside

### Comparison Table

| Aspect | Read-Through | Cache-Aside |
|--------|--------------|-------------|
| **Who loads?** | Cache | Application |
| **Code complexity** | Lower | Higher |
| **Flexibility** | Lower | Higher |
| **Performance** | Same | Same |
| **Cache miss handling** | Cache-managed | App-managed |
| **Error handling** | Cache layer | Application layer |
| **Best for** | Standard use cases | Custom logic needed |

---

### Code Comparison

**Read-Through**:
```java
// Simple
Product product = cache.get(id);
```

**Cache-Aside**:
```java
// More verbose
Product cached = cache.get(id);
if (cached == null) {
    Product product = database.get(id);
    if (product != null) {
        cache.set(id, product);
    }
    return product;
}
return cached;
```

---

### When to Choose

**Choose Read-Through When**:
- ✅ Standard caching needs
- ✅ Want simpler code
- ✅ Using caching library (Guava, Caffeine)
- ✅ Consistent access patterns

**Choose Cache-Aside When**:
- ✅ Need custom loading logic
- ✅ Complex fallback strategies
- ✅ Different caching strategies per endpoint
- ✅ Specific error handling requirements

---

## Use Cases

### ✅ Good Fit

**1. Standard CRUD Applications**
```java
@Service
public class UserService {
    @Cacheable("users")
    public User getUser(String id) {
        return repository.findById(id).orElse(null);
    }
}
```

**2. Reference Data**
```java
LoadingCache<String, Country> countries = Caffeine.newBuilder()
    .expireAfterWrite(24, TimeUnit.HOURS)
    .build(code -> countryRepository.findByCode(code));
```

**3. Configuration**
```java
LoadingCache<String, Config> config = Caffeine.newBuilder()
    .refreshAfterWrite(5, TimeUnit.MINUTES)
    .build(key -> configService.loadConfig(key));
```

**4. Product Catalogs**
```java
@Cacheable(value = "products", key = "#sku")
public Product getProductBySku(String sku) {
    return productRepository.findBySku(sku);
}
```

---

## Best Practices

### 1. **Set Appropriate TTL**

```java
LoadingCache<String, Product> cache = Caffeine.newBuilder()
    .expireAfterWrite(30, TimeUnit.MINUTES)  // TTL
    .refreshAfterWrite(15, TimeUnit.MINUTES)  // Proactive refresh
    .build(key -> loadProduct(key));
```

---

### 2. **Handle Null Values**

```java
LoadingCache<String, Product> cache = Caffeine.newBuilder()
    .build(key -> {
        Optional<Product> product = repository.findById(key);
        
        // Cache null to prevent cache penetration
        return product.orElse(Product.NULL_OBJECT);
    });
```

---

### 3. **Implement Fallbacks**

```java
public Product getProduct(String id) {
    try {
        return productCache.get(id);
    } catch (Exception e) {
        logger.error("Cache load failed", e);
        
        // Fallback: Direct database access
        return repository.findById(id).orElse(null);
    }
}
```

---

### 4. **Use Refresh-Ahead**

```java
LoadingCache<String, Product> cache = Caffeine.newBuilder()
    .expireAfterWrite(1, TimeUnit.HOURS)
    .refreshAfterWrite(50, TimeUnit.MINUTES)  // Refresh before expiry
    .build(key -> loadProduct(key));
```

Prevents cache misses for hot data.

---

### 5. **Monitor Cache Performance**

```java
@Scheduled(fixedRate = 60000)
public void logCacheStats() {
    CacheStats stats = cache.stats();
    
    logger.info("Cache stats: Hit rate: {}, Miss rate: {}, Load time: {}ms",
        stats.hitRate(),
        stats.missRate(),
        stats.averageLoadPenalty() / 1_000_000);
    
    if (stats.hitRate() < 0.8) {
        logger.warn("Low cache hit rate: {}", stats.hitRate());
    }
}
```

---

### 6. **Bulk Loading**

```java
LoadingCache<String, Product> cache = Caffeine.newBuilder()
    .build(new CacheLoader<String, Product>() {
        @Override
        public Product load(String key) {
            return repository.findById(key).orElse(null);
        }
        
        @Override
        public Map<String, Product> loadAll(Set<String> keys) {
            // Bulk load for efficiency
            return repository.findAllById(keys).stream()
                .collect(Collectors.toMap(Product::getId, p -> p));
        }
    });

// Usage
Map<String, Product> products = cache.getAll(Arrays.asList("id1", "id2", "id3"));
```

---

### 7. **Eviction Listeners**

```java
LoadingCache<String, Product> cache = Caffeine.newBuilder()
    .removalListener((key, value, cause) -> {
        logger.info("Evicted: {} ({})", key, cause);
        metrics.counter("cache.eviction", "cause", cause.name()).increment();
    })
    .build(key -> loadProduct(key));
```

---

## Key Takeaways

1. **Read-Through simplifies application code** — cache handles loading
2. **Same performance as cache-aside** — just cleaner API
3. **Well-supported by libraries** — Guava, Caffeine, Spring
4. **Transparent to application** — works like simple map lookup
5. **Less flexible** — for standard use cases, not custom logic
6. **Use with refresh-ahead** — prevents misses on hot data
7. **Monitor hit rates** — validate caching effectiveness

---

## Next Steps

- [Cache-Aside Pattern](./cache-aside.md)
- [Write-Through Pattern](./write-through.md)
- [Cache Warming](./cache-warming.md)
- [Cache Eviction Policies](./cache-eviction-policies.md)

---

**Last Updated**: 2026-01-17
**Target Audience**: Senior Software Engineers
**Difficulty Level**: Intermediate
