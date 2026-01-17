# Write-Behind (Write-Back) Cache Pattern

## Table of Contents
1. [Introduction](#introduction)
2. [Pattern Overview](#pattern-overview)
3. [How It Works](#how-it-works)
4. [Implementation](#implementation)
5. [Advantages](#advantages)
6. [Disadvantages](#disadvantages)
7. [Use Cases](#use-cases)
8. [Best Practices](#best-practices)
9. [Production Considerations](#production-considerations)

---

## Introduction

**Write-Behind** (also called **Write-Back**) is a caching pattern where writes are immediately stored in the cache and asynchronously written to the database later. This provides extremely fast write operations but introduces complexity and risk of data loss.

---

## Pattern Overview

### Architecture

```
┌──────────────┐
│ Application  │
└──────┬───────┘
       │
       │ Write (fast)
       ▼
   ┌────────┐
   │ Cache  │ ← Write updates cache immediately
   └────┬───┘
        │
        │ Asynchronous write (later)
        ▼
   ┌──────────┐
   │ Database │ ← Eventually updated
   └──────────┘
```

### Key Characteristics
- **Asynchronous writes**: Database updated in background
- **Fast writes**: Sub-millisecond response time
- **Eventual consistency**: Database lags behind cache
- **Risk of data loss**: If cache fails before DB write

---

## How It Works

### Write Flow

```
1. Application issues write request
2. Update cache immediately
3. Return success to client (fast!)
4. Queue write for later database update
5. Background worker writes to database
```

### Timeline

```
T=0ms:    Client: UPDATE product
T=1ms:    Cache: Updated ✓
T=2ms:    Client: Receives success
T=100ms:  Queue: Write operation queued
T=500ms:  Worker: Processes queue
T=550ms:  Database: Updated ✓
```

**Client latency**: 2ms (cache only)
**vs Write-Through**: 52ms (cache + database)

---

## Implementation

### Basic Implementation (Java)

```java
@Service
public class WriteBehindCacheService {
    @Autowired private JedisPool redisPool;
    @Autowired private ProductRepository repository;
    @Autowired private BlockingQueue<WriteOperation> writeQueue;
    
    private static final int TTL = 3600;
    
    // Write operation
    public Product updateProduct(Product product) {
        String key = "product:" + product.getId();
        
        // 1. Update cache immediately
        try (Jedis jedis = redisPool.getResource()) {
            String json = objectMapper.writeValueAsString(product);
            jedis.setex(key, TTL, json);
        }
        
        // 2. Queue for async database write
        writeQueue.offer(new WriteOperation(product.getId(), product));
        
        // 3. Return immediately (fast!)
        return product;
    }
    
    // Background worker
    @Scheduled(fixedDelay = 100)  // Every 100ms
    public void processWriteQueue() {
        List<WriteOperation> batch = new ArrayList<>();
        writeQueue.drainTo(batch, 100);  // Batch up to 100 operations
        
        if (!batch.isEmpty()) {
            // Batch write to database
            List<Product> products = batch.stream()
                .map(WriteOperation::getProduct)
                .collect(Collectors.toList());
            
            try {
                repository.saveAll(products);
                logger.info("Flushed {} writes to database", batch.size());
            } catch (Exception e) {
                logger.error("Failed to flush writes to database", e);
                // Re-queue failed writes
                writeQueue.addAll(batch);
            }
        }
    }
    
    static class WriteOperation {
        private final String id;
        private final Product product;
        private final Instant timestamp;
        
        WriteOperation(String id, Product product) {
            this.id = id;
            this.product = product;
            this.timestamp = Instant.now();
        }
        
        // Getters
    }
}
```

---

### Advanced Implementation with Redis + Periodic Flush

```java
@Service
public class AdvancedWriteBehindCache {
    @Autowired private JedisPool redisPool;
    @Autowired private ProductRepository repository;
    
    private static final String DIRTY_SET = "cache:dirty:products";
    private static final int CACHE_TTL = 3600;
    private static final int FLUSH_INTERVAL_MS = 5000;  // 5 seconds
    
    public Product updateProduct(Product product) {
        String key = "product:" + product.getId();
        
        try (Jedis jedis = redisPool.getResource()) {
            // 1. Update cache
            String json = objectMapper.writeValueAsString(product);
            jedis.setex(key, CACHE_TTL, json);
            
            // 2. Mark as dirty (needs DB write)
            jedis.sadd(DIRTY_SET, product.getId());
            
            // 3. Return immediately
            return product;
        }
    }
    
    @Scheduled(fixedRate = FLUSH_INTERVAL_MS)
    public void flushDirtyData() {
        try (Jedis jedis = redisPool.getResource()) {
            // Get all dirty product IDs
            Set<String> dirtyIds = jedis.smembers(DIRTY_SET);
            
            if (dirtyIds.isEmpty()) {
                return;
            }
            
            logger.info("Flushing {} dirty products to database", dirtyIds.size());
            
            // Retrieve products from cache
            List<Product> products = new ArrayList<>();
            for (String id : dirtyIds) {
                String key = "product:" + id;
                String json = jedis.get(key);
                if (json != null) {
                    products.add(objectMapper.readValue(json, Product.class));
                }
            }
            
            // Batch write to database
            if (!products.isEmpty()) {
                try {
                    repository.saveAll(products);
                    
                    // Clear dirty flag
                    jedis.del(DIRTY_SET);
                    
                    logger.info("Successfully flushed {} products", products.size());
                } catch (Exception e) {
                    logger.error("Failed to flush products to database", e);
                    // Dirty set remains, will retry next interval
                }
            }
        } catch (Exception e) {
            logger.error("Error in flush cycle", e);
        }
    }
    
    // Graceful shutdown - flush remaining data
    @PreDestroy
    public void shutdown() {
        logger.info("Shutting down, flushing remaining dirty data...");
        flushDirtyData();
    }
}
```

---

### Write-Behind with Write Coalescing

```java
@Service
public class CoalescingWriteBehindCache {
    private final Map<String, Product> writeBuffer = new ConcurrentHashMap<>();
    private final ScheduledExecutorService scheduler = 
        Executors.newScheduledThreadPool(1);
    
    public CoalescingWriteBehindCache() {
        // Flush every 5 seconds
        scheduler.scheduleAtFixedRate(
            this::flush, 
            5, 5, 
            TimeUnit.SECONDS
        );
    }
    
    public Product updateProduct(Product product) {
        String key = "product:" + product.getId();
        
        // 1. Update cache
        redis.setex(key, 3600, serialize(product));
        
        // 2. Add to write buffer (coalesces multiple updates)
        writeBuffer.put(product.getId(), product);
        
        // 3. Return immediately
        return product;
    }
    
    private void flush() {
        if (writeBuffer.isEmpty()) {
            return;
        }
        
        // Get snapshot and clear buffer
        Map<String, Product> snapshot = new HashMap<>(writeBuffer);
        writeBuffer.clear();
        
        // Batch write to database
        try {
            List<Product> products = new ArrayList<>(snapshot.values());
            repository.saveAll(products);
            
            logger.info("Flushed {} coalesced writes", products.size());
        } catch (Exception e) {
            logger.error("Flush failed, re-adding to buffer", e);
            // Re-add failed writes
            writeBuffer.putAll(snapshot);
        }
    }
    
    @PreDestroy
    public void shutdown() {
        scheduler.shutdown();
        try {
            if (!scheduler.awaitTermination(10, TimeUnit.SECONDS)) {
                scheduler.shutdownNow();
            }
            flush();  // Final flush
        } catch (InterruptedException e) {
            scheduler.shutdownNow();
        }
    }
}
```

**Write Coalescing Example**:
```
T=0s:  Update product:123 (name="A")
T=1s:  Update product:123 (name="B")
T=2s:  Update product:123 (name="C")
T=5s:  Flush: Only 1 DB write (name="C")

Result: 3 cache updates, 1 database write
```

---

## Advantages

### 1. **Extremely Fast Writes**

```java
Write-Behind:   1-2ms  (cache only)
Write-Through:  50ms   (cache + database)
Cache-Aside:    51ms   (database + invalidate)

Speedup: 25-50x faster
```

### 2. **Write Coalescing**

```java
Scenario: Update same product 100 times in 5 seconds

Without coalescing:
- 100 cache writes
- 100 database writes

With coalescing:
- 100 cache writes
- 1 database write (latest version)

Database write reduction: 99%
```

### 3. **Reduced Database Load**

```java
// Batching reduces database load
Before: 1000 individual writes
After:  10 batch writes (100 items each)

Connection overhead reduced
Transaction overhead reduced
```

### 4. **High Write Throughput**

```java
// Can handle massive write spikes
Writes/second: 100,000+ (to cache)
vs
Database limit: 10,000/sec

Result: 10x higher write throughput
```

### 5. **Better User Experience**

```java
// User sees instant response
User clicks "Save" → 2ms response → Happy user

vs

Write-Through: "Save" → 50ms → Acceptable
Direct DB: "Save" → 100ms → Noticeable
```

---

## Disadvantages

### 1. **Risk of Data Loss**

```java
Scenario:
T=0s:   Update cache (product:123)
T=1s:   Queue write operation
T=2s:   CACHE SERVER CRASHES ← Data lost!
T=3s:   Database never updated

Result: User's update lost permanently
```

**Mitigation**: Redis persistence (RDB/AOF), replication

### 2. **Eventual Consistency**

```java
T=0s:   Update product:123 in cache
T=1s:   Client reads from cache → New value
T=2s:   Another service reads from database → Old value
T=5s:   Database updated

Window of inconsistency: 0-5 seconds
```

### 3. **Complex Failure Handling**

```java
// What if database write fails?
try {
    repository.saveAll(products);
} catch (DatabaseException e) {
    // Options:
    // 1. Retry → How many times?
    // 2. Alert → Manual intervention?
    // 3. Dead letter queue → Process later?
    // 4. Give up → Data loss?
}
```

### 4. **Ordering Issues**

```java
// Without careful handling:
T=0s:   Update product:123 (price=100)
T=1s:   Update product:123 (price=200)
T=2s:   Worker 1: Writes price=100 to DB
T=3s:   Worker 2: Writes price=200 to DB
T=4s:   Worker 1: Writes price=100 to DB ← Wrong!

Result: Out-of-order writes
```

### 5. **Difficult to Debug**

```java
// Asynchronous writes = harder to trace
User reports: "My update didn't save"

Questions:
- Did it reach cache? Yes
- Did it reach database? No
- Why not? Worker error? Queue full? Crash?
- When did it fail? Minutes ago? Need logs
```

---

## Use Cases

### ✅ Good Fit

**1. Write-Heavy Workloads**
```java
Example: Analytics, event logging

Writes/sec: 100,000
Reads/sec: 1,000

Benefit: Cache absorbs write load
```

**2. Time-Series Data**
```java
Example: IoT sensor data, metrics

Data points/sec: 10,000
Database: Can't handle write rate

Solution: Write-behind batches writes
```

**3. Session Management**
```java
Example: User session updates

Updates: Every page view (high frequency)
Reads: Occasional

Benefit: Fast session updates
```

**4. Gaming Leaderboards**
```java
Example: Real-time score updates

Updates: Thousands per second
Database: Persist periodically

Benefit: Low latency updates
```

**5. Counters and Aggregations**
```java
Example: Page view counters

Increment: 10,000 times/minute
Database write: Every 5 minutes (batched)

Result: 10,000 cache ops → 1 DB write
```

---

### ❌ Poor Fit

**1. Financial Transactions**
```java
Example: Payment processing

Requirement: Zero data loss
Write-Behind: Risk of data loss

Verdict: Use write-through or direct DB
```

**2. Critical Data**
```java
Example: Medical records, legal documents

Requirement: Immediate persistence
Write-Behind: Delayed persistence

Verdict: Not suitable
```

**3. Strong Consistency Requirements**
```java
Example: Inventory management

Requirement: Database must be immediately updated
Write-Behind: Eventual consistency

Verdict: Use write-through
```

**4. Audit/Compliance Data**
```java
Example: Access logs, audit trails

Requirement: Guaranteed persistence
Write-Behind: Risk of loss

Verdict: Write directly to database
```

---

## Best Practices

### 1. **Use Redis Persistence**

```conf
# redis.conf
appendonly yes
appendfsync everysec

# Enable both RDB and AOF
save 900 1
save 300 10
save 60 10000
```

Protects against cache data loss.

### 2. **Implement Dead Letter Queue**

```java
@Service
public class WriteBehindWithDLQ {
    @Autowired private BlockingQueue<WriteOperation> writeQueue;
    @Autowired private BlockingQueue<WriteOperation> deadLetterQueue;
    
    private static final int MAX_RETRIES = 3;
    
    @Scheduled(fixedDelay = 100)
    public void processQueue() {
        List<WriteOperation> operations = new ArrayList<>();
        writeQueue.drainTo(operations, 100);
        
        for (WriteOperation op : operations) {
            try {
                repository.save(op.getProduct());
            } catch (Exception e) {
                op.incrementRetries();
                
                if (op.getRetries() < MAX_RETRIES) {
                    writeQueue.offer(op);  // Retry
                } else {
                    deadLetterQueue.offer(op);  // Give up, DLQ
                    logger.error("Write failed after {} retries: {}", 
                        MAX_RETRIES, op.getId());
                }
            }
        }
    }
}
```

### 3. **Monitor Queue Depth**

```java
@Scheduled(fixedRate = 10000)  // Every 10 seconds
public void monitorQueue() {
    int queueSize = writeQueue.size();
    
    metrics.gauge("write_behind.queue.size", queueSize);
    
    if (queueSize > WARNING_THRESHOLD) {
        logger.warn("Write queue backing up: {} items", queueSize);
    }
    
    if (queueSize > CRITICAL_THRESHOLD) {
        logger.error("Write queue critical: {} items", queueSize);
        alerting.triggerAlert("write_queue_critical");
    }
}
```

### 4. **Graceful Shutdown**

```java
@Component
public class GracefulShutdown {
    @Autowired private WriteBehindCacheService cacheService;
    
    @PreDestroy
    public void onShutdown() {
        logger.info("Graceful shutdown initiated");
        
        // Stop accepting new writes
        cacheService.stopAcceptingWrites();
        
        // Flush remaining queue
        int remaining = cacheService.flushAll();
        
        logger.info("Flushed {} remaining writes", remaining);
    }
}
```

### 5. **Timestamp/Version Tracking**

```java
public class WriteOperation {
    private String id;
    private Product product;
    private long version;
    private Instant timestamp;
    
    public boolean isNewerThan(WriteOperation other) {
        return this.version > other.version;
    }
}

// Only apply if newer
if (incomingWrite.isNewerThan(currentWrite)) {
    repository.save(incomingWrite.getProduct());
}
```

### 6. **Batch Size Tuning**

```java
@Configuration
public class WriteBehindConfig {
    
    @Value("${write-behind.batch-size:100}")
    private int batchSize;
    
    @Value("${write-behind.flush-interval-ms:5000}")
    private int flushInterval;
    
    // Tune based on:
    // - Database capacity
    // - Write latency SLA
    // - Memory constraints
}
```

### 7. **Circuit Breaker for Database**

```java
@Service
public class ResilientWriteBehind {
    @Autowired private CircuitBreaker dbCircuitBreaker;
    
    @Scheduled(fixedDelay = 5000)
    public void flush() {
        List<WriteOperation> operations = getOperations();
        
        dbCircuitBreaker.run(() -> {
            repository.saveAll(operations);
        }, () -> {
            // Circuit open - database down
            logger.error("Database unavailable, requeueing writes");
            requeueOperations(operations);
        });
    }
}
```

---

## Production Considerations

### Monitoring

```java
// Essential metrics
metrics.gauge("write_behind.queue.size");
metrics.counter("write_behind.writes.success");
metrics.counter("write_behind.writes.failure");
metrics.histogram("write_behind.flush.duration");
metrics.histogram("write_behind.batch.size");
metrics.counter("write_behind.data_loss.events");
```

### Alerting

```java
// Alert conditions
- Queue depth > 1000 (backing up)
- Queue depth > 10000 (critical)
- Flush failure rate > 1%
- Flush duration > 10 seconds
- Dead letter queue not empty
```

### Capacity Planning

```java
// Calculate required queue capacity
Write rate: 10,000 writes/sec
Flush interval: 5 seconds
Required capacity: 10,000 × 5 = 50,000 items

Add safety factor: 50,000 × 2 = 100,000
```

### Testing

```java
// Must test:
1. Normal operation
2. Cache failure scenarios
3. Database failure scenarios
4. High write load
5. Graceful shutdown
6. Recovery from crash

// Chaos engineering
- Kill cache server mid-write
- Disconnect database
- Fill queue to capacity
```

---

## Key Takeaways

1. **Write-Behind provides extremely fast writes** — 25-50x faster than write-through
2. **Risk of data loss** — cache crash before database write
3. **Eventual consistency** — database lags behind cache
4. **Best for write-heavy workloads** — analytics, logging, counters
5. **Use Redis persistence** — RDB + AOF for durability
6. **Implement graceful shutdown** — flush queue before exit
7. **Monitor queue depth** — detect backlog early
8. **Not suitable for critical data** — financial, medical, compliance

---

## Next Steps

- [Write-Through Pattern](./write-through.md)
- [Cache-Aside Pattern](./cache-aside.md)
- [Cache Invalidation](./cache-invalidation.md)
- [Redis Basics](./redis-basics.md)

---

**Last Updated**: 2026-01-17
**Target Audience**: Senior Software Engineers
**Difficulty Level**: Advanced
