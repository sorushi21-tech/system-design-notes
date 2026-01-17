# Redis Basics for System Design

## Table of Contents
1. [Introduction](#introduction)
2. [What is Redis?](#what-is-redis)
3. [Core Features](#core-features)
4. [Data Structures](#data-structures)
5. [Persistence Options](#persistence-options)
6. [Redis Architecture](#redis-architecture)
7. [Replication and High Availability](#replication-and-high-availability)
8. [Performance Characteristics](#performance-characteristics)
9. [Common Use Cases](#common-use-cases)
10. [Best Practices](#best-practices)
11. [Redis vs Memcached](#redis-vs-memcached)

---

## Introduction

Redis (Remote Dictionary Server) is the de facto standard for distributed caching and in-memory data storage in modern system design. Understanding Redis is crucial for senior engineers designing scalable systems.

---

## What is Redis?

**Redis** is an open-source, in-memory data structure store used as:
- **Cache**: Fast data access layer
- **Database**: Primary data store for certain use cases
- **Message Broker**: Pub/Sub and streaming
- **Session Store**: Distributed session management

### Key Characteristics
- **In-Memory**: All data stored in RAM for speed
- **Persistent**: Optional disk persistence (unlike pure caches)
- **Single-Threaded**: Simplified concurrency model (mostly)
- **Atomic Operations**: All operations are atomic
- **Rich Data Types**: Beyond simple key-value

### Performance
- **Throughput**: 100,000+ operations/second on a single instance
- **Latency**: Sub-millisecond (typically <1ms)
- **Memory**: Limited by available RAM (can use disk as overflow)

---

## Core Features

### 1. In-Memory Storage
```
Speed Comparison:
- RAM access: ~100 nanoseconds
- SSD access: ~100 microseconds (1000x slower)
- HDD access: ~10 milliseconds (100,000x slower)
```

### 2. Key-Value Store
```redis
SET user:1000:name "John Doe"
GET user:1000:name
> "John Doe"
```

### 3. Expiration (TTL)
```redis
SET session:abc123 "user_data" EX 3600  # Expires in 1 hour
TTL session:abc123
> 3600  # Seconds remaining
```

### 4. Atomic Operations
All commands execute atomically — no partial execution or race conditions.

### 5. Pub/Sub Messaging
```redis
# Publisher
PUBLISH news:tech "Breaking: New Redis version released"

# Subscriber
SUBSCRIBE news:tech
```

### 6. Transactions (Limited)
```redis
MULTI
SET account:1 balance 1000
SET account:2 balance 2000
EXEC
```

### 7. Lua Scripting
Execute complex operations atomically:
```lua
-- Atomic increment with limit
EVAL "
  local current = redis.call('GET', KEYS[1]) or 0
  if tonumber(current) < tonumber(ARGV[1]) then
    return redis.call('INCR', KEYS[1])
  else
    return 0
  end
" 1 counter:requests 1000
```

---

## Data Structures

Redis provides multiple data structures, making it more than a simple cache.

### 1. Strings (Simple Key-Value)
```redis
SET key "value"
GET key
INCR counter
INCRBY counter 5
APPEND message " world"
```

**Use Cases**:
- Simple caching
- Counters (views, likes)
- Session tokens
- Configuration values

**Size**: Up to 512 MB per value

---

### 2. Hashes (Objects/Maps)
Store field-value pairs — perfect for objects.

```redis
HSET user:1000 name "Alice" email "alice@example.com" age 30
HGET user:1000 name
> "Alice"

HGETALL user:1000
> 1) "name"
> 2) "Alice"
> 3) "email"
> 4) "alice@example.com"
> 5) "age"
> 6) "30"

HINCRBY user:1000 age 1  # Atomic increment
```

**Use Cases**:
- User profiles
- Product details
- Configuration objects
- Shopping carts

**Advantages**:
- More memory efficient than multiple strings
- Partial updates (single field)
- Atomic field operations

---

### 3. Lists (Ordered Collections)
Doubly-linked lists for fast head/tail operations.

```redis
LPUSH queue:tasks "task1"  # Push to head
RPUSH queue:tasks "task2"  # Push to tail
LPOP queue:tasks            # Pop from head
RPOP queue:tasks            # Pop from tail

LRANGE queue:tasks 0 -1     # Get all items
LTRIM queue:tasks 0 99      # Keep only first 100
```

**Use Cases**:
- Message queues (FIFO)
- Activity feeds (recent items)
- Task processing queues
- Capped collections (LTRIM)

**Performance**: O(1) for push/pop, O(N) for access by index

---

### 4. Sets (Unordered Unique Collections)
```redis
SADD tags:post:1 "redis" "caching" "database"
SADD tags:post:2 "redis" "performance"

SMEMBERS tags:post:1
> 1) "redis"
> 2) "caching"
> 3) "database"

SISMEMBER tags:post:1 "redis"
> 1  # True

# Set operations
SINTER tags:post:1 tags:post:2   # Intersection
> 1) "redis"

SUNION tags:post:1 tags:post:2   # Union
SDIFF tags:post:1 tags:post:2    # Difference
```

**Use Cases**:
- Tags/labels
- Unique visitors tracking
- Friend relationships
- Permissions/roles
- Deduplication

---

### 5. Sorted Sets (Scored Ordered Collections)
Members with scores, automatically sorted.

```redis
ZADD leaderboard 100 "player1" 200 "player2" 150 "player3"

# Get top 3
ZREVRANGE leaderboard 0 2 WITHSCORES
> 1) "player2"
> 2) "200"
> 3) "player3"
> 4) "150"
> 5) "player1"
> 6) "100"

# Get rank
ZRANK leaderboard "player1"
> 0  # Position in sorted order

# Range by score
ZRANGEBYSCORE leaderboard 100 200

# Increment score atomically
ZINCRBY leaderboard 50 "player1"
```

**Use Cases**:
- Leaderboards/rankings
- Priority queues
- Time-series data (score = timestamp)
- Rate limiting (sliding window)
- Auto-complete suggestions
- Trending items

**Performance**: O(log N) for most operations

---

### 6. Bitmaps
Efficient bit-level operations.

```redis
# Track daily active users
SETBIT users:2026-01-17 1000 1  # User 1000 was active
SETBIT users:2026-01-17 1001 1

GETBIT users:2026-01-17 1000
> 1

# Count active users
BITCOUNT users:2026-01-17
> 2

# Users active on both days
BITOP AND result users:2026-01-17 users:2026-01-18
```

**Use Cases**:
- Daily/weekly active users
- Feature flags
- Permissions (bit per permission)
- Real-time analytics

**Memory Efficient**: 1 billion users = ~120 MB

---

### 7. HyperLogLog
Probabilistic cardinality estimation.

```redis
PFADD unique:visitors "user1" "user2" "user3"
PFCOUNT unique:visitors
> 3

# Merge multiple sets
PFMERGE unique:all unique:visitors:day1 unique:visitors:day2
```

**Use Cases**:
- Unique visitors counting
- Distinct elements in streams
- Cardinality estimation

**Advantages**:
- Fixed memory (12 KB per key)
- 0.81% error rate
- Handles billions of elements

---

### 8. Streams (Redis 5.0+)
Append-only log for event streaming.

```redis
XADD events:user * action "login" user_id 1000 timestamp 1642444800
> "1642444800000-0"

XREAD COUNT 10 STREAMS events:user 0
XGROUP CREATE events:user mygroup 0
XREADGROUP GROUP mygroup consumer1 COUNT 1 STREAMS events:user >
```

**Use Cases**:
- Event sourcing
- Activity streams
- Message queuing with acknowledgments
- Time-series data

---

## Persistence Options

Redis offers optional persistence to survive restarts.

### 1. RDB (Redis Database Backup)
Point-in-time snapshots.

```conf
# redis.conf
save 900 1      # Save if 1 key changed in 900 seconds
save 300 10     # Save if 10 keys changed in 300 seconds
save 60 10000   # Save if 10,000 keys changed in 60 seconds
```

**Process**:
1. Fork child process
2. Child writes snapshot to disk
3. Atomic rename when complete

**Pros**:
- Compact single file
- Fast restarts
- Good for backups

**Cons**:
- Data loss between snapshots
- Forking can be expensive (large datasets)
- Not suitable for write-heavy workloads

---

### 2. AOF (Append-Only File)
Log of every write operation.

```conf
# redis.conf
appendonly yes
appendfsync everysec  # Options: always, everysec, no

# Rewrite when too large
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

**Fsync Options**:
- **always**: Every write (slow, most durable)
- **everysec**: Every second (balanced) ✅ Recommended
- **no**: OS decides (fast, least durable)

**Pros**:
- Minimal data loss (1 second max with everysec)
- More durable
- Automatic compaction

**Cons**:
- Larger files
- Slower restarts
- Slightly slower writes

---

### 3. Hybrid (RDB + AOF)
Redis 4.0+ supports hybrid persistence:
- RDB snapshot as base
- AOF for recent changes

**Recommended for Production**: Hybrid approach provides fast restarts with durability.

---

### 4. No Persistence (Pure Cache)
```conf
save ""
appendonly no
```

**Use When**:
- Using Redis purely as cache
- Data can be regenerated
- Maximum performance needed

---

## Redis Architecture

### Single Instance
```
┌─────────────────┐
│  Application    │
│  ┌───────────┐  │
│  │  Clients  │  │
└──┴─────┬─────┴──┘
         │
    ┌────▼────┐
    │  Redis  │
    │  Server │
    └─────────┘
```

**Pros**: Simple, low latency
**Cons**: Single point of failure, limited capacity

---

### Master-Slave Replication
```
         ┌─────────────┐
         │   Master    │
         │  (Writes)   │
         └──────┬──────┘
                │
        ┌───────┴───────┐
        │               │
   ┌────▼────┐    ┌────▼────┐
   │ Slave 1 │    │ Slave 2 │
   │ (Reads) │    │ (Reads) │
   └─────────┘    └─────────┘
```

**Configuration**:
```conf
# On slave
replicaof master-host master-port
```

**Benefits**:
- Read scaling (distribute reads to slaves)
- Data redundancy
- Failover capability

**Replication**:
- Asynchronous by default
- Eventually consistent
- Full sync + partial sync

---

### Redis Sentinel (High Availability)
```
┌──────────┐  ┌──────────┐  ┌──────────┐
│Sentinel 1│  │Sentinel 2│  │Sentinel 3│
└─────┬────┘  └─────┬────┘  └─────┬────┘
      │             │             │
      └─────────────┼─────────────┘
                    │
              ┌─────▼──────┐
              │   Master   │
              └────────────┘
```

**Features**:
- Automatic failover
- Monitoring
- Notification
- Configuration provider

**Quorum**: Majority of sentinels must agree (typically 3 sentinels)

---

### Redis Cluster (Sharding)
```
         ┌──────────────────────┐
         │   Hash Slot Router   │
         │   (16,384 slots)     │
         └──────────┬───────────┘
                    │
     ┌──────────────┼──────────────┐
     │              │              │
┌────▼────┐    ┌───▼─────┐   ┌───▼─────┐
│ Node 1  │    │ Node 2  │   │ Node 3  │
│0-5460   │    │5461-    │   │10923-   │
│         │    │10922    │   │16383    │
└────┬────┘    └────┬────┘   └────┬────┘
     │              │             │
┌────▼────┐    ┌───▼─────┐   ┌───▼─────┐
│Replica 1│    │Replica 2│   │Replica 3│
└─────────┘    └─────────┘   └─────────┘
```

**Features**:
- Horizontal scaling (data partitioning)
- Automatic sharding
- High availability (built-in replication)

**Limitations**:
- Multi-key operations limited
- No database selection (only DB 0)
- Client library must support cluster mode

---

## Replication and High Availability

### Replication Modes

**1. Asynchronous Replication (Default)**
```
Master: Write → ACK to client → Replicate in background
```
- Fast writes
- Risk of data loss on master failure

**2. Synchronous Replication**
```
Master: Write → Wait for N replicas → ACK to client
```
```redis
WAIT 2 1000  # Wait for 2 replicas with 1s timeout
```
- Stronger durability
- Higher latency

---

### Failover Strategies

**Manual Failover**
```bash
redis-cli> SLAVEOF NO ONE  # Promote slave to master
```

**Sentinel Automatic Failover**
1. Sentinel detects master down
2. Quorum reached
3. Elect new master from slaves
4. Reconfigure other slaves
5. Notify clients

**Cluster Automatic Failover**
- Built-in replica promotion
- Gossip protocol for failure detection
- Automatic reconfiguration

---

## Performance Characteristics

### Throughput
- **Single Instance**: 100K+ ops/sec
- **Cluster**: Linear scaling (N nodes ≈ N × throughput)

### Latency
- **Local Network**: <1ms
- **Same Data Center**: 1-2ms
- **Cross-Region**: 50-200ms (avoid if possible)

### Memory Efficiency

**String vs Hash**:
```
100,000 users stored as strings: ~60 MB
100,000 users stored as hash:    ~20 MB (3x more efficient)
```

**Key Size**:
- Shorter keys = less memory
- But keep readable: `user:1000:profile` not `u:1k:p`

**Value Size**:
- Avoid >1 MB values
- Split large objects
- Consider compression

---

## Common Use Cases

### 1. Caching
```java
// Cache-aside pattern
public User getUser(Long userId) {
    String key = "user:" + userId;
    String cached = redis.get(key);
    
    if (cached != null) {
        return deserialize(cached);
    }
    
    User user = database.findUser(userId);
    redis.setex(key, 3600, serialize(user));
    return user;
}
```

### 2. Session Store
```java
String sessionId = UUID.randomUUID().toString();
redis.setex("session:" + sessionId, 1800, sessionData);
```

### 3. Rate Limiting (Sliding Window)
```lua
local current = redis.call('INCR', KEYS[1])
if current == 1 then
    redis.call('EXPIRE', KEYS[1], ARGV[1])
end
if current > tonumber(ARGV[2]) then
    return 0
else
    return 1
end
```

### 4. Leaderboard
```java
redis.zadd("leaderboard:2026-01", score, playerId);
List<String> topPlayers = redis.zrevrange("leaderboard:2026-01", 0, 9);
```

### 5. Distributed Locks
```java
// Acquire lock
String lockValue = UUID.randomUUID().toString();
boolean acquired = redis.set("lock:resource", lockValue, "NX", "EX", 30);

// Release lock (with Lua for atomicity)
String script = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                "return redis.call('del', KEYS[1]) else return 0 end";
redis.eval(script, 1, "lock:resource", lockValue);
```

### 6. Pub/Sub for Real-Time Updates
```java
// Publisher
redis.publish("notifications:user:1000", "New message received");

// Subscriber
redis.subscribe("notifications:user:1000", (channel, message) -> {
    // Handle notification
});
```

### 7. Time-Series Data (Sorted Sets)
```java
long timestamp = System.currentTimeMillis();
redis.zadd("metrics:cpu", timestamp, value);
// Get last hour
redis.zrangebyscore("metrics:cpu", timestamp - 3600000, timestamp);
```

---

## Best Practices

### 1. Key Naming Conventions
```
Pattern: {namespace}:{entity}:{id}:{field}

Examples:
user:profile:1000
product:inventory:sku-123
session:auth:abc-def-ghi
cache:api:users:list:page1
```

### 2. Use Pipelining for Bulk Operations
```java
Pipeline pipeline = redis.pipelined();
for (int i = 0; i < 1000; i++) {
    pipeline.set("key" + i, "value" + i);
}
pipeline.sync();  // Execute all at once
```

**Performance**: 10x faster for bulk operations

### 3. Avoid Large Keys
- Max value size: 512 MB
- Practical limit: <10 MB
- Split large objects into smaller keys

### 4. Set Appropriate TTLs
```java
redis.setex("cache:product:1000", 3600, data);  // 1 hour
redis.expire("session:abc", 1800);              // 30 minutes
```

### 5. Monitor Memory Usage
```redis
INFO memory
CONFIG GET maxmemory
CONFIG GET maxmemory-policy
```

### 6. Choose Right Eviction Policy
```conf
maxmemory 2gb
maxmemory-policy allkeys-lru
```

**Options**:
- `noeviction`: Return errors when memory full
- `allkeys-lru`: Evict least recently used keys ✅ Recommended for cache
- `volatile-lru`: Evict LRU among keys with TTL
- `allkeys-random`: Evict random keys
- `volatile-ttl`: Evict keys with shortest TTL

### 7. Use Connection Pooling
```java
JedisPoolConfig config = new JedisPoolConfig();
config.setMaxTotal(128);
config.setMaxIdle(64);
config.setMinIdle(16);

JedisPool pool = new JedisPool(config, "localhost", 6379);
```

### 8. Handle Failures Gracefully
```java
try {
    return redis.get(key);
} catch (JedisException e) {
    logger.error("Redis failure, falling back to database", e);
    return database.get(key);
}
```

### 9. Monitor Key Metrics
- Hit/miss rate
- Memory usage
- Connected clients
- Evicted keys
- Commands per second

### 10. Secure Redis
```conf
# Require password
requirepass your-strong-password

# Bind to specific interface
bind 127.0.0.1

# Rename dangerous commands
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command CONFIG "CONFIG-abc123"
```

---

## Redis vs Memcached

| Feature | Redis | Memcached |
|---------|-------|-----------|
| **Data Structures** | Strings, Hashes, Lists, Sets, Sorted Sets, Bitmaps, HyperLogLog, Streams | Strings only |
| **Persistence** | RDB, AOF, Hybrid | No persistence |
| **Replication** | Master-slave, Cluster | No built-in replication |
| **Transactions** | MULTI/EXEC, Lua scripts | No transactions |
| **Pub/Sub** | Yes | No |
| **Memory Efficiency** | Better (data structures) | Good |
| **Multi-threading** | Single-threaded (I/O threads in 6.0+) | Multi-threaded |
| **Use Case** | Cache + Data structure store | Pure cache |

**When to Choose Redis**:
- Need data structures beyond key-value
- Require persistence
- Want pub/sub messaging
- Need atomic operations
- Data structure operations (sorted sets, etc.)

**When to Choose Memcached**:
- Pure caching with simple key-value
- Multi-threaded performance critical
- Simpler deployment

**Reality**: Redis has largely replaced Memcached in modern systems.

---

## Common Pitfalls

1. **Using Redis as Primary Database** (without understanding trade-offs)
   - Ensure proper persistence configuration
   - Understand memory limits
   - Plan for data eviction

2. **Not Setting TTLs on Cache Keys**
   - Keys never expire → memory leak
   - Always set expiration

3. **Large Keys/Values**
   - Blocks other operations (single-threaded)
   - Use pipelining, split large objects

4. **N+1 Queries**
   - Use MGET, pipelining, or hashes
   - Batch operations when possible

5. **Ignoring Memory Policies**
   - Set maxmemory and eviction policy
   - Monitor memory usage

6. **Not Using Connection Pooling**
   - Each connection has overhead
   - Pool connections for efficiency

7. **Synchronous Replication Everywhere**
   - Significantly impacts write latency
   - Use only when necessary

---

## Key Takeaways

1. **Redis is more than a cache** — it's a versatile data structure server
2. **Choose appropriate data structures** — can dramatically improve performance and memory usage
3. **Persistence is optional** — configure based on durability needs
4. **Plan for high availability** — use Sentinel or Cluster for production
5. **Monitor and tune** — Redis provides rich metrics for optimization
6. **Security matters** — don't expose Redis to public internet
7. **Use pipelining** — batch operations for better throughput
8. **Set TTLs** — prevent memory leaks and data staleness

---

## Next Steps

- [Local vs Distributed Cache](./local-vs-distributed-cache.md)
- [Cache-Aside Pattern](./cache-aside.md)
- [Cache Eviction Policies](./cache-eviction-policies.md)
- [Handling Cache Stampede](./cache-stampede.md)

---

**Last Updated**: 2026-01-17
**Target Audience**: Senior Software Engineers
**Difficulty Level**: Intermediate
