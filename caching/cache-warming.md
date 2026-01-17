# Cache Warming Strategies

## Table of Contents
1. [Introduction](#introduction)
2. [Why Cache Warming?](#why-cache-warming)
3. [Warming Strategies](#warming-strategies)
4. [Implementation Approaches](#implementation-approaches)
5. [Best Practices](#best-practices)
6. [Common Pitfalls](#common-pitfalls)
7. [Monitoring and Validation](#monitoring-and-validation)

---

## Introduction

**Cache warming** is the process of proactively loading data into cache before it's requested by users. This prevents the "cold start" problem where an empty cache leads to poor performance after application restarts or deployments.

---

## Why Cache Warming?

### The Cold Start Problem

```
Scenario: Application restart at 9:00 AM (peak traffic)

9:00:00 - App starts, cache empty
9:00:01 - 1000 requests arrive
Result:
- 1000 cache misses
- 1000 database queries
- Database overloaded
- Response times: 5000ms (vs normal 50ms)
- User complaints spike
```

### Impact

**Without Warming**:
```
Request 1:  Cache MISS → 500ms
Request 2:  Cache MISS → 500ms
Request 3:  Cache MISS → 500ms
...
Request 100: Cache HIT → 2ms (finally cached)

Average first 100 requests: 450ms
```

**With Warming**:
```
Pre-warm: Load 1000 popular items
Request 1:  Cache HIT → 2ms
Request 2:  Cache HIT → 2ms
Request 3:  Cache HIT → 2ms

Average: 2ms from start
```

---

## Warming Strategies

### 1. Preload on Startup

Load critical data when application starts.

```java
@Component
public class CacheWarmer implements ApplicationListener<ContextRefreshedEvent> {
    @Autowired private ProductRepository repository;
    @Autowired private Cache<String, Product> cache;
    
    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        logger.info("Starting cache warming...");
        
        long start = System.currentTimeMillis();
        int loaded = warmCache();
        long duration = System.currentTimeMillis() - start;
        
        logger.info("Cache warming complete: {} items in {}ms", 
            loaded, duration);
    }
    
    private int warmCache() {
        // Load popular products
        List<String> popularIds = getPopularProductIds();
        
        List<Product> products = repository.findAllById(popularIds);
        
        for (Product product : products) {
            cache.put("product:" + product.getId(), product);
        }
        
        return products.size();
    }
    
    private List<String> getPopularProductIds() {
        // Option 1: Static list
        // Option 2: Query from analytics
        // Option 3: Load from Redis sorted set
        return analytics.getTopProductIds(1000);
    }
}
```

---

### 2. Lazy Warming (Gradual)

Warm cache gradually as traffic comes in.

```java
@Service
public class GradualCacheWarmer {
    private final ScheduledExecutorService scheduler = 
        Executors.newScheduledThreadPool(1);
    
    private final Queue<String> warmingQueue = new ConcurrentLinkedQueue<>();
    
    @PostConstruct
    public void startWarming() {
        // Queue popular item IDs
        warmingQueue.addAll(getPopularItemIds());
        
        // Warm 10 items every second
        scheduler.scheduleAtFixedRate(() -> {
            for (int i = 0; i < 10 && !warmingQueue.isEmpty(); i++) {
                String id = warmingQueue.poll();
                if (id != null) {
                    warmItem(id);
                }
            }
        }, 1, 1, TimeUnit.SECONDS);
    }
    
    private void warmItem(String id) {
        try {
            Product product = repository.findById(id).orElse(null);
            if (product != null) {
                cache.put("product:" + id, product);
            }
        } catch (Exception e) {
            logger.warn("Failed to warm item: {}", id, e);
        }
    }
}
```

---

### 3. Scheduled Refresh

Periodically refresh cache before expiration.

```java
@Component
public class ScheduledCacheRefresh {
    
    @Scheduled(cron = "0 */30 * * * *")  // Every 30 minutes
    public void refreshPopularProducts() {
        logger.info("Refreshing popular products cache...");
        
        List<String> popularIds = analytics.getTopProductIds(1000);
        
        List<Product> products = repository.findAllById(popularIds);
        
        for (Product product : products) {
            String key = "product:" + product.getId();
            redis.setex(key, 3600, serialize(product));
        }
        
        logger.info("Refreshed {} products", products.size());
    }
    
    @Scheduled(cron = "0 0 2 * * *")  // 2 AM daily
    public void refreshReferenceData() {
        // Refresh slowly-changing reference data
        warmCountries();
        warmCategories();
        warmConfigurations();
    }
}
```

---

### 4. Access Pattern-Based Warming

Warm based on actual access patterns from previous day.

```java
@Service
public class AccessPatternWarmer {
    @Autowired private RedisTemplate<String, String> redis;
    
    // Track access patterns
    public void recordAccess(String productId) {
        String key = "access:product:" + productId;
        redis.opsForZSet().incrementScore("popular:products", productId, 1);
    }
    
    // Warm based on yesterday's top items
    @Scheduled(cron = "0 5 0 * * *")  // 12:05 AM
    public void warmBasedOnYesterdayAccess() {
        // Get top 1000 accessed products from yesterday
        Set<String> topProducts = redis.opsForZSet()
            .reverseRange("popular:products", 0, 999);
        
        logger.info("Warming {} products based on access patterns", 
            topProducts.size());
        
        for (String productId : topProducts) {
            Product product = repository.findById(productId).orElse(null);
            if (product != null) {
                cache.put("product:" + productId, product);
            }
        }
        
        // Reset counter for today
        redis.delete("popular:products");
    }
}
```

---

### 5. Predictive Warming

Warm cache based on predicted future access.

```java
@Service
public class PredictiveWarmer {
    
    // Warm products for upcoming sale
    public void warmForSale(String saleId) {
        Sale sale = saleRepository.findById(saleId).orElse(null);
        if (sale == null) return;
        
        logger.info("Warming cache for sale: {}", sale.getName());
        
        // Get products in sale
        List<Product> saleProducts = productRepository
            .findBySaleId(saleId);
        
        // Warm cache
        for (Product product : saleProducts) {
            cache.put("product:" + product.getId(), product);
        }
        
        logger.info("Warmed {} products for sale", saleProducts.size());
    }
    
    // Schedule warming before peak hours
    @Scheduled(cron = "0 30 7 * * *")  // 7:30 AM (before 8 AM peak)
    public void warmBeforePeakHours() {
        logger.info("Warming cache before peak hours...");
        warmTopProducts(2000);
    }
}
```

---

## Implementation Approaches

### Approach 1: Synchronous Startup Warming

```java
@Component
public class SynchronousWarmer implements ApplicationListener<ContextRefreshedEvent> {
    
    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        logger.info("Application ready, warming cache...");
        
        // Block startup until warming complete
        warmCache();
        
        logger.info("Cache warming complete, ready to serve traffic");
    }
    
    private void warmCache() {
        // Load critical data
        List<Product> products = repository.findTop1000();
        
        products.forEach(p -> 
            cache.put("product:" + p.getId(), p)
        );
    }
}
```

**Pros**: 
- Simple
- Cache ready when app starts serving traffic

**Cons**:
- Delays startup
- Blocks health checks
- May timeout deployment

---

### Approach 2: Asynchronous Background Warming

```java
@Component
public class AsyncWarmer {
    
    @Autowired private ApplicationEventPublisher eventPublisher;
    
    @EventListener(ContextRefreshedEvent.class)
    @Async
    public void warmCacheAsync() {
        logger.info("Starting async cache warming...");
        
        try {
            warmCache();
            eventPublisher.publishEvent(new CacheWarmingCompleteEvent());
        } catch (Exception e) {
            logger.error("Cache warming failed", e);
        }
    }
    
    private void warmCache() {
        // Warm in background
        List<String> ids = getPopularIds();
        
        ids.parallelStream()
            .forEach(id -> {
                Product p = repository.findById(id).orElse(null);
                if (p != null) {
                    cache.put("product:" + id, p);
                }
            });
    }
}

// Enable async
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("cache-warmer-");
        executor.initialize();
        return executor;
    }
}
```

**Pros**:
- Doesn't block startup
- Faster deployment
- Can serve traffic while warming

**Cons**:
- Initial requests may be slow
- More complex

---

### Approach 3: Database Dump to Cache

```java
@Service
public class BulkCacheWarmer {
    
    public void warmFromDatabase() {
        logger.info("Bulk warming cache from database...");
        
        long start = System.currentTimeMillis();
        
        // Stream from database to avoid loading all into memory
        try (Stream<Product> stream = repository.streamAll()) {
            stream.forEach(product -> {
                try {
                    String key = "product:" + product.getId();
                    redis.setex(key, 3600, serialize(product));
                } catch (Exception e) {
                    logger.warn("Failed to cache product: {}", 
                        product.getId(), e);
                }
            });
        }
        
        long duration = System.currentTimeMillis() - start;
        logger.info("Bulk warming complete in {}ms", duration);
    }
}

// Repository with streaming
public interface ProductRepository extends JpaRepository<Product, String> {
    
    @Query("SELECT p FROM Product p")
    Stream<Product> streamAll();
}
```

---

### Approach 4: Cache Snapshot/Restore

```java
@Service
public class CacheSnapshotService {
    
    // Before shutdown: Save cache snapshot
    @PreDestroy
    public void saveCacheSnapshot() {
        logger.info("Saving cache snapshot...");
        
        Map<String, String> snapshot = new HashMap<>();
        
        // Get all cache keys
        Set<String> keys = redis.keys("product:*");
        
        for (String key : keys) {
            String value = redis.get(key);
            if (value != null) {
                snapshot.put(key, value);
            }
        }
        
        // Save to file
        try (ObjectOutputStream oos = new ObjectOutputStream(
                new FileOutputStream("/tmp/cache-snapshot.dat"))) {
            oos.writeObject(snapshot);
        } catch (IOException e) {
            logger.error("Failed to save snapshot", e);
        }
        
        logger.info("Saved {} cache entries", snapshot.size());
    }
    
    // After startup: Restore from snapshot
    @PostConstruct
    public void restoreCacheSnapshot() {
        File snapshot = new File("/tmp/cache-snapshot.dat");
        if (!snapshot.exists()) {
            logger.info("No cache snapshot found");
            return;
        }
        
        logger.info("Restoring cache from snapshot...");
        
        try (ObjectInputStream ois = new ObjectInputStream(
                new FileInputStream(snapshot))) {
            
            @SuppressWarnings("unchecked")
            Map<String, String> data = (Map<String, String>) ois.readObject();
            
            // Restore to Redis
            for (Map.Entry<String, String> entry : data.entrySet()) {
                redis.setex(entry.getKey(), 3600, entry.getValue());
            }
            
            logger.info("Restored {} cache entries", data.size());
        } catch (Exception e) {
            logger.error("Failed to restore snapshot", e);
        }
    }
}
```

---

## Best Practices

### 1. **Prioritize Critical Data**

```java
public void warmCache() {
    // Tier 1: Critical data (must have)
    warmHomepageFeatured();
    warmTopProducts(100);
    
    // Tier 2: Important data (nice to have)
    warmPopularCategories();
    warmTopProducts(1000);
    
    // Tier 3: Optional data (low priority)
    warmAllProducts();
}
```

---

### 2. **Use Batching**

```java
public void warmProducts(List<String> productIds) {
    // Batch database queries
    List<List<String>> batches = Lists.partition(productIds, 100);
    
    for (List<String> batch : batches) {
        List<Product> products = repository.findAllById(batch);
        
        // Batch Redis writes
        Map<String, String> cacheData = products.stream()
            .collect(Collectors.toMap(
                p -> "product:" + p.getId(),
                p -> serialize(p)
            ));
        
        redis.mset(cacheData);
    }
}
```

---

### 3. **Set TTL During Warming**

```java
public void warmWithTTL(Product product) {
    String key = "product:" + product.getId();
    
    // Add jitter to prevent synchronized expiration
    int ttl = 3600 + random.nextInt(600);  // 3600-4200 sec
    
    redis.setex(key, ttl, serialize(product));
}
```

---

### 4. **Monitor Warming Progress**

```java
@Service
public class WarmingMonitor {
    private final AtomicInteger total = new AtomicInteger(0);
    private final AtomicInteger completed = new AtomicInteger(0);
    
    public void startWarming(int totalItems) {
        total.set(totalItems);
        completed.set(0);
    }
    
    public void recordCompleted() {
        int current = completed.incrementAndGet();
        
        if (current % 100 == 0) {
            int progress = (current * 100) / total.get();
            logger.info("Warming progress: {}% ({}/{})", 
                progress, current, total.get());
        }
    }
    
    public boolean isComplete() {
        return completed.get() >= total.get();
    }
}
```

---

### 5. **Graceful Degradation**

```java
@Service
public class GracefulWarmer {
    private volatile boolean warmingComplete = false;
    
    @Async
    public void warmCache() {
        try {
            doWarmCache();
            warmingComplete = true;
        } catch (Exception e) {
            logger.error("Warming failed, continuing anyway", e);
            warmingComplete = true;  // Don't block traffic
        }
    }
    
    @GetMapping("/health")
    public ResponseEntity<String> health() {
        if (!warmingComplete) {
            // Return 503 while warming
            return ResponseEntity.status(503)
                .body("Warming cache...");
        }
        return ResponseEntity.ok("OK");
    }
}
```

---

### 6. **Use Data Sources Wisely**

```java
// Option 1: From database
List<Product> products = repository.findTop1000ByOrderByViewsDesc();

// Option 2: From analytics
List<String> ids = analytics.getTopProductIds(1000);

// Option 3: From previous cache (Redis persistence)
Set<String> keys = redis.keys("product:*");

// Option 4: From static configuration
List<String> ids = Arrays.asList("prod1", "prod2", ...);

// Best: Combine multiple sources
Set<String> idsToWarm = new HashSet<>();
idsToWarm.addAll(getTopFromAnalytics(500));
idsToWarm.addAll(getFeatureProducts());
idsToWarm.addAll(getNewReleases());
```

---

## Common Pitfalls

### 1. **Warming Too Much Data**

```java
// BAD: Warm entire database
repository.findAll().forEach(p -> cache.put(p.getId(), p));

// GOOD: Warm only popular data (80/20 rule)
analytics.getTop20Percent().forEach(p -> cache.put(p.getId(), p));
```

---

### 2. **Blocking Startup Too Long**

```java
// BAD: Block startup for minutes
@PostConstruct
public void warmCache() {
    warmAllMillionProducts();  // Takes 5 minutes
}

// GOOD: Warm critical data quickly, rest async
@PostConstruct
public void warmCache() {
    warmTop100Products();  // Takes 1 second
    scheduleBackgroundWarming();
}
```

---

### 3. **Ignoring Errors**

```java
// BAD: Silent failure
warmCache();  // If it fails, cache empty

// GOOD: Handle errors gracefully
try {
    warmCache();
} catch (Exception e) {
    logger.error("Cache warming failed", e);
    metrics.counter("cache.warming.failure").increment();
    // Application still starts, serves from DB
}
```

---

### 4. **Not Setting TTL**

```java
// BAD: No TTL during warming
cache.set(key, value);  // Never expires

// GOOD: Set TTL
cache.setex(key, 3600, value);  // Expires in 1 hour
```

---

## Monitoring and Validation

### Metrics to Track

```java
@Service
public class WarmingMetrics {
    
    public void recordWarming(int itemsWarmed, long durationMs) {
        metrics.counter("cache.warming.items").increment(itemsWarmed);
        metrics.timer("cache.warming.duration").record(durationMs, TimeUnit.MILLISECONDS);
    }
    
    @Scheduled(fixedRate = 60000)
    public void checkCacheHealth() {
        long cacheSize = redis.dbSize();
        double hitRate = getCacheHitRate();
        
        metrics.gauge("cache.size", cacheSize);
        metrics.gauge("cache.hit_rate", hitRate);
        
        if (hitRate < 0.8) {
            logger.warn("Low cache hit rate: {}, consider warming more data", 
                hitRate);
        }
    }
}
```

---

### Validation

```java
@Test
public void validateCacheWarming() {
    // Clear cache
    cache.clear();
    
    // Warm cache
    cacheWarmer.warmCache();
    
    // Validate
    List<String> criticalIds = getCriticalProductIds();
    for (String id : criticalIds) {
        assertNotNull("Product should be cached: " + id, 
            cache.get("product:" + id));
    }
}
```

---

## Key Takeaways

1. **Cache warming prevents cold start problems** — empty cache hurts performance
2. **Prioritize critical data** — don't warm everything
3. **Use async warming** — don't block startup
4. **Batch operations** — efficient database and cache loading
5. **Monitor warming progress** — track success and failures
6. **Set TTL during warming** — prevent stale data
7. **Graceful degradation** — application works even if warming fails
8. **Use access patterns** — warm what users actually need

---

## Next Steps

- [Cache Stampede Prevention](./cache-stampede.md)
- [Cache Eviction Policies](./cache-eviction-policies.md)
- [Read-Through Pattern](./read-through.md)

---

**Last Updated**: 2026-01-17
**Target Audience**: Senior Software Engineers
**Difficulty Level**: Intermediate to Advanced
