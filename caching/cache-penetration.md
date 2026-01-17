# Cache Penetration Prevention

## Table of Contents
1. [Introduction](#introduction)
2. [The Problem](#the-problem)
3. [Attack Scenarios](#attack-scenarios)
4. [Solution Strategies](#solution-strategies)
5. [Implementation Examples](#implementation-examples)
6. [Best Practices](#best-practices)
7. [Related Problems](#related-problems)

---

## Introduction

**Cache Penetration** (also called **Cache Miss Attack** or **Bypass Attack**) occurs when requests query for data that doesn't exist in the cache OR the database. Since the data doesn't exist, it's never cached, causing every request to hit the database.

This can be exploited as a DDoS vector to overload your database.

---

## The Problem

### Normal Cache Operation

```
Request: GET /product/valid-id-123
1. Check cache → MISS
2. Query database → FOUND
3. Store in cache
4. Return data

Next request: GET /product/valid-id-123
1. Check cache → HIT
2. Return from cache (fast)
```

---

### Cache Penetration Scenario

```
Request: GET /product/invalid-id-999999
1. Check cache → MISS (doesn't exist)
2. Query database → NOT FOUND
3. Don't cache (null result)
4. Return 404

Next request: GET /product/invalid-id-999999
1. Check cache → MISS (still not cached)
2. Query database → NOT FOUND (again!)
3. Database hit every time

Result: Every request hits database, cache provides no protection
```

---

### Malicious Attack

```
Attacker script:
for i in 1..1000000:
    request GET /product/invalid-id-{random()}

Result:
- 1M cache misses
- 1M database queries for non-existent data
- Database overwhelmed
- Service degradation or outage
```

---

### Visual Representation

```
Normal (valid IDs):
────────────────────────────────────────
Request:  ████████████████████████████
Cache:    ✗✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓
Database: ✓──────────────────────────── (only first miss)

Cache Penetration (invalid IDs):
────────────────────────────────────────
Request:  ████████████████████████████
Cache:    ✗✗✗✗✗✗✗✗✗✗✗✗✗✗✗✗✗✗✗✗✗✗✗✗✗✗✗✗
Database: ✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓ (every request!)
```

---

## Attack Scenarios

### Scenario 1: Brute Force Attack

```java
// Attacker tries random product IDs
for (int i = 0; i < 1000000; i++) {
    String randomId = UUID.randomUUID().toString();
    httpClient.get("/api/products/" + randomId);
}

// Each request:
// - Misses cache
// - Queries database
// - Returns 404

// Database: 1M queries for non-existent data
```

---

### Scenario 2: Sequential Scanning

```java
// Attacker scans ID range
for (int i = 1; i < 1000000; i++) {
    httpClient.get("/api/products/" + i);
}

// Most IDs don't exist
// Database queries for each
```

---

### Scenario 3: Typo/Fat-Finger Attacks

```java
// Legitimate user with typos
Request: /api/products/12345 → Valid, cached
Request: /api/products/12346 → Typo, DB query
Request: /api/products/12347 → Typo, DB query
Request: /api/products/12348 → Typo, DB query

// Accumulation of typos creates DB load
```

---

## Solution Strategies

### Strategy Comparison

| Solution | Effectiveness | Performance | Complexity | Memory | False Positives |
|----------|---------------|-------------|------------|--------|-----------------|
| Cache Null Results | High | Good | Low | Medium | None |
| Bloom Filter | High | Excellent | Medium | Low | Yes (< 1%) |
| Request Validation | Medium | Excellent | Low | None | None |
| Rate Limiting | Medium | Good | Medium | Low | None |
| Empty Object Pattern | High | Good | Low | Medium | None |

---

### Solution 1: Cache Null Results

**Concept**: Cache the fact that data doesn't exist.

```java
@Service
public class CacheWithNullHandling {
    private static final String NULL_MARKER = "NULL";
    
    public Product getProduct(String productId) {
        String key = "product:" + productId;
        
        // Check cache
        String cached = redis.get(key);
        
        if (cached != null) {
            if (NULL_MARKER.equals(cached)) {
                return null;  // Cached negative result
            }
            return deserialize(cached);
        }
        
        // Query database
        Product product = repository.findById(productId).orElse(null);
        
        if (product != null) {
            // Cache the product
            redis.setex(key, 3600, serialize(product));
        } else {
            // Cache the fact that it doesn't exist
            redis.setex(key, 300, NULL_MARKER);  // Shorter TTL (5 min)
        }
        
        return product;
    }
}
```

**Why shorter TTL for null?**
- Data might be created later
- Prevent caching incorrect state long-term
- Balance between protection and freshness

**Advantages**:
- ✅ Simple to implement
- ✅ Prevents repeated DB queries
- ✅ Low complexity

**Disadvantages**:
- ❌ Uses cache memory for non-existent data
- ❌ Attacker can fill cache with null entries
- ❌ May cache transient "not found" states

---

### Solution 2: Bloom Filter

**Concept**: Use probabilistic data structure to check if ID exists before querying.

#### What is a Bloom Filter?

```
Bloom Filter properties:
- Space efficient: ~10 bits per element
- Fast: O(1) lookup
- False positives possible (~0.1-1%)
- False negatives: NEVER

If Bloom Filter says "NO" → Definitely doesn't exist
If Bloom Filter says "YES" → Probably exists (verify with DB)
```

---

#### Implementation

```java
@Service
public class BloomFilterCache {
    private final BloomFilter<String> productFilter;
    private final ProductRepository repository;
    
    public BloomFilterCache() {
        // Create Bloom Filter for 1M elements with 1% false positive rate
        this.productFilter = BloomFilter.create(
            Funnels.stringFunnel(Charset.defaultCharset()),
            1_000_000,  // Expected elements
            0.01        // False positive probability
        );
        
        // Initialize filter with existing product IDs
        initializeFilter();
    }
    
    private void initializeFilter() {
        logger.info("Initializing Bloom Filter...");
        
        List<String> productIds = repository.findAllIds();
        productIds.forEach(productFilter::put);
        
        logger.info("Bloom Filter initialized with {} products", 
            productIds.size());
    }
    
    public Product getProduct(String productId) {
        // 1. Check Bloom Filter first
        if (!productFilter.mightContain(productId)) {
            // Definitely doesn't exist - no DB query needed!
            logger.debug("Bloom filter: Product {} definitely doesn't exist", 
                productId);
            return null;
        }
        
        // 2. Might exist - check cache
        String key = "product:" + productId;
        Product cached = cache.get(key);
        if (cached != null) {
            return cached;
        }
        
        // 3. Query database (might be false positive)
        Product product = repository.findById(productId).orElse(null);
        
        if (product != null) {
            cache.put(key, product);
        }
        
        return product;
    }
    
    // Update filter when new product created
    @EventListener
    public void onProductCreated(ProductCreatedEvent event) {
        productFilter.put(event.getProductId());
    }
}
```

---

#### Redis-based Bloom Filter

```java
@Service
public class RedisBloomFilter {
    @Autowired private JedisPool jedisPool;
    
    private static final String BLOOM_KEY = "bloom:products";
    
    // Initialize (use RedisBloom module)
    public void initialize(List<String> productIds) {
        try (Jedis jedis = jedisPool.getResource()) {
            // Create Bloom Filter
            // BF.RESERVE key error_rate capacity
            jedis.sendCommand(Protocol.Command.BF_RESERVE, 
                BLOOM_KEY, "0.01", "1000000");
            
            // Add all product IDs
            for (String id : productIds) {
                jedis.sendCommand(Protocol.Command.BF_ADD, 
                    BLOOM_KEY, id);
            }
        }
    }
    
    public boolean mightExist(String productId) {
        try (Jedis jedis = jedisPool.getResource()) {
            Object result = jedis.sendCommand(Protocol.Command.BF_EXISTS, 
                BLOOM_KEY, productId);
            return (Long) result == 1;
        }
    }
    
    public void add(String productId) {
        try (Jedis jedis = jedisPool.getResource()) {
            jedis.sendCommand(Protocol.Command.BF_ADD, 
                BLOOM_KEY, productId);
        }
    }
}
```

**Advantages**:
- ✅ Extremely space efficient
- ✅ Fast lookups (O(1))
- ✅ Prevents most invalid queries
- ✅ Low memory overhead

**Disadvantages**:
- ❌ False positives (~1%)
- ❌ Cannot remove elements
- ❌ Requires initialization
- ❌ Must update on new items

---

### Solution 3: Request Validation

**Concept**: Validate request parameters before cache/database lookup.

```java
@RestController
public class ProductController {
    
    @GetMapping("/products/{id}")
    public ResponseEntity<Product> getProduct(@PathVariable String id) {
        // 1. Validate ID format
        if (!isValidProductId(id)) {
            return ResponseEntity.badRequest().build();
        }
        
        // 2. Check cache/database
        Product product = productService.getProduct(id);
        
        if (product == null) {
            return ResponseEntity.notFound().build();
        }
        
        return ResponseEntity.ok(product);
    }
    
    private boolean isValidProductId(String id) {
        // Validate format
        if (id == null || id.isEmpty()) {
            return false;
        }
        
        // Check length
        if (id.length() != 36) {  // UUID length
            return false;
        }
        
        // Check pattern
        if (!id.matches("^[a-f0-9-]{36}$")) {
            return false;
        }
        
        return true;
    }
}
```

**Advantages**:
- ✅ Prevents obviously invalid requests
- ✅ No cache/DB overhead
- ✅ Low complexity

**Disadvantages**:
- ❌ Doesn't prevent well-formed but non-existent IDs
- ❌ Limited effectiveness

---

### Solution 4: Rate Limiting

**Concept**: Limit requests from single source.

```java
@Service
public class RateLimitedCache {
    private final RateLimiter rateLimiter;
    
    public RateLimitedCache() {
        // Allow 100 requests per minute per IP
        this.rateLimiter = RateLimiter.create(100.0 / 60.0);
    }
    
    public Product getProduct(String productId, String clientIp) {
        // Check rate limit
        if (!rateLimiter.tryAcquire()) {
            throw new TooManyRequestsException("Rate limit exceeded");
        }
        
        // Proceed with normal cache logic
        return fetchProduct(productId);
    }
}
```

**Redis-based Rate Limiting**:
```java
public boolean isAllowed(String clientIp) {
    String key = "rate_limit:" + clientIp;
    
    try (Jedis jedis = redisPool.getResource()) {
        Long count = jedis.incr(key);
        
        if (count == 1) {
            jedis.expire(key, 60);  // 60 second window
        }
        
        return count <= 100;  // Max 100 requests per minute
    }
}
```

---

### Solution 5: Empty Object Pattern

**Concept**: Return a special empty object instead of null.

```java
public class Product {
    public static final Product EMPTY = new Product(
        "EMPTY", "Not Found", BigDecimal.ZERO
    );
    
    private final String id;
    private final String name;
    private final BigDecimal price;
    
    public boolean isEmpty() {
        return this == EMPTY;
    }
}

@Service
public class ProductService {
    
    public Product getProduct(String productId) {
        String key = "product:" + productId;
        
        // Check cache
        Product cached = cache.get(key);
        if (cached != null) {
            return cached;
        }
        
        // Query database
        Product product = repository.findById(productId)
            .orElse(Product.EMPTY);
        
        // Cache result (even if empty)
        cache.put(key, product, 300);  // 5 min TTL
        
        return product;
    }
}
```

---

## Implementation Examples

### Complete Solution (Multiple Strategies Combined)

```java
@Service
public class ProtectedCacheService {
    @Autowired private BloomFilter<String> productFilter;
    @Autowired private JedisPool redisPool;
    @Autowired private ProductRepository repository;
    @Autowired private RateLimiter rateLimiter;
    
    private static final String NULL_MARKER = "NULL";
    
    public Product getProduct(String productId, String clientIp) {
        // 1. Rate limiting
        if (!rateLimiter.tryAcquire()) {
            throw new RateLimitException("Too many requests");
        }
        
        // 2. Input validation
        if (!isValidProductId(productId)) {
            throw new IllegalArgumentException("Invalid product ID");
        }
        
        // 3. Bloom filter check
        if (!productFilter.mightContain(productId)) {
            logger.debug("Bloom filter: Product {} doesn't exist", productId);
            return null;  // Definitely doesn't exist
        }
        
        // 4. Cache check (including null cache)
        String key = "product:" + productId;
        try (Jedis jedis = redisPool.getResource()) {
            String cached = jedis.get(key);
            
            if (cached != null) {
                if (NULL_MARKER.equals(cached)) {
                    return null;  // Cached negative result
                }
                return deserialize(cached);
            }
        }
        
        // 5. Database query
        Product product = repository.findById(productId).orElse(null);
        
        // 6. Update cache
        try (Jedis jedis = redisPool.getResource()) {
            if (product != null) {
                jedis.setex(key, 3600, serialize(product));
            } else {
                // Cache negative result with shorter TTL
                jedis.setex(key, 300, NULL_MARKER);
            }
        }
        
        return product;
    }
    
    private boolean isValidProductId(String id) {
        return id != null && 
               id.length() == 36 && 
               id.matches("^[a-f0-9-]{36}$");
    }
}
```

---

## Best Practices

### 1. **Use Short TTL for Null Results**

```java
if (product != null) {
    cache.setex(key, 3600, value);   // 1 hour for valid data
} else {
    cache.setex(key, 300, "NULL");   // 5 min for null
}
```

**Why?** Data might be created later; don't cache "not found" forever.

---

### 2. **Combine Multiple Defenses**

```java
// Layer defense:
1. Input validation (reject obviously invalid)
2. Rate limiting (prevent brute force)
3. Bloom filter (fast negative lookup)
4. Null caching (cache not-found results)
5. Monitoring (detect attacks)
```

---

### 3. **Monitor for Attack Patterns**

```java
@Service
public class PenetrationDetector {
    
    @Scheduled(fixedRate = 60000)  // Every minute
    public void detectAttacks() {
        // Get cache miss rate
        double missRate = getCacheMissRate();
        
        // Get 404 rate
        double notFoundRate = get404Rate();
        
        if (missRate > 0.5 && notFoundRate > 0.3) {
            logger.error("Potential cache penetration attack detected!");
            logger.error("Cache miss rate: {}%, 404 rate: {}%", 
                missRate * 100, notFoundRate * 100);
            
            alerting.triggerAlert("cache_penetration_attack");
        }
    }
}
```

---

### 4. **Limit Null Cache Size**

```java
// Use LRU eviction for null cache to prevent memory exhaustion
RedisCacheConfiguration nullCacheConfig = RedisCacheConfiguration
    .defaultCacheConfig()
    .entryTtl(Duration.ofMinutes(5))
    .disableCachingNullValues(false)  // Allow null caching
    .serializeValuesWith(/* ... */);

// Set maxmemory policy
// redis.conf:
maxmemory-policy allkeys-lru
```

---

### 5. **Log Suspicious Patterns**

```java
public Product getProduct(String productId, HttpServletRequest request) {
    Product product = service.getProduct(productId);
    
    if (product == null) {
        // Log 404 with client info
        logger.warn("Product not found: {} from IP: {}, User-Agent: {}", 
            productId, 
            request.getRemoteAddr(), 
            request.getHeader("User-Agent"));
        
        // Track per-IP 404 count
        metrics.counter("product.not_found", 
            "ip", request.getRemoteAddr()).increment();
    }
    
    return product;
}
```

---

### 6. **Implement Circuit Breaker**

```java
@Service
public class CircuitBreakerProtection {
    private final CircuitBreaker circuitBreaker;
    
    public Product getProduct(String productId) {
        return circuitBreaker.executeSupplier(() -> {
            // If database is being hammered, circuit opens
            // Prevent further DB queries temporarily
            return repository.findById(productId).orElse(null);
        });
    }
}
```

---

## Related Problems

### Cache Stampede

**Different problem**:
- Stampede: Valid data, cache expires, surge of requests
- Penetration: Invalid data, never cached, constant DB load

### Cache Avalanche

**Different problem**:
- Avalanche: Many keys expire simultaneously
- Penetration: Keys never exist, never cached

### Cache Breakdown

**Similar to stampede**:
- Hot key expires → surge
- Penetration: Key never exists → constant load

---

## Key Takeaways

1. **Cache penetration bypasses cache protection** — every request hits database
2. **Can be exploited as DDoS** — intentional attack vector
3. **Cache null results** — simple, effective solution
4. **Bloom filters are powerful** — space-efficient, fast negative lookups
5. **Combine multiple defenses** — layered security approach
6. **Monitor attack patterns** — detect and respond to attacks
7. **Use short TTL for nulls** — prevent stale negative caching
8. **Validate inputs** — reject obviously invalid requests early

---

## Next Steps

- [Cache Stampede Prevention](./cache-stampede.md)
- [Cache Warming](./cache-warming.md)
- [Cache Invalidation](./cache-invalidation.md)
- [Rate Limiting Strategies](../design-patterns/rate-limiting.md)

---

**Last Updated**: 2026-01-17
**Target Audience**: Senior Software Engineers
**Difficulty Level**: Advanced
