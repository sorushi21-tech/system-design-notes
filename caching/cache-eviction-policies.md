# Cache Eviction Policies

## Table of Contents
1. [Introduction](#introduction)
2. [Why Eviction is Needed](#why-eviction-is-needed)
3. [Common Eviction Policies](#common-eviction-policies)
4. [LRU (Least Recently Used)](#lru-least-recently-used)
5. [LFU (Least Frequently Used)](#lfu-least-frequently-used)
6. [FIFO (First In First Out)](#fifo-first-in-first-out)
7. [Random Eviction](#random-eviction)
8. [TTL-Based Eviction](#ttl-based-eviction)
9. [Redis Eviction Policies](#redis-eviction-policies)
10. [Choosing the Right Policy](#choosing-the-right-policy)
11. [Implementation Examples](#implementation-examples)
12. [Best Practices](#best-practices)

---

## Introduction

**Cache eviction** is the process of removing entries from cache when it reaches capacity limits. Since cache memory is finite, eviction policies determine which data to remove to make room for new data.

The choice of eviction policy significantly impacts cache hit rate and application performance.

---

## Why Eviction is Needed

### Memory Constraints

```
Available Memory: 8 GB
Current Cache Size: 7.9 GB
New Item Size: 500 MB
Action: Evict ~500 MB of data to make room
```

### The Fundamental Problem

```
Cache Capacity: Fixed (e.g., 10,000 items)
Potential Items: Unlimited (e.g., 1,000,000 products)
Reality: Must choose which 10,000 items to keep
```

### What Happens Without Eviction?

1. **Out of Memory**: Cache grows until OOM error
2. **Performance Degradation**: Large caches slow down
3. **Unpredictable Behavior**: Random failures

---

## Common Eviction Policies

### Quick Comparison

| Policy | Complexity | Hit Rate | Use Case | Memory Overhead |
|--------|------------|----------|----------|-----------------|
| **LRU** | O(1) | High | General purpose | Low-Medium |
| **LFU** | O(log N) | High | Stable patterns | Medium |
| **FIFO** | O(1) | Medium | Simple, predictable | Low |
| **Random** | O(1) | Low | High performance | Very Low |
| **TTL** | O(1) | N/A | Time-sensitive data | Low |

---

## LRU (Least Recently Used)

### Concept

Evict the item that hasn't been accessed for the longest time.

**Assumption**: Recently used items are more likely to be used again soon (temporal locality).

### How It Works

```
Cache: [A, B, C, D, E]  (Max size: 5)

Access B → [A, C, D, E, B]  (B moved to front)
Access A → [C, D, E, B, A]  (A moved to front)
Add F   → [D, E, B, A, F]   (C evicted - least recently used)
```

### Visual Representation

```
Time →
─────────────────────────────────────────
Access: A  B  C  D  E  B  A  F
              ↑           ↑
              C used   C evicted
              
LRU Order (most → least recent):
Before F: A, B, E, D, C
After F:  F, A, B, E, D  (C removed)
```

---

### Implementation

#### Naive Approach (Inefficient)

```java
// O(N) time complexity - BAD
public class NaiveLRU<K, V> {
    private final Map<K, CacheEntry<V>> map;
    private final int capacity;
    
    static class CacheEntry<V> {
        V value;
        long lastAccessed;
    }
    
    public V get(K key) {
        CacheEntry<V> entry = map.get(key);
        if (entry != null) {
            entry.lastAccessed = System.currentTimeMillis();
            return entry.value;
        }
        return null;
    }
    
    public void put(K key, V value) {
        if (map.size() >= capacity) {
            // O(N) - scan all entries to find LRU
            K lruKey = findLRUKey();
            map.remove(lruKey);
        }
        map.put(key, new CacheEntry<>(value, System.currentTimeMillis()));
    }
}
```

#### Efficient Implementation (LinkedHashMap)

```java
// O(1) time complexity - GOOD
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;
    
    public LRUCache(int capacity) {
        super(capacity, 0.75f, true);  // accessOrder=true
        this.capacity = capacity;
    }
    
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
}

// Usage
LRUCache<String, Product> cache = new LRUCache<>(1000);
cache.put("key1", product1);
Product p = cache.get("key1");  // Marks as recently used
```

#### Custom Implementation (Doubly Linked List + HashMap)

```java
public class LRUCache<K, V> {
    private final int capacity;
    private final Map<K, Node<K, V>> map;
    private final DoublyLinkedList<K, V> list;
    
    static class Node<K, V> {
        K key;
        V value;
        Node<K, V> prev, next;
        
        Node(K key, V value) {
            this.key = key;
            this.value = value;
        }
    }
    
    static class DoublyLinkedList<K, V> {
        Node<K, V> head, tail;
        
        void moveToFront(Node<K, V> node) {
            if (node == head) return;
            remove(node);
            addToFront(node);
        }
        
        void addToFront(Node<K, V> node) {
            node.next = head;
            node.prev = null;
            if (head != null) head.prev = node;
            head = node;
            if (tail == null) tail = node;
        }
        
        void remove(Node<K, V> node) {
            if (node.prev != null) node.prev.next = node.next;
            else head = node.next;
            
            if (node.next != null) node.next.prev = node.prev;
            else tail = node.prev;
        }
        
        Node<K, V> removeLast() {
            if (tail == null) return null;
            Node<K, V> last = tail;
            remove(last);
            return last;
        }
    }
    
    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.map = new HashMap<>();
        this.list = new DoublyLinkedList<>();
    }
    
    public V get(K key) {
        Node<K, V> node = map.get(key);
        if (node == null) return null;
        
        list.moveToFront(node);
        return node.value;
    }
    
    public void put(K key, V value) {
        Node<K, V> node = map.get(key);
        
        if (node != null) {
            node.value = value;
            list.moveToFront(node);
        } else {
            if (map.size() >= capacity) {
                Node<K, V> lru = list.removeLast();
                map.remove(lru.key);
            }
            
            Node<K, V> newNode = new Node<>(key, value);
            map.put(key, newNode);
            list.addToFront(newNode);
        }
    }
}
```

---

### Advantages

✅ **High hit rate** for workloads with temporal locality
✅ **Simple to understand** and reason about
✅ **O(1) operations** with proper data structure
✅ **Industry standard** — well-tested and understood

### Disadvantages

❌ **Memory overhead** for linked list pointers
❌ **Can evict popular items** if not accessed recently
❌ **Scan-resistant issue** — one-time bulk scan evicts hot data

### Use Cases

- **Web applications**: User sessions, page caching
- **API responses**: Recent queries repeated
- **Database query cache**: Repeated queries
- **General purpose caching**: Default choice

---

## LFU (Least Frequently Used)

### Concept

Evict the item accessed the fewest number of times.

**Assumption**: Frequently used items are more valuable than recently used items.

### How It Works

```
Cache: [(A,5), (B,3), (C,10), (D,2), (E,7)]  (Max size: 5)
       (key, frequency)

Access B → [(A,5), (B,4), (C,10), (D,2), (E,7)]  (B frequency++)
Add F    → [(A,5), (B,4), (C,10), (F,1), (E,7)]  (D evicted - lowest freq)
```

---

### Implementation

```java
public class LFUCache<K, V> {
    private final int capacity;
    private final Map<K, CacheNode<K, V>> cache;
    private final Map<Integer, DoublyLinkedList<K, V>> frequencyMap;
    private int minFrequency;
    
    static class CacheNode<K, V> {
        K key;
        V value;
        int frequency;
        
        CacheNode(K key, V value) {
            this.key = key;
            this.value = value;
            this.frequency = 1;
        }
    }
    
    public LFUCache(int capacity) {
        this.capacity = capacity;
        this.cache = new HashMap<>();
        this.frequencyMap = new HashMap<>();
        this.minFrequency = 0;
    }
    
    public V get(K key) {
        CacheNode<K, V> node = cache.get(key);
        if (node == null) return null;
        
        updateFrequency(node);
        return node.value;
    }
    
    public void put(K key, V value) {
        if (capacity == 0) return;
        
        CacheNode<K, V> node = cache.get(key);
        
        if (node != null) {
            node.value = value;
            updateFrequency(node);
        } else {
            if (cache.size() >= capacity) {
                evictLFU();
            }
            
            CacheNode<K, V> newNode = new CacheNode<>(key, value);
            cache.put(key, newNode);
            
            frequencyMap.computeIfAbsent(1, k -> new DoublyLinkedList<>())
                .addToFront(newNode);
            minFrequency = 1;
        }
    }
    
    private void updateFrequency(CacheNode<K, V> node) {
        int freq = node.frequency;
        frequencyMap.get(freq).remove(node);
        
        if (frequencyMap.get(freq).isEmpty() && freq == minFrequency) {
            minFrequency++;
        }
        
        node.frequency++;
        frequencyMap.computeIfAbsent(node.frequency, k -> new DoublyLinkedList<>())
            .addToFront(node);
    }
    
    private void evictLFU() {
        DoublyLinkedList<K, V> list = frequencyMap.get(minFrequency);
        CacheNode<K, V> nodeToEvict = list.removeLast();
        cache.remove(nodeToEvict.key);
    }
}
```

---

### Advantages

✅ **High hit rate** for stable access patterns
✅ **Protects hot data** — popular items stay cached
✅ **Good for long-running caches**

### Disadvantages

❌ **Complex implementation** — more code, harder to debug
❌ **Higher memory overhead** — frequency counters + multiple data structures
❌ **Slow adaptation** — new popular items take time to gain high frequency
❌ **Frequency decay needed** — old popular items may no longer be relevant

### Use Cases

- **Content delivery**: Popular videos, images
- **Static assets**: Frequently accessed files
- **Reference data**: Commonly used lookup tables
- **Long-running caches**: Where access patterns are stable

---

## FIFO (First In First Out)

### Concept

Evict the oldest item (first inserted), regardless of access pattern.

**Simple queue**: Items evicted in the order they were added.

### How It Works

```
Cache: [A, B, C, D, E]  (Max size: 5, insertion order)

Add F → [B, C, D, E, F]  (A evicted - first in)
Add G → [C, D, E, F, G]  (B evicted - first in)
```

---

### Implementation

```java
public class FIFOCache<K, V> {
    private final int capacity;
    private final Map<K, V> cache;
    private final Queue<K> queue;
    
    public FIFOCache(int capacity) {
        this.capacity = capacity;
        this.cache = new HashMap<>();
        this.queue = new LinkedList<>();
    }
    
    public V get(K key) {
        return cache.get(key);
    }
    
    public void put(K key, V value) {
        if (!cache.containsKey(key)) {
            if (cache.size() >= capacity) {
                K oldest = queue.poll();
                cache.remove(oldest);
            }
            queue.offer(key);
        }
        cache.put(key, value);
    }
}
```

---

### Advantages

✅ **Simplest implementation**
✅ **Low memory overhead**
✅ **Predictable behavior**
✅ **O(1) operations**

### Disadvantages

❌ **Poor hit rate** — ignores access patterns
❌ **Evicts potentially hot data**
❌ **Not suitable for most applications**

### Use Cases

- **Logging buffers**: Order matters more than access pattern
- **Event queues**: Process in order
- **When simplicity is paramount**: Embedded systems, simple caches

---

## Random Eviction

### Concept

Evict a random entry when cache is full.

### Implementation

```java
public class RandomEvictionCache<K, V> {
    private final int capacity;
    private final Map<K, V> cache;
    private final Random random;
    
    public RandomEvictionCache(int capacity) {
        this.capacity = capacity;
        this.cache = new HashMap<>();
        this.random = new Random();
    }
    
    public V get(K key) {
        return cache.get(key);
    }
    
    public void put(K key, V value) {
        if (!cache.containsKey(key) && cache.size() >= capacity) {
            K[] keys = (K[]) cache.keySet().toArray();
            K randomKey = keys[random.nextInt(keys.length)];
            cache.remove(randomKey);
        }
        cache.put(key, value);
    }
}
```

---

### Advantages

✅ **Simplest eviction logic**
✅ **No overhead** for tracking access
✅ **Fast** — O(1) eviction

### Disadvantages

❌ **Worst hit rate** — completely random
❌ **Unpredictable** — may evict hot data
❌ **Rarely used in practice**

### Use Cases

- **Academic interest** — baseline for comparison
- **Extremely simple systems** — when hit rate doesn't matter
- **Randomized algorithms** — when randomness is desired

---

## TTL-Based Eviction

### Concept

Items expire after a fixed time period, regardless of usage.

### Implementation

```java
public class TTLCache<K, V> {
    private final Map<K, CacheEntry<V>> cache;
    
    static class CacheEntry<V> {
        V value;
        long expiryTime;
        
        CacheEntry(V value, long ttlMillis) {
            this.value = value;
            this.expiryTime = System.currentTimeMillis() + ttlMillis;
        }
        
        boolean isExpired() {
            return System.currentTimeMillis() > expiryTime;
        }
    }
    
    public V get(K key) {
        CacheEntry<V> entry = cache.get(key);
        if (entry == null || entry.isExpired()) {
            cache.remove(key);
            return null;
        }
        return entry.value;
    }
    
    public void put(K key, V value, long ttlMillis) {
        cache.put(key, new CacheEntry<>(value, ttlMillis));
    }
    
    // Background cleanup
    @Scheduled(fixedRate = 60000)  // Every minute
    public void cleanup() {
        cache.entrySet().removeIf(entry -> entry.getValue().isExpired());
    }
}
```

---

### Advantages

✅ **Time-based consistency** — data freshness guaranteed
✅ **Automatic cleanup** — expired data removed
✅ **Simple logic**

### Disadvantages

❌ **Not capacity-based** — cache can grow unbounded without size limit
❌ **Memory leaks** — if cleanup doesn't run
❌ **May evict hot data** — if TTL expires

### Use Cases

- **Session management**: 30-minute session timeout
- **API rate limiting**: Reset after time window
- **Temporary data**: OTPs, verification codes

---

## Redis Eviction Policies

Redis provides several built-in eviction policies.

### Configuration

```conf
# redis.conf
maxmemory 2gb
maxmemory-policy allkeys-lru
```

### Available Policies

#### 1. **noeviction**
```conf
maxmemory-policy noeviction
```
- Return errors when memory limit reached
- Don't evict any keys
- Use when: Data loss is unacceptable

#### 2. **allkeys-lru** ✅ Recommended for Cache
```conf
maxmemory-policy allkeys-lru
```
- Evict any key using LRU
- Use when: Redis is pure cache

#### 3. **volatile-lru**
```conf
maxmemory-policy volatile-lru
```
- Evict keys with TTL using LRU
- Use when: Mix of cache and persistent data

#### 4. **allkeys-lfu** (Redis 4.0+)
```conf
maxmemory-policy allkeys-lfu
```
- Evict any key using LFU
- Use when: Stable access patterns, protect hot data

#### 5. **volatile-lfu**
```conf
maxmemory-policy volatile-lfu
```
- Evict keys with TTL using LFU

#### 6. **allkeys-random**
```conf
maxmemory-policy allkeys-random
```
- Evict random key
- Use when: All keys equally important (rare)

#### 7. **volatile-random**
```conf
maxmemory-policy volatile-random
```
- Evict random key with TTL

#### 8. **volatile-ttl**
```conf
maxmemory-policy volatile-ttl
```
- Evict key with shortest TTL
- Use when: TTL indicates priority

---

### Redis LRU Approximation

Redis uses **approximated LRU** for efficiency:

```conf
# Sample 5 keys and evict least recently used among them
maxmemory-samples 5  # Default
```

**Trade-off**:
- Lower samples (3) = Faster but less accurate
- Higher samples (10) = Slower but more accurate

---

## Choosing the Right Policy

### Decision Tree

```
Q: What is your primary use case?
├─ Pure Cache
│  ├─ Q: Access pattern?
│  │  ├─ Temporal (recent items reused) → LRU ✅
│  │  ├─ Frequency (popular items reused) → LFU
│  │  └─ Unknown → LRU (safe default)
│  └─ Redis: allkeys-lru or allkeys-lfu
│
├─ Mixed (Cache + Persistent)
│  └─ Redis: volatile-lru or volatile-lfu
│
├─ Time-Sensitive Data
│  └─ TTL-based or volatile-ttl
│
└─ Simple/Learning
   └─ FIFO or Random
```

### Workload Characteristics

| Workload | Best Policy | Reason |
|----------|-------------|--------|
| Web sessions | LRU | Recent users likely to return |
| Video streaming | LFU | Popular content stays hot |
| API responses | LRU | Recent queries repeated |
| Static assets | LFU | Popular files accessed often |
| Search results | LRU | Recent searches similar |
| Product catalog | LRU | Seasonal, trending items |
| User profiles | LRU | Active users access frequently |

---

## Best Practices

### 1. **Monitor Eviction Rates**

```java
@Scheduled(fixedRate = 60000)
public void monitorEvictions() {
    long evictions = cache.stats().evictionCount();
    long hitRate = cache.stats().hitRate();
    
    if (evictions > threshold) {
        logger.warn("High eviction rate: {}/min", evictions);
    }
    
    metrics.gauge("cache.evictions", evictions);
    metrics.gauge("cache.hitRate", hitRate);
}
```

**High eviction rate** → Cache too small or poor policy choice

### 2. **Combine Eviction with TTL**

```java
// Use both size-based and time-based eviction
Cache<String, Product> cache = Caffeine.newBuilder()
    .maximumSize(10000)                     // Size-based (LRU)
    .expireAfterWrite(1, TimeUnit.HOURS)    // Time-based (TTL)
    .build();
```

### 3. **Size Your Cache Appropriately**

```
Working Set Size: 100,000 items
Cache Size: 20,000 items (20%)

If hit rate < 80%:
- Increase cache size OR
- Improve eviction policy OR
- Analyze access patterns
```

### 4. **Test Different Policies**

```java
// A/B test eviction policies
@Configuration
public class CacheConfig {
    
    @Bean
    @Profile("lru")
    public CacheManager lruCacheManager() {
        // LRU configuration
    }
    
    @Bean
    @Profile("lfu")
    public CacheManager lfuCacheManager() {
        // LFU configuration
    }
}
```

Compare hit rates, latency, and eviction rates.

### 5. **Consider Multi-Tier Eviction**

```
L1 (Local): Small, aggressive LRU (5 min)
L2 (Redis): Large, conservative LRU (1 hour)
```

---

## Key Takeaways

1. **LRU is the default choice** — good hit rate, simple, efficient
2. **LFU for stable patterns** — protects truly popular data
3. **Always set maxmemory** — prevent unbounded growth
4. **Combine size + TTL eviction** — defense in depth
5. **Monitor eviction rates** — high evictions = undersized cache
6. **Redis uses approximated LRU** — adjust samples for accuracy
7. **Test in production** — real traffic reveals best policy
8. **No one-size-fits-all** — choose based on access patterns

---

## Next Steps

- [Cache Stampede Prevention](./cache-stampede.md)
- [Cache Warming Strategies](./cache-warming.md)
- [Redis Basics](./redis-basics.md)

---

**Last Updated**: 2026-01-17
**Target Audience**: Senior Software Engineers
**Difficulty Level**: Intermediate to Advanced
