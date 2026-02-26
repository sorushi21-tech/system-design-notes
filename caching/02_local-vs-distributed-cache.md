# Local vs Distributed Cache

---

## 1. The Main Question: Where Should Cache Data Live?

When we design a caching system, the first big decision is — **where should we keep the cached data?**

We have three choices:
- **Local Cache (L1):** Data is stored inside each application server's own memory.
- **Distributed Cache (L2):** Data is stored in a separate, shared cache server (like Redis).
- **Both Together (Multi-Level):** Use local cache AND distributed cache at the same time.


This decision affects four important things:
- **Performance** — How fast will data be served?
- **Consistency** — Will all servers see the same data?
- **Scalability** — Will it handle more users as system grows?
- **Cost** — How much will infrastructure cost?

---

## 2. Local Cache (L1 Cache)

### What is it?

Local cache stores data **inside the application's own memory**. Each application instance (server) has its own separate cache. They do NOT share data with each other.

**Key Point:** If you have 3 servers, each server has its own copy of cached data. They are independent.

### Characteristics

| Aspect         | Details                                          |
|----------------|--------------------------------------------------|
| Where it lives | Inside application memory (same process)         |
| Scope          | Only that one server                             |
| Speed          | Ultra-fast (0.1 to 1 microsecond)                |
| Network needed | No — direct memory access                        |
| Consistency    | Each server has its own cache — can be different |

---

### Advantages of Local Cache

**1. Extremely Fast**
Direct memory access — no network call needed. Typically less than 100 nanoseconds. That is 10 to 100 times faster than distributed cache.

**2. No Network Overhead**
Zero network latency. No need to serialize or deserialize data. No connection pooling to manage.

**3. Very High Throughput**
Can handle millions of requests per second. Only limited by CPU and memory speed.

**4. Simple to Set Up**
Easy to configure with size limits and TTL. No external server needed.

**5. No Extra Infrastructure**
Uses existing application memory. No Redis or Memcached to install and manage. Lower cost and simpler deployment.

---

### Disadvantages of Local Cache

**1. Limited Memory**
Cache is limited to the application server's RAM. It competes with the application itself for memory.

**2. Inconsistency Between Servers**


**3. Cache Warm-Up on Every Restart**
Each server starts with empty cache after restart or new deployment. All servers have to slowly warm up again from scratch.

**4. Same Data Copied Multiple Times**
If 3 servers all cache the same product data, that data is stored 3 times. Wasteful memory usage.

**5. Hard to Invalidate Across All Servers**

---

### Popular Local Cache Tools

- **Java:** Caffeine, Guava Cache, Ehcache

---

## 3. Distributed Cache (L2 Cache)

### What is it?

Distributed cache is a **separate server** that all application instances share.

**Key Point:** All servers see the SAME data. When one server updates a key, all others also see the updated value.

### Characteristics

| Aspect         | Details                                                   |
|----------------|-----------------------------------------------------------|
| Where it lives | External server (Redis, Memcached)                        |
| Scope          | Shared across ALL application servers                     |
| Speed          | Fast but not ultra-fast (1 to 5 milliseconds via network) |
| Network needed | Yes — adds some latency                                   |
| Consistency    | Single source of truth — all servers see same data        |

---

### Advantages of Distributed Cache

**1. All Servers Share the Same Data**
One server fetches and caches data — all other servers also benefit. No duplication. Memory is used efficiently.

**2. Consistency — Everyone Sees Same Data**
No stale data problem across servers. Invalidation works globally — delete a key and it is gone for everyone.

**3. Much Larger Storage**
Not limited by one server's memory. Can cache GBs or even TBs of data. Cache and application can be scaled separately.

**4. Survives Application Restarts**
Redis supports persistence (saves data to disk). No cold start problem — cache is still warm even after app restart.

**5. Extra Features (especially Redis)**
- Complex data structures: lists, sets, sorted sets, hashmaps.
- Pub/Sub messaging.
- Distributed locking.
- Atomic operations.

**6. Centralized Monitoring**
All cache metrics in one place. Easy to see hit rate, memory usage, error rate.

---

### Disadvantages of Distributed Cache

**1. Network Latency**
Every cache read/write goes over the network. 1 to 5 ms per request (vs nanoseconds for local cache). Serialization and deserialization also adds overhead.

**2. Extra Infrastructure to Manage**

**3. Single Point of Failure**
If the cache server goes down, all application servers are affected

**4. Serialization Overhead**
Data must be converted to bytes before sending over network (serialize), and converted back after receiving (deserialize). This takes extra time.

**5. Connection Management**
Need to maintain connection pool. Too many connections can hit Redis limits. Must handle connection failures gracefully.

---

## 4. Comparison: Local vs Distributed

| Aspect          | Local Cache (L1)         | Distributed Cache (L2) |
|-----------------|--------------------------|------------------------|
| Speed/Latency   | 0.1–1 microsecond        | 1–5 milliseconds       |
| Throughput      | Millions/sec             | 100K–1M/sec            |
| Network needed  | No                       | Yes                    |
| Consistency     | Per-server (can differ)  | Shared (all same)      |
| Memory Capacity | Limited (MBs)            | Scalable (GBs)         |
| Complexity      | Low                      | Medium-High            |
| Cost            | Free (uses existing RAM) | Extra infra cost       |
| Invalidation    | Per-server (hard)        | Global (easy)          |
| Cold Start      | Every restart            | Persistent option      |
| Data Sharing    | No                       | Yes                    |

**Simple summary:**
- **Local Cache wins** on: Speed, Simplicity, Cost.
- **Distributed Cache wins** on: Consistency, Storage Capacity, Sharing.

---

## 5. When to Use Which?

### Use Local Cache When:

- Data is **read very frequently** but **changes rarely** — like configuration settings, country/state lists, feature flags, currency exchange rates.
- When we need **sub-millisecond latency** — thousands of reads per second.
- **Dataset is small** — can easily fit in application memory (MBs, not GBs).

---

### Use Distributed Cache When:

- When we need **same data visible to all servers**
- **Dataset is large** — does not fit in one server's memory.
- **Multiple services** need the same cached data (microservices architecture).
- Data **changes frequently** and we to need invalidation to affect all servers immediately.
- When we need **cache to survive app restarts**.

---

## 6. Multi-Level Caching — Best of Both Worlds (L1 + L2)

### What is it?

We use BOTH local cache and distributed cache together. This gives us ultra-fast speed from local cache AND consistency/sharing from distributed cache.

### How it works — Step by Step

**When reading data:**

```
Step 1: Check L1 (Local Cache)
        → HIT? Return immediately. Ultra fast (0.1 ms).

Step 2: Check L2 (Distributed Cache / Redis)
        → HIT? Save to L1 also. Return. Fast (1–5 ms).

Step 3: Go to Database
        → Save to L2 and L1 both. Return. Slow (50–100 ms).
```

**Performance levels:**
- **L1 Hit** → 0.1 to 1 microsecond ⚡⚡⚡
- **L2 Hit** → 1 to 5 milliseconds ⚡⚡
- **DB Query** → 50 to 100 milliseconds ⚡


---

### The Invalidation Problem in Multi-Level Cache

**Problem:** If Server 1 updates data and clears its L1 and L2 cache, Servers 2 and 3 still have old data in their L1 cache!

**Solution: Use Redis Pub/Sub**

```
When Server 1 updates a key:
  1. Clear own L1 cache
  2. Delete from Redis (L2)
  3. Publish message: "cache:invalidate" → "user:123"

Servers 2 and 3 are subscribed to this channel:
  → They receive the message
  → They also clear "user:123" from their own L1 cache

Result: All servers' L1 caches are now in sync!
```

---

## 7. Common Mistakes to Avoid

**Mistake 1: Caching too much in Local Cache**
Storing too much data locally causes OutOfMemory errors. Always set a maximum size limit on local cache.

**Mistake 2: Not handling Redis failure**
If Redis goes down and your app crashes — that is bad design. Always have a fallback to database when Redis is unavailable.

```
try:
    return redis.get(key)
catch RedisError:
    log("Redis down, using database as fallback")
    return database.get(key)
```

**Mistake 3: Forgetting TTL on distributed cache**
Without TTL, cache grows forever and never cleans up. Always set TTL on every key.

**Mistake 4: Slow serialization**
Using slow serialization formats like JSON for very large objects adds significant overhead. Consider Protocol Buffers or MessagePack for better performance.

**Mistake 5: Cache Stampede on L2 Miss**
When a popular key expires from L2, all servers hit the database at once. Use distributed locking or single-flight pattern so only one server loads data and others wait.

---