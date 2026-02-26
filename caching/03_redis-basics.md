# Redis Basics for System Design

---

## 1. What is Redis?

**Redis** stands for **Remote Dictionary Server**. It is an in-memory data store — meaning it keeps all data in RAM (computer memory) instead of on disk (hard drive).


Redis is not just a cache. It can work as:
- A **Cache** — fast temporary data storage
- A **Session Store** — storing user login sessions
- A **Message Broker** — sending messages between services (Pub/Sub)
- A **Primary Database** — for some use cases

### Key Features

| Feature           | What it means                                                  |
|-------------------|----------------------------------------------------------------|
| In-Memory         | All data lives in RAM — ultra-fast access                      |
| Persistent        | Can optionally save data to disk so it survives restarts       |
| Single-Threaded   | Processes one command at a time — no race conditions           |
| Atomic Operations | Every operation completes fully or not at all                  |
| Rich Data Types   | Not just key-value — supports lists, sets, sorted sets, hashes |


---

## 2. Why is In-Memory So Fast?

```
RAM access:   ~100 nanoseconds     ← Redis lives here
SSD access:   ~100 microseconds    (1,000x slower than RAM)
HDD access:   ~10 milliseconds     (100,000x slower than RAM)
```

This is why Redis is so popular — it serves data from RAM, which is orders of magnitude faster than reading from disk (which is what a traditional database does).

---

## 3. Redis Data Structures

Redis supports 7 different data structures. 

---

### Type 1: Strings — Simple Key-Value

The most basic type. One key maps to one value. Value can be text, numbers, or even binary data.

```
SET user:name "Ravi"        # Store
GET user:name               # Retrieve → "Ravi"

INCR page:views             # Add 1 (atomic)
INCRBY page:views 5         # Add 5
DECR page:views             # Subtract 1

SETEX session:abc 3600 data # Store with 1 hour expiry
```

**When to use Strings:**
- Simple caching (store any JSON data as a string)
- Counters (page views, login attempts)
- Session tokens
- Rate limiting
- Feature flags (on/off)

**Limit:** Maximum 512 MB per value.

---

### Type 2: Hashes — Objects / Maps

Store multiple fields under one key. Like a mini object or dictionary. Perfect for storing entity data like a user profile.

```
HSET user:1000 name "Priya" email "priya@gmail.com" age 28 city "Mumbai"

HGET user:1000 name           # → "Priya"
HGETALL user:1000             # → all fields and values
HSET user:1000 age 29         # Update only the age field
HINCRBY user:1000 age 1       # Atomic increment on one field
HEXISTS user:1000 phone       # → 0 (false, field does not exist)
HDEL user:1000 city           # Delete one field
```

**Why Hashes instead of storing each field as separate String keys?**

Storing as separate Strings: `user:1000:name`, `user:1000:email`, `user:1000:age` — three separate keys, more memory wasted.

Storing as Hash: `user:1000` — one key, all fields together. More memory efficient and semantically cleaner.

**When to use Hashes:**
- User profiles
- Product details (name, price, stock, category)
- Shopping cart items
- App configuration settings
- Real-time stats (views, clicks, conversions)

---

### Type 3: Lists — Ordered Sequences

A linked list that supports fast operations at both ends (head and tail). Think of it like a queue or a timeline.

```
LPUSH queue:tasks "task1"     # Add to front (left)
RPUSH queue:tasks "task3"     # Add to back (right)

LRANGE queue:tasks 0 -1       # View all items (without removing)
LPOP queue:tasks              # Remove from front
RPOP queue:tasks              # Remove from back

LTRIM queue:tasks 0 99        # Keep only first 100 items
BLPOP queue:tasks 5           # Wait up to 5 seconds for an item (blocking)
```

**Common Patterns:**
- **Queue (FIFO):** Add with RPUSH, remove with LPOP
- **Stack (LIFO):** Add with LPUSH, remove with LPOP
- **Capped Timeline:** Add with LPUSH + trim with LTRIM to keep only latest N items

**When to use Lists:**
- Task/job queues (background processing)
- Activity feeds (latest 100 actions)
- Notification history (recent 50 notifications)
- Log rotation (keep only last N logs)

**Performance:** O(1) for adding/removing at head or tail. O(N) for accessing by index.

---

### Type 4: Sets — Unique Collections

Unordered collection of unique strings. Duplicates are automatically ignored.

```
SADD tags:post:1 "redis" "caching" "database"
SADD tags:post:1 "redis"      # Ignored — already exists

SISMEMBER tags:post:1 "redis" # → 1 (true)
SMEMBERS tags:post:1          # → all members
SCARD tags:post:1             # → 3 (count)
SREM tags:post:1 "caching"    # Remove member
SRANDMEMBER tags:post:1       # Get random member
```

**Powerful Set Operations:**

```
SADD users:online "Amit" "Priya" "Rohan"
SADD users:premium "Priya" "Sunita"

SINTER users:online users:premium  # Common → "Priya"
SUNION users:online users:premium  # All → "Amit", "Priya", "Rohan", "Sunita"
SDIFF  users:online users:premium  # In online but NOT premium → "Amit", "Rohan"
```

**When to use Sets:**
- Tags or labels on articles/products
- Tracking unique visitors per day (deduplication)
- Friend/follower relationships
- IP blocklists (fast lookup)
- User permissions

---

### Type 5: Sorted Sets — Scored Rankings

Like Sets, but every member has a **score** (a number). Members are automatically sorted by score. This is Redis's most powerful data structure for ranking use cases.

```
ZADD leaderboard 100 "Amit" 200 "Priya" 150 "Rohan"

ZREVRANGE leaderboard 0 2 WITHSCORES   # Top 3 (highest score first)
# → Priya: 200, Rohan: 150, Amit: 100

ZRANK leaderboard "Amit"        # → 0 (position, 0-indexed from lowest)
ZSCORE leaderboard "Priya"      # → 200

ZINCRBY leaderboard 50 "Amit"   # Add 50 to Amit's score → 150
ZCOUNT leaderboard 100 200      # Count members with score between 100 and 200
ZRANGEBYSCORE leaderboard 100 200  # Members with score 100 to 200
```

**When to use Sorted Sets:**
- Leaderboards (score = points)
- Priority queues (score = priority, process highest first)
- Rate limiting with sliding window (score = timestamp)
- Trending topics (score = popularity)
- Autocomplete suggestions (score = frequency)
- Time-series data (score = timestamp)

**Performance:** O(log N) for most operations.

---

### Type 6: Bitmaps — Memory-Efficient Binary Tracking

A bitmap is like an array of on/off switches. Each bit position represents one user or event. Extremely memory efficient for tracking binary states at scale.

```
SETBIT users:2024-01-17 1000 1    # User 1000 was active today
SETBIT users:2024-01-17 5000 1    # User 5000 was active today

GETBIT users:2024-01-17 1000      # → 1 (true, was active)
GETBIT users:2024-01-17 9999      # → 0 (false, was not active)

BITCOUNT users:2024-01-17         # → Total active users today

# Users active on BOTH days
BITOP AND result users:2024-01-17 users:2024-01-18
BITCOUNT result                   # Count of users active both days
```

**Memory comparison — this is amazing:**

```
Using Set (SADD) for 1 million users:   ~50 MB
Using Bitmap for 1 million users:       ~125 KB   (400x smaller!)
```

**When to use Bitmaps:**
- Daily active user tracking
- Feature flags per user
- A/B testing (is this user in test group?)
- Real-time analytics (click tracking for billions of events)

---

### Quick Data Structure Cheat Sheet

| Situation                        | Use This   | Why                               |
|----------------------------------|------------|-----------------------------------|
| Store a single value or counter  | String     | Simple and fast                   |
| Store an object with many fields | Hash       | Memory efficient, partial updates |
| Ordered list, queue, or feed     | List       | Fast head/tail operations         |
| Unique items, deduplication      | Set        | No duplicates, set operations     |
| Rankings or scoring              | Sorted Set | Auto-sorted by score              |
| Track billions of on/off states  | Bitmap     | Extremely memory efficient        |

---

## 4. Persistence — How Redis Survives Restarts

Redis is in-memory, so by default all data is lost when it restarts. But Redis gives two options to save data to disk.

### Option 1: RDB — Periodic Snapshots

Redis takes a full snapshot of all data and saves it to a file (`dump.rdb`) at regular intervals.

```
Configuration example:
save 900 1       # Save if at least 1 key changed in 15 minutes
save 300 10      # Save if at least 10 keys changed in 5 minutes
save 60 10000    # Save if at least 10,000 keys changed in 1 minute
```

**Pros:** Fast restart (single compact file), good for backups, minimal performance impact.

**Cons:** Data between last snapshot and crash is lost. If Redis crashes at 2:25 PM and last snapshot was at 2:20 PM, you lose 5 minutes of data.

**Use when:** Caching only (data loss is acceptable), read-heavy workloads.

---

### Option 2: AOF — Append-Only File

Every write command is logged to a file. On restart, Redis replays all commands to restore data.

```
Configuration:
appendonly yes
appendfsync everysec    # Recommended: sync to disk every 1 second
```

**Fsync Policy Options:**

| Policy   | Safety  | Speed   | Max Data Loss     |
|----------|---------|---------|-------------------|
| always   | Highest | Slowest | 0 commands        |
| everysec | Good    | Good    | ~1 second of data |
| no       | Lowest  | Fastest | Unpredictable     |

**Pros:** Much less data loss (at most 1 second with everysec policy). More reliable for critical data.

**Cons:** Larger file size. Slower restart (must replay all commands).

**Use when:** You cannot afford to lose data — sessions, user state, important counters.

---

### RDB vs AOF — Which to Use?

| Aspect            | RDB     | AOF             | Both Together      |
|-------------------|---------|-----------------|--------------------|
| Durability        | Low     | High            | Highest            |
| Restart Speed     | Fast    | Slow            | Fast (uses RDB)    |
| File Size         | Small   | Large           | Both files         |
| Write Performance | Fast    | Slightly slower | Slowest            |
| Max Data Loss     | Minutes | ~1 second       | ~1 second          |
| Best For          | Caching | Critical data   | Production systems |

**Recommendation:** For most production systems, use both RDB + AOF together. On restart, Redis uses the RDB file for fast loading, and AOF for recent changes.

---

## 5. Redis Deployment Options

### Option 1: Single Instance

One Redis server. Simple to set up and manage.

**Good for:** Development, testing, small applications, caching only.

**Problem:** If that one server goes down, everything goes down. Single point of failure.

---

### Option 2: Replication (Master + Replicas)

One Master handles all writes. One or more Replicas copy data from master and handle reads.

```
All WRITES  → Master
All READS   → Replicas (or master)
```

**Pros:** Read scalability (add more replicas for more read capacity). If master fails, a replica can be promoted.

**Cons:** Writes still bottlenecked at one master. Replication is asynchronous — replicas might be slightly behind (eventual consistency). Manual failover if master crashes.

**Good for:** Read-heavy applications needing more read throughput.

---

### Option 3: Sentinel (Automatic Failover)

Adds 3 or more Sentinel processes that **monitor** your Master and Replicas. If Master goes down, Sentinels automatically elect a new Master from the Replicas.

How it works:
1. Sentinels keep checking if Master is alive.
2. If Master is down and majority of Sentinels agree (quorum), failover begins.
3. One Replica is promoted to new Master automatically.
4. Other Replicas start following the new Master.
5. Application clients are notified of the new Master address.

**Pros:** Automatic failover, no manual intervention needed. High availability.

**Cons:** Writes still limited to one Master. Slightly more complex to set up.

**Good for:** Applications that need high availability and cannot tolerate downtime.

---

### Option 4: Redis Cluster (Horizontal Scaling)

Data is split (sharded) across multiple Master nodes. Each Master owns a range of **hash slots** (Redis uses 16,384 total slots). Each Master also has Replicas for high availability.

```
Node 1 (Master): Slots 0 to 5,460
Node 2 (Master): Slots 5,461 to 10,922
Node 3 (Master): Slots 10,923 to 16,383

When you write key "user:1000":
  Slot = CRC16("user:1000") % 16384
  → Goes to whichever Node owns that slot
```

**Pros:** Both reads AND writes scale horizontally. No single point of failure. Automatic failover built in.

**Cons:** More complex to set up and manage. Multi-key operations across different nodes are not supported. Only one database (DB 0).

**Good for:** Very large scale applications where a single Redis server is not enough.

---

### Deployment Comparison

| Aspect            | Single   | Replication | Sentinel  | Cluster     |
|-------------------|----------|-------------|-----------|-------------|
| Setup Difficulty  | Easy     | Medium      | Medium    | Complex     |
| High Availability | No       | Manual      | Automatic | Automatic   |
| Read Scaling      | No       | Yes         | Yes       | Yes         |
| Write Scaling     | No       | No          | No        | Yes         |
| Best For          | Dev/Test | Read-heavy  | HA needed | Large scale |

---

## 6. Common Redis Use Cases

### Use Case 1: Caching (Cache-Aside Pattern)

```
function getUser(id):
    data = redis.GET("user:" + id)
    if data found:
        return data                    # Cache hit — fast!

    user = database.findById(id)       # Cache miss — go to DB
    redis.SETEX("user:" + id, 3600, user)  # Save in cache for 1 hour
    return user
```

---

### Use Case 2: Session Store

```
SETEX session:abc123 1800 "{user_id: 101, name: 'Ravi'}"
# Session expires automatically after 30 minutes
```

---

### Use Case 3: Rate Limiting

```
# Every time user makes a request:
count = INCR rate:user:101:minute
if count == 1:
    EXPIRE rate:user:101:minute 60    # Reset after 60 seconds

if count > 100:
    return "Too many requests! Please wait."
```

---

### Use Case 4: Leaderboard

```
ZADD leaderboard:january 5000 "Priya"
ZADD leaderboard:january 7200 "Amit"
ZADD leaderboard:january 3100 "Rohan"

ZREVRANGE leaderboard:january 0 9 WITHSCORES   # Top 10 players
```

---

### Use Case 5: Distributed Lock

When multiple servers need to ensure only ONE of them performs an action at a time (like sending a payment or running a scheduled job):

```
# Acquire lock — NX means "only set if key does NOT exist"
SET lock:payment:order123  my-unique-id  NX  EX 30

# If SET succeeded → you have the lock, do your work
# If SET failed → someone else has the lock, skip or retry

# Release lock (use Lua script to make it atomic)
if redis.GET(lock:payment:order123) == my-unique-id:
    redis.DEL(lock:payment:order123)
```

---

### Use Case 6: Real-Time Notifications with Pub/Sub

```
# Service A publishes a notification
PUBLISH notifications:user:1000 "Your order has been shipped!"

# Service B (notification server) is subscribed and receives it instantly
SUBSCRIBE notifications:user:1000
→ Receives: "Your order has been shipped!"
```

---

## 7. Best Practices

### 1. Use a Consistent Key Naming Pattern

```
Good pattern:  {namespace}:{entity}:{id}
Examples:
  user:profile:1000
  product:details:SKU-456
  session:auth:abc-def-ghi
  cache:search:results:page1

Bad examples:
  123                  ← What is this?
  userprofile1000      ← No separation
  u:1k:p               ← Too cryptic
```

---

### 2. Always Set TTL (Expiry Time)

```
BAD  — No expiry, key lives forever:
SET user:1000 data

GOOD — Set expiry:
SETEX user:1000 3600 data       # Expires in 1 hour

Set TTL based on how fresh the data needs to be:
  Sessions:             30 minutes
  Product details:      24 hours
  Config/reference:     1 hour to 12 hours
  Rate limit counters:  1 minute
```

Without TTL, Redis memory fills up over time and starts evicting important keys.

---

### 3. Use Pipelining for Bulk Operations

```
BAD — 1000 separate network round trips:
for each item:
    redis.set(key, value)

GOOD — 1 network round trip:
pipe = redis.pipeline()
for each item:
    pipe.set(key, value)
pipe.execute()    # Sends all at once
```

This can give 10x to 100x improvement for bulk operations.

---

### 4. Set a Memory Limit and Eviction Policy

```
In redis.conf:
maxmemory 2gb
maxmemory-policy allkeys-lru     # When full, remove least recently used keys
```

Without this, Redis will keep consuming memory until the server runs out of RAM and crashes!

**Eviction policy options:**
- `allkeys-lru` — Remove least recently used keys (good for general caching)
- `volatile-lru` — Remove least recently used keys that have TTL set
- `allkeys-lfu` — Remove least frequently used keys
- `noeviction` — Return error when memory is full (do not evict anything)

---

### 5. Monitor Regularly

```
INFO memory        # Check memory usage
INFO stats         # See hit rate, total commands, etc.
SLOWLOG GET 10     # Find slow commands
```

Target a **hit rate of 80% or above**. If hit rate is low, your TTL might be too short or you are caching the wrong data.

---

### 6. Do Not Store Very Large Values

Avoid storing values larger than 1 MB in Redis. Large values cause:
- Network slowdown when transferring
- High memory consumption
- Slow serialization/deserialization

If you must store large data, compress it first or split it into smaller chunks.

---

### 7. Secure Your Redis

```
In redis.conf:
requirepass your-strong-password      # Require password
bind 127.0.0.1                        # Only allow local connections
rename-command FLUSHDB ""             # Disable dangerous commands
rename-command FLUSHALL ""            # Disable dangerous commands
```

By default, Redis has no password and accepts connections from anywhere! Always configure security before going to production.

---

## 8. When to Use Redis vs When Not To

### Use Redis When:
- You need sub-millisecond response time
- Data is read much more often than written
- Data is temporary (sessions, cache, rate limit counters)
- You need real-time features (leaderboards, live counters, notifications)
- You need distributed locking across multiple servers

### Do NOT Use Redis When:
- You need complex SQL queries, joins, or aggregations (use a proper database)
- Data must be 100% durable with ACID guarantees (use PostgreSQL or MySQL)
- You are storing very large files or blobs (use object storage like S3)
- Data size is much larger than available RAM (Redis is memory-limited)

---