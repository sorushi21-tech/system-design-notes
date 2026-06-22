# System Design Mastery Roadmap

---

## Progress Checklist
- [ ] Phase 1: Foundations
- [ ] Phase 2: Storage & Data Management
- [ ] Phase 3: Caching
- [ ] Phase 4: Communication & Messaging
- [ ] Phase 5: Design Patterns & Resilience
- [ ] Phase 6: Architecture Patterns
- [ ] Phase 7: Security & Observability
- [ ] Phase 8: Performance & Optimization
- [ ] Phase 9: Java-Specific Deep Dive
- [ ] Phase 10: Distributed Systems
- [ ] Phase 11: Real-World System Design Cases
- [ ] Phase 12: Cloud & Infrastructure

---

## Phase 1: Foundations

📁 `fundamentals/`

### Core Fundamentals

- [ ] **scalability**
    - Vertical vs horizontal scaling trade-offs
    - Stateless design as a prerequisite for scaling
    - Bottlenecks: CPU, memory, I/O, network
    - Scalability vs performance distinction

- [ ] **latency-vs-throughput**
    - Definitions: latency (time per request), throughput (requests per second)
    - Little's Law: L = λW
    - Why optimizing one can hurt the other
    - P50, P95, P99 percentiles — why averages lie

- [ ] **horizontal-vs-vertical-scaling**
    - When to scale up vs scale out
    - Limits of vertical scaling (single machine ceiling)
    - Stateful services and the challenge of horizontal scaling
    - Cost implications of each approach

- [ ] **availability-reliability**
    - Availability = uptime percentage (99.9% = 8.7 hrs/year downtime)
    - SLA, SLO, SLI definitions
    - MTTR (Mean Time To Recover) vs MTBF (Mean Time Between Failures)
    - Designing for failure, not against it

- [ ] **fault-tolerance**
    - Fail-fast vs fail-safe strategies
    - Redundancy: active-active vs active-passive
    - Cascading failure prevention
    - Timeouts, retries, and fallback patterns

- [ ] **backpressure-basics**
    - What backpressure is and why it matters
    - Producer faster than consumer — what happens
    - Buffer, drop, or block strategies
    - Reactive Streams and backpressure propagation

- [ ] **graceful-degradation**
    - Serving partial results instead of full failure
    - Feature flags for emergency kill switches
    - Fallback responses (stale cache, defaults)
    - Circuit breaker as graceful degradation enabler

📌 **Goal**: Understand *why* systems fail and *how* they scale.

---

### Capacity Estimation & System Math

- [ ] **capacity-estimation**
    - Start with: DAU (Daily Active Users) → requests per second
    - Storage growth projections over 1, 3, 5 years
    - Read/write ratio analysis
    - Network bandwidth requirements

- [ ] **traffic-estimation**
    - Peak vs average traffic (10x rule of thumb)
    - Bursty traffic patterns and their impact
    - Traffic shaping vs traffic shedding
    - Estimating concurrent connections

- [ ] **storage-estimation**
    - Object size × volume = raw storage
    - Replication factor (typically 3x)
    - Compression ratios for different data types
    - Hot vs cold storage tiering costs

- [ ] **qps-calculation**
    - QPS = total requests / seconds in period
    - Translating DAU to QPS (DAU × avg_requests / 86400)
    - Read QPS vs Write QPS separate calculations
    - Caching impact on backend QPS

- [ ] **latency-budgeting**
    - End-to-end latency = sum of all hops
    - Allocating budget: network + DB + compute + serialization
    - SLA-driven budget: if SLA = 100ms, how much per layer?
    - Tail latency (P99) vs median latency

- [ ] **back-of-the-envelope-calculations**
    - Key numbers: RAM ~100ns, SSD ~100µs, HDD ~10ms, network ~1ms
    - Powers of 2 for storage: KB, MB, GB, TB, PB
    - Common shortcuts: 1M requests/day ≈ 12 QPS
    - Practice: Twitter, YouTube, WhatsApp scale estimates

📌 **Goal**: Size systems realistically and justify design decisions.

---

### Data & Consistency

- [ ] **cap-theorem**
    - Consistency, Availability, Partition Tolerance — pick 2
    - CP systems (HBase, Zookeeper) vs AP systems (Cassandra, DynamoDB)
    - Partition tolerance is non-negotiable in distributed systems
    - PACELC extension: latency trade-off even without partition

- [ ] **consistency-models**
    - Strong consistency: reads always see latest write
    - Eventual consistency: replicas converge over time
    - Read-your-writes, Monotonic reads, Causal consistency
    - When to choose which model

- [ ] **eventual-consistency-patterns**
    - How replicas sync (gossip, log shipping, etc.)
    - Conflict resolution: last-write-wins, vector clocks, CRDTs
    - Read repair and anti-entropy
    - Handling "stale read" in application code

- [ ] **data-partitioning**
    - Range partitioning vs hash partitioning
    - Consistent hashing — virtual nodes, rebalancing
    - Directory-based partitioning
    - Cross-partition queries and their cost

- [ ] **hot-keys-problem**
    - What causes hot keys (celebrity posts, viral content)
    - Detection: monitoring request distribution
    - Solutions: key splitting, local caching, random suffixes
    - Shard re-balancing strategies

- [ ] **denormalization-at-scale**
    - Why normalization breaks at high read throughput
    - Pre-computing aggregates and materialized views
    - Write amplification as the trade-off
    - Event sourcing as an alternative

- [ ] **load-balancing-algorithms**
    - Round Robin, Weighted Round Robin
    - Least Connections, Least Response Time
    - IP Hash (sticky sessions)
    - Consistent Hashing for cache/shard affinity

📌 **Goal**: Balance correctness, availability, and performance.

---

### Time, Ordering & Clocks

- [ ] **clock-skew**
    - Why distributed clocks drift apart
    - Impact on ordering events across nodes
    - TrueTime (Google Spanner) approach
    - Practical bounds: NTP skew ~100ms

- [ ] **ntp-time-sync**
    - How NTP works (Stratum levels)
    - Limitations in distributed systems
    - PTP (Precision Time Protocol) for tighter sync
    - Never rely on wall clock for ordering

- [ ] **lamport-clocks**
    - Logical clock: increment on send, max+1 on receive
    - Captures "happens-before" relationship
    - Limitation: can't detect concurrent events
    - Use case: ordering events in a single service

- [ ] **vector-clocks**
    - One counter per node in the system
    - Detects causality AND concurrency
    - Used in Dynamo-style systems for conflict detection
    - Trade-off: size grows with node count

- [ ] **event-time-vs-processing-time**
    - Event time: when event occurred
    - Processing time: when system processed it
    - Why they differ (network delay, retries, batching)
    - Watermarks in stream processing (Flink, Kafka Streams)

- [ ] **out-of-order-events**
    - Late-arriving data in streaming systems
    - Windowing strategies: tumbling, sliding, session windows
    - Grace periods and when to close a window
    - Reprocessing vs accepting inaccuracy

📌 **Goal**: Handle time correctly in distributed systems.

---

### Networking Basics

- [ ] **dns**
    - Resolution flow: browser → resolver → root → TLD → authoritative
    - A, CNAME, MX, TXT record types
    - TTL and its impact on propagation and failover speed
    - DNS-based load balancing and GeoDNS

- [ ] **http-https**
    - HTTP/1.1 vs HTTP/2 vs HTTP/3 key differences
    - TLS handshake steps and certificate chain
    - Common headers: Cache-Control, ETag, Content-Encoding
    - Status codes that matter in distributed systems (502, 503, 504)

- [ ] **tcp-vs-udp**
    - TCP: reliable, ordered, connection-oriented (3-way handshake)
    - UDP: unreliable, unordered, low overhead
    - Use UDP for: DNS, streaming, gaming, WebRTC
    - QUIC (HTTP/3): UDP-based with reliability built in

- [ ] **connection-lifecycle**
    - TCP connection: SYN → SYN-ACK → ACK → data → FIN
    - Connection overhead and why pooling matters
    - TIME_WAIT state and port exhaustion
    - Half-open connections and keepalive probes

- [ ] **keep-alive**
    - Reusing TCP connections across multiple requests
    - HTTP/1.1 keep-alive vs HTTP/2 multiplexing
    - Connection pool sizing: too small = latency, too large = resource waste
    - Idle connection timeouts and their risks

- [ ] **proxies-reverse-proxies**
    - Forward proxy: client-side (hides client identity)
    - Reverse proxy: server-side (hides server identity, load balances)
    - Nginx/HAProxy as reverse proxies
    - SSL termination at proxy layer

- [ ] **cdn**
    - Edge nodes cache static and dynamic content
    - Cache-Control headers drive CDN behavior
    - Cache invalidation at CDN (purge APIs)
    - CDN for latency reduction by geo-proximity

📌 **Goal**: Understand how requests flow across networks.

---

## Phase 2: Storage & Data Management

📁 `databases/`

### Database Fundamentals

- [ ] **sql-vs-nosql**
    - SQL: ACID, schema-on-write, joins, strong consistency
    - NoSQL: schema-on-read, horizontal scale, various consistency levels
    - Don't default to NoSQL — SQL scales further than people think
    - Polyglot persistence: right DB for right job

- [ ] **acid-vs-base**
    - ACID: Atomicity, Consistency, Isolation, Durability
    - BASE: Basically Available, Soft-state, Eventually consistent
    - Isolation levels: Read Uncommitted → Serializable
    - Phantom reads, dirty reads, non-repeatable reads

- [ ] **transactions**
    - Local transactions vs distributed transactions
    - Two-Phase Commit (2PC): prepare + commit phases, coordinator failure
    - Saga pattern as an alternative to 2PC
    - Optimistic vs pessimistic locking

- [ ] **indexing**
    - B-Tree index: balanced, good for range queries
    - Hash index: O(1) point lookups, no range support
    - Composite indexes and column order matters
    - Index selectivity, covering indexes, index bloat

- [ ] **normalization**
    - 1NF, 2NF, 3NF — eliminate redundancy and anomalies
    - When to denormalize: read-heavy, reporting, analytics
    - Join cost at scale vs update anomalies
    - OLTP favors normalization; OLAP favors denormalization

- [ ] **read-vs-write-optimized-models**
    - LSM Tree (write-optimized): RocksDB, Cassandra
    - B-Tree (read-optimized): PostgreSQL, MySQL
    - Write amplification vs read amplification
    - Choose model based on read/write ratio

---

### Database Scalability

- [ ] **replication**
    - Leader-follower (master-slave): sync vs async replication
    - Multi-leader: conflict resolution required
    - Leaderless (Quorum reads/writes): Dynamo model
    - Replication lag and its application impact

- [ ] **sharding**
    - Horizontal partitioning: each shard holds subset of data
    - Shard key selection is critical — avoid hot shards
    - Cross-shard queries: scatter-gather pattern
    - Resharding pain and consistent hashing solution

- [ ] **leader-election**
    - Why you need one: prevent split-brain
    - Algorithms: Bully, Raft, Paxos
    - Zookeeper/etcd for distributed coordination
    - Lease-based leader with TTL expiry

- [ ] **failover-strategies**
    - Automatic failover: promote replica on leader failure
    - Failover detection: heartbeat + timeout
    - Split-brain risk during network partition
    - Manual vs automatic failover trade-offs

- [ ] **connection-pooling**
    - Why: TCP connection setup is expensive
    - Pool size formula: (core_count × 2) + effective_spindle_count
    - HikariCP internals and configuration
    - Connection leak detection and max lifetime settings

- [ ] **nosql-types**
    - Key-Value: Redis, DynamoDB — simple, fast, no joins
    - Document: MongoDB, Firestore — flexible schema, nested data
    - Wide-Column: Cassandra, HBase — time-series, high write throughput
    - Graph: Neo4j — relationships are first-class citizens

📌 **Goal**: Scale data without breaking consistency.

---

###  Data Engineering & Analytics

- [ ] **olap-vs-oltp**
    - OLTP: many small transactions, normalized, row-oriented
    - OLAP: few large analytical queries, denormalized, columnar
    - Separate them: write to OLTP, ETL/CDC to OLAP
    - Hybrid HTAP databases (TiDB, SingleStore)

- [ ] **data-warehouse-basics**
    - Star schema vs snowflake schema
    - Fact tables vs dimension tables
    - ETL vs ELT pipeline patterns
    - Tools: Redshift, BigQuery, Snowflake

- [ ] **lambda-architecture**
    - Batch layer: recompute from raw data (correctness)
    - Speed layer: real-time processing (low latency)
    - Serving layer: merge batch + speed views
    - Complexity problem: maintaining two code paths

- [ ] **kappa-architecture**
    - Single stream processing layer only
    - Reprocess by replaying from Kafka topic
    - Simpler than Lambda but requires replayable log
    - When to choose: unified logic, Kafka-native

- [ ] **streaming-analytics-design**
    - Stateful stream processing: windowing, joins
    - Exactly-once semantics in stream processing
    - Flink vs Kafka Streams vs Spark Structured Streaming
    - Backpressure in streaming pipelines

---

## Phase 3: Caching

📁 `caching/`

- [ ] **redis-basics**
    - Data structures: String, Hash, List, Set, Sorted Set, Stream
    - Persistence: RDB snapshots vs AOF (Append-Only File)
    - Redis Cluster: hash slots, consistent hashing
    - Pub/Sub, Lua scripting, transactions with MULTI/EXEC

- [ ] **local-vs-distributed-cache**
    - Local (in-process): zero network latency, not shared, Caffeine/Guava
    - Distributed: shared across instances, network hop, Redis/Memcached
    - Hybrid L1/L2: local for hot data, distributed for cold
    - Consistency challenges with local cache in clustered deployments

- [ ] **two-level-cache**
    - L1 = local JVM cache, L2 = Redis
    - L1 hit: fastest, L2 hit: fast, DB hit: slow
    - Cache invalidation across L1 instances via pub/sub
    - Spring Cache with CaffeineCacheManager + RedisCacheManager

- [ ] **cache-aside**
    - App checks cache first, on miss loads from DB and populates cache
    - Lazy population — only cache what's needed
    - Risk: cache stampede on cold start
    - Most common pattern; gives app full control

- [ ] **read-through**
    - Cache sits in front of DB; cache handles DB fetch on miss
    - App only talks to cache, not DB directly
    - Consistent population logic in one place
    - Trade-off: first request always slow

- [ ] **write-through**
    - Write to cache and DB synchronously on every update
    - Cache always in sync with DB
    - Write latency doubles (cache + DB write)
    - Good for read-heavy data that must be fresh

- [ ] **write-behind**
    - Write to cache immediately, async flush to DB later
    - Lower write latency; higher risk of data loss
    - Requires durable queue between cache and DB
    - Good for high-write, loss-tolerant scenarios (analytics, counters)

- [ ] **cache-invalidation**
    - TTL-based: simple, but may serve stale data until expiry
    - Event-driven: invalidate on data change via messaging
    - Write-through: always consistent but adds write latency
    - Hardest problem in distributed caching — Phil Karlton's quote

- [ ] **cache-eviction-policies**
    - LRU (Least Recently Used): evict oldest accessed
    - LFU (Least Frequently Used): evict least accessed overall
    - FIFO, Random eviction
    - Redis `allkeys-lru` vs `volatile-lru` policy settings

- [ ] **cache-stampede**
    - Many requests hit DB simultaneously when cache expires
    - Solutions: mutex lock, probabilistic early expiration, request coalescing
    - Redis lock with SETNX + TTL pattern
    - Background refresh before TTL expiry

- [ ] **cache-penetration**
    - Requests for non-existent keys bypass cache, hit DB every time
    - Cache null/empty result with short TTL
    - Bloom filter to check existence before cache lookup
    - Attack vector: malicious requests with invalid keys

- [ ] **multi-tenant-caching**
    - Namespace keys by tenant: `tenant:{id}:resource:{key}`
    - Prevent cross-tenant data leaks
    - Per-tenant TTL and eviction policies
    - Redis keyspace isolation with logical DBs or separate clusters

📌 **Goal**: Reduce latency and DB load safely.

---

## Phase 4: Communication & Messaging

📁 `api-design/`

### API Design

- [ ] **rest-api-design**
    - Resource-based URLs: nouns not verbs (`/users` not `/getUsers`)
    - HTTP methods: GET (safe), POST (create), PUT (replace), PATCH (partial), DELETE
    - Status codes: 200, 201, 204, 400, 401, 403, 404, 409, 422, 500, 503
    - HATEOAS, Richardson Maturity Model

- [ ] **api-versioning**
    - URI versioning: `/v1/users` — most common, visible
    - Header versioning: `Accept: application/vnd.api.v1+json`
    - Query param: `/users?version=1`
    - Deprecation strategy and sunset headers

- [ ] **pagination**
    - Offset pagination: `?page=2&size=20` — simple but slow at depth
    - Cursor pagination: `?after=cursor_id` — stable, efficient
    - Keyset pagination: `WHERE id > last_id LIMIT 20` — index-friendly
    - Total count: expensive, avoid for large datasets

- [ ] **idempotency**
    - GET, PUT, DELETE are naturally idempotent; POST is not
    - Idempotency keys: client sends unique key, server deduplicates
    - Store idempotency keys in DB with result
    - Critical for payments and retry-safe operations

- [ ] **graphql**
    - Single endpoint, client specifies exact data shape
    - Solves over-fetching and under-fetching
    - N+1 problem and DataLoader batching solution
    - When NOT to use: public APIs, simple CRUD, caching complexity

- [ ] **grpc**
    - Binary Protocol Buffers over HTTP/2
    - Strongly typed contracts via `.proto` files
    - Streaming: unary, server-side, client-side, bidirectional
    - Use for: internal microservices, high-throughput, low-latency

---

### API Economy & Integration Patterns

- [ ] **event-driven-apis**
    - Push vs pull: webhooks vs polling
    - AsyncAPI specification for event-driven contracts
    - CloudEvents standard for event envelope
    - Streaming APIs: SSE, WebSockets, gRPC streaming

- [ ] **webhook-design**
    - Webhook delivery: at-least-once with retry
    - Signature verification (HMAC) to validate sender
    - Idempotency handling on consumer side
    - Webhook fan-out and delivery order guarantees

- [ ] **enterprise-integration-patterns-eip**
    - Message Router, Splitter, Aggregator, Filter
    - Content-Based Routing pattern
    - Correlation ID for request tracking across systems
    - Apache Camel as EIP implementation in Java

- [ ] **esb-vs-modern-event-broker**
    - ESB: centralized, heavyweight, orchestration-heavy
    - Modern: Kafka/Pulsar, decentralized, choreography-based
    - Smart endpoints, dumb pipes principle
    - Migration path: strangler fig on legacy ESB

- [ ] **api-gateway-as-contract**
    - Gateway as the contract boundary between client and services
    - Cross-cutting concerns: auth, rate-limiting, logging at gateway
    - API composition / BFF pattern at gateway layer
    - Kong, AWS API Gateway, Spring Cloud Gateway

---

### Asynchronous Communication

- [ ] **kafka**
    - Topics, Partitions, Offsets — core model
    - Producer: acks=0/1/all durability settings
    - Consumer groups: each partition consumed by one member
    - Log compaction, retention policy, segment files

- [ ] **rabbitmq**
    - Exchange types: Direct, Fanout, Topic, Headers
    - Queues, bindings, routing keys
    - Acknowledgment modes: auto-ack vs manual-ack
    - Dead letter exchange (DLX) setup

- [ ] **pub-sub-pattern**
    - Publisher doesn't know subscribers
    - Topic-based vs content-based filtering
    - Fan-out: one message → many consumers
    - vs point-to-point queue

- [ ] **message-queue-vs-event-streaming**
    - Queue: message consumed and deleted (RabbitMQ, SQS)
    - Stream: message retained, multiple consumers replay (Kafka)
    - Use queue for: tasks, jobs, commands
    - Use stream for: events, audit log, event sourcing

- [ ] **at-least-once-vs-at-most-once**
    - At-most-once: fire and forget, possible message loss
    - At-least-once: retry on failure, possible duplicates
    - Idempotent consumers handle duplicate messages safely
    - Default choice for critical data: at-least-once

- [ ] **exactly-once**
    - Kafka transactions: `transactional.id` + `isolation.level=read_committed`
    - Idempotent producer: `enable.idempotence=true`
    - End-to-end exactly-once requires idempotent consumer too
    - Performance cost: use only when business requires it

- [ ] **message-ordering**
    - Kafka: ordering guaranteed within a partition only
    - Partition by entity ID (e.g., user ID) to preserve per-entity order
    - Global ordering: single partition (limits throughput)
    - Sequence numbers for detecting and reordering out-of-order messages

- [ ] **consumer-groups**
    - Group ID: each group gets all messages (fan-out)
    - Within group: each partition assigned to one consumer
    - Rebalancing on join/leave — causes pause in consumption
    - Static membership to reduce rebalance frequency

- [ ] **event-schema-evolution**
    - Backward compatible: new optional fields (consumers can ignore)
    - Forward compatible: consumers handle unknown fields
    - Full compatibility: both directions
    - Confluent Schema Registry + Avro/Protobuf for enforcement

- [ ] **outbox-pattern**
    - Atomically write to DB + outbox table in same transaction
    - Polling publisher reads outbox and publishes to Kafka
    - Solves dual-write problem (DB write + message publish)
    - Debezium CDC as outbox implementation

- [ ] **transactional-messaging**
    - Problem: DB commit and message publish can't be atomic natively
    - Solutions: outbox pattern, change data capture (CDC)
    - Saga + messaging for distributed transactions
    - At-least-once delivery with idempotent handlers

- [ ] **dead-letter-queue**
    - Messages that fail processing after N retries go to DLQ
    - DLQ enables: investigation, reprocessing, alerting
    - Separate DLQ per consumer for isolation
    - Never silently discard failed messages

- [ ] **replayability**
    - Kafka log as source of truth: replay from offset 0
    - Use cases: rebuild projections, debug, audit
    - Retention period must match replay window needs
    - Idempotent consumers required for safe replay

📌 **Goal**: Build resilient, decoupled systems.

---

## Phase 5: Design Patterns & Resilience

📁 `design-patterns/`

- [ ] **singleton**
    - Ensure one instance: double-checked locking, enum-based, holder idiom
    - Thread safety in Java singleton implementations
    - Problems: global state, testability, hidden dependencies
    - Spring beans are singletons by default — implications

- [ ] **factory**
    - Factory Method: subclass decides which object to create
    - Abstract Factory: families of related objects
    - When to use: object creation complexity, type selection at runtime
    - Spring `@Bean` and `@Configuration` as factory pattern

- [ ] **strategy**
    - Encapsulate interchangeable algorithms behind interface
    - Select algorithm at runtime (payment provider, sorting, routing)
    - Eliminates large if-else/switch blocks
    - Spring DI for injecting strategy implementations

- [ ] **circuit-breaker**
    - States: Closed (normal) → Open (failing) → Half-Open (testing)
    - Failure threshold and timeout window configuration
    - Resilience4j implementation in Spring Boot
    - Fallback method when circuit is open

- [ ] **retry-pattern**
    - Exponential backoff with jitter to prevent thundering herd
    - Max retry limit and total timeout budget
    - Idempotency required for safe retries
    - Resilience4j `@Retry` annotation in Spring

- [ ] **bulkhead**
    - Isolate failures: thread pool per downstream service
    - Semaphore-based vs thread pool-based bulkhead
    - Prevent one slow dependency from starving others
    - Resilience4j `@Bulkhead` with `THREADPOOL` type

- [ ] **rate-limiting**
    - Algorithms: Token Bucket, Leaky Bucket, Fixed Window, Sliding Window
    - Distributed rate limiting: Redis-based counters
    - Limit per user, per IP, per API key
    - Return 429 with `Retry-After` header

- [ ] **load-balancing**
    - Client-side: Ribbon, Spring Cloud LoadBalancer
    - Server-side: Nginx, HAProxy, AWS ALB
    - Health checks drive load balancer routing decisions
    - Session stickiness and its downsides

- [ ] **backpressure**
    - Signal upstream to slow down when overwhelmed
    - Project Reactor: `onBackpressureBuffer`, `onBackpressureDrop`
    - Bounded queues as backpressure mechanism
    - Reactive Streams `Subscription.request(n)` protocol

📌 **Goal**: Survive partial failures.

---

## Phase 6: Architecture Patterns

📁 `architecture-patterns/`

### Architecture Patterns

- [ ] **monolith-vs-microservices**
    - Start with monolith: simpler, faster to build
    - Microservices: independent deploy, scale, and fault isolation
    - Distributed monolith: worst of both worlds — avoid
    - Migration: strangler fig, modular monolith first

- [ ] **database-per-service**
    - Each service owns its data — no shared tables
    - Polyglot persistence: each service picks best DB
    - Cross-service data: API calls or events, never direct DB access
    - Reporting challenge: need read-model aggregation

- [ ] **shared-database-anti-pattern**
    - Multiple services sharing one DB = tight coupling
    - Schema changes become multi-team coordination nightmares
    - Runtime coupling: one service's query harms others
    - Exception: deliberate read-only reporting database

- [ ] **sync-vs-async-communication**
    - Sync: HTTP/gRPC — simple, immediate response, temporal coupling
    - Async: messaging — decoupled, resilient, eventual
    - Sync for: queries, user-facing request/response
    - Async for: commands, events, background processing

- [ ] **event-driven-architecture**
    - Events as facts: immutable, past tense (`OrderPlaced`, not `PlaceOrder`)
    - Event notification vs event-carried state transfer
    - Event sourcing: events as source of truth, not current state
    - Choreography: services react to events (no central coordinator)

- [ ] **cqrs**
    - Separate read model (Query) from write model (Command)
    - Write side: handle commands, emit events, update write DB
    - Read side: consume events, update optimized read projections
    - Eventual consistency between write and read models

- [ ] **saga-pattern**
    - Distributed transactions without 2PC
    - Each step publishes event on success, triggers compensating action on failure
    - Choreography: event-driven, decentralized
    - Orchestration: central saga orchestrator controls flow

- [ ] **choreography-vs-orchestration**
    - Choreography: services react to events — decoupled but hard to track
    - Orchestration: orchestrator tells services what to do — centralized visibility
    - Temporal/Conductor for orchestration in Java
    - Use choreography for simple flows, orchestration for complex multi-step

- [ ] **api-gateway**
    - Single entry point: auth, rate-limit, routing, logging
    - Request aggregation and response transformation
    - BFF variant: gateway per client type (mobile, web)
    - Spring Cloud Gateway with predicates and filters

- [ ] **service-mesh**
    - Sidecar proxy (Envoy) handles: mTLS, retries, circuit breaking, tracing
    - Control plane (Istio) vs data plane (Envoy)
    - Offloads resilience from application code to infrastructure
    - Overhead: added latency, operational complexity

- [ ] **backend-for-frontend**
    - Dedicated backend per frontend type (mobile, web, 3rd party)
    - Aggregates and adapts data to frontend's specific needs
    - Reduces chattiness (fewer round trips from mobile)
    - Owned by the frontend team

- [ ] **hexagonal-architecture**
    - Core domain has no infrastructure dependencies
    - Ports (interfaces) + Adapters (implementations: DB, HTTP, messaging)
    - Inbound adapters: REST controllers, message consumers
    - Outbound adapters: repositories, HTTP clients, event publishers

- [ ] **strangler-fig-pattern**
    - Incrementally replace legacy system by intercepting calls
    - New functionality goes to new system; old routes to legacy
    - Feature by feature migration — no big bang rewrite
    - Proxy/facade layer routes between old and new

---

### Domain-Driven Design (DDD)

- [ ] **bounded-contexts**
    - A boundary within which a domain model is consistent
    - Same word means different things in different contexts (e.g., "User")
    - Each microservice should map to a bounded context
    - Context map: relationships between contexts

- [ ] **aggregates-and-entities**
    - Aggregate: cluster of objects treated as a unit
    - Aggregate root: single entry point for all external access
    - Invariants enforced within aggregate boundary
    - Keep aggregates small for concurrency and performance

- [ ] **domain-events**
    - Something that happened in the domain, past tense
    - Published at aggregate boundary, consumed by others
    - Drives eventual consistency across bounded contexts
    - Difference from integration events: scope and audience

- [ ] **ubiquitous-language**
    - Shared language between developers and domain experts
    - Code reflects domain terms exactly (no translation layer)
    - Reduces miscommunication and bugs from misunderstanding
    - The language lives inside a bounded context

- [ ] **context-mapping**
    - Partnership, Shared Kernel, Customer-Supplier, Conformist
    - Anti-Corruption Layer (ACL): translate between contexts
    - Open Host Service: provide well-defined API for others
    - Published Language: common schema (event schemas, Protobuf)

- [ ] **anti-corruption-layer**
    - Translates external model to internal domain model
    - Protects your domain from polluting external concepts
    - Implement at integration boundaries (consuming 3rd party APIs)
    - Adapter + Facade + Translator patterns combined

---

### Multi-Region & Global Systems

- [ ] **geo-replication-strategies**
    - Active-Passive: one region writes, others read
    - Active-Active: all regions write, conflict resolution needed
    - Follow-the-sun: route to nearest active region
    - Data replication lag across regions (10ms–100ms+ typical)

- [ ] **active-active-vs-active-passive**
    - Active-Passive: simple, no conflicts, failover time needed
    - Active-Active: zero downtime, complex conflict resolution
    - CRDTs and last-write-wins for conflict resolution
    - Active-Active for stateless services; harder for stateful

- [ ] **data-residency-and-compliance**
    - GDPR: EU data must stay in EU
    - Data sovereignty: data subject to laws of where it's stored
    - Sharding by region: EU users' data only in EU regions
    - Audit logs for data access and cross-border transfers

- [ ] **global-load-balancing**
    - GeoDNS: route to nearest region based on client IP
    - Anycast routing: same IP, multiple locations
    - Health checks drive traffic away from degraded regions
    - AWS Route 53, Cloudflare, Akamai GTM

- [ ] **conflict-resolution-in-multi-region**
    - Last-Write-Wins (LWW): simple but can lose data
    - Vector clocks: detect conflicts, present to user or business logic
    - CRDTs: data structures that merge automatically
    - Application-level resolution: e.g., shopping cart merge

---

### Architectural Decision Making

- [ ] **architecture-decision-records-adr**
    - Document: Context, Decision, Status, Consequences
    - Living documents — update when decision changes
    - Store in version control alongside code
    - Tool: adr-tools CLI, GitHub/Confluence templates

- [ ] **trade-off-analysis-frameworks**
    - ATAM (Architecture Tradeoff Analysis Method)
    - Fitness functions: automated architecture tests
    - Competing concerns matrix: consistency vs availability vs cost
    - "It depends" is the answer — document what it depends on

- [ ] **non-functional-requirements-nfr-gathering**
    - Performance: latency SLAs, throughput targets
    - Reliability: availability %, RTO, RPO
    - Scalability: expected growth, peak load
    - Security, compliance, maintainability, cost constraints

- [ ] **c4-model-for-diagramming**
    - Level 1: System Context — system + external actors
    - Level 2: Container — applications, databases, services
    - Level 3: Component — internal structure of a container
    - Level 4: Code — class diagrams (rarely needed)
    - Tools: Structurizal DSL, PlantUML, draw.io

- [ ] **architecture-runway-planning**
    - Technical enablers: infrastructure work before features
    - Architectural backlog alongside feature backlog
    - Just-enough architecture upfront (YAGNI vs technical debt)
    - Evolving architecture over time, not big upfront design

📌 **Goal**: Choose the right architecture at the right time.

---

## Phase 7: Security & Observability

📁 `security/`

### Security

- [ ] **authentication-vs-authorization**
    - AuthN: who are you? (identity verification)
    - AuthZ: what can you do? (permission check)
    - RBAC, ABAC, PBAC models
    - Principle of least privilege

- [ ] **oauth2-jwt**
    - OAuth2 flows: Authorization Code, Client Credentials, Device Flow
    - JWT structure: Header.Payload.Signature
    - Access token (short-lived) vs Refresh token (long-lived)
    - JWT validation: signature, expiry, issuer, audience claims

- [ ] **token-expiry-refresh**
    - Short access token TTL (15min) reduces exposure window
    - Refresh token rotation on every use
    - Refresh token revocation list (Redis-backed blacklist)
    - Silent refresh from client before token expiry

- [ ] **mTLS**
    - Both client AND server present certificates
    - Each service has its own certificate (issued by internal CA)
    - Service mesh (Istio) automates mTLS between services
    - Use for: service-to-service in zero-trust environments

- [ ] **api-security**
    - Input validation: reject malformed/unexpected data early
    - SQL injection, XSS, CSRF prevention
    - API rate limiting to prevent abuse
    - OWASP API Security Top 10

- [ ] **secrets-management**
    - Never store secrets in code or environment variables plaintext
    - HashiCorp Vault: dynamic secrets, lease-based rotation
    - AWS Secrets Manager / Parameter Store
    - Kubernetes Secrets + external-secrets-operator

- [ ] **encryption**
    - At rest: AES-256 for data, KMS for key management
    - In transit: TLS 1.3 minimum
    - End-to-end encryption: only endpoints can decrypt
    - Key rotation strategy and impact on encrypted data

- [ ] **zero-trust-architecture**
    - "Never trust, always verify" — no implicit trust for internal traffic
    - Every request authenticated + authorized regardless of network location
    - Micro-segmentation: limit blast radius of breach
    - mTLS + identity-aware proxy + short-lived credentials

- [ ] **ddos-protection**
    - Volumetric, Protocol, Application (L7) attack types
    - CDN absorbs volumetric attacks at edge
    - Rate limiting, CAPTCHAs, IP reputation filtering
    - AWS Shield, Cloudflare DDoS protection

---

### Observability

- [ ] **structured-logging**
    - JSON logs: machine-parseable, query-friendly
    - Standard fields: timestamp, level, service, trace_id, span_id, user_id
    - Log levels: ERROR (fix now), WARN (investigate), INFO (normal), DEBUG (verbose)
    - Logback + Logstash encoder in Spring Boot

- [ ] **log-correlation**
    - Trace ID propagated through all service calls (MDC in Java)
    - Inject trace ID in HTTP headers (`X-Trace-Id`)
    - Correlate logs across services by trace ID
    - Spring Cloud Sleuth / Micrometer Tracing for auto-propagation

- [ ] **monitoring**
    - Whitebox monitoring: internal metrics (JVM, DB pool, queue depth)
    - Blackbox monitoring: external checks (can users log in?)
    - Prometheus + Grafana stack in Java ecosystem
    - Alert on symptoms (user impact), not just causes

- [ ] **metrics**
    - Counter: ever-increasing (requests served, errors)
    - Gauge: current value (queue depth, active connections)
    - Histogram: distribution of values (latency buckets)
    - Micrometer in Spring Boot with Prometheus registry

- [ ] **golden-signals**
    - Latency: how long requests take (P50, P99)
    - Traffic: how many requests per second
    - Errors: rate of failed requests
    - Saturation: how full is your system (CPU, memory, queue depth)

- [ ] **distributed-tracing**
    - Trace: end-to-end request across services
    - Span: single unit of work within a trace
    - Context propagation via headers (W3C TraceContext standard)
    - Jaeger / Zipkin / AWS X-Ray as backends

- [ ] **health-checks**
    - Liveness: is the process alive? (restart if failed)
    - Readiness: is the process ready to serve traffic? (remove from LB if failed)
    - Spring Boot Actuator `/actuator/health` with groups
    - Include dependency checks: DB, Redis, downstream services

- [ ] **alert-fatigue**
    - Too many alerts → on-call ignores them
    - Alert only on user-impacting symptoms
    - Silence noisy alerts, fix root cause or raise threshold
    - Runbooks for every alert — what to do when it fires

- [ ] **slo-error-budgets**
    - SLO: e.g., 99.9% requests succeed within 200ms over 30 days
    - Error budget: 100% - SLO = allowed failure budget
    - When budget exhausted: freeze features, focus on reliability
    - Error budget drives the conversation between dev and ops

---

### Reliability Engineering

- [ ] **chaos-engineering-principles**
    - Hypothesis: system behaves normally under X failure condition
    - Inject failures in controlled way: kill instances, add latency, drop packets
    - Start small: staging → canary → production
    - Chaos Monkey, Gremlin, AWS Fault Injection Simulator

- [ ] **game-days-and-fire-drills**
    - Simulate failure scenarios with full team
    - Test runbooks, on-call procedures, escalation paths
    - Measure time-to-detect and time-to-recover
    - Document gaps found and fix them

- [ ] **runbooks-and-playbooks**
    - Runbook: step-by-step instructions for specific scenarios
    - Playbook: broader incident response guide
    - Every alert must have an associated runbook
    - Keep runbooks in version control, review quarterly

- [ ] **incident-management-lifecycle**
    - Detect → Triage → Communicate → Mitigate → Resolve → Review
    - Incident commander role: owns communication and coordination
    - Status page updates: keep users informed
    - Time-boxing: escalate if not resolved in N minutes

- [ ] **post-mortem-culture-blameless**
    - Focus on systems and processes, not individuals
    - 5 Whys root cause analysis
    - Action items with owners and deadlines
    - Share post-mortems widely — learning for whole org

📌 **Goal**: Understand production behavior and engineer for failure.

---

## Phase 8: Performance & Optimization

📁 `performance/`

- [ ] **bottleneck-identification**
    - USE Method: Utilization, Saturation, Errors per resource
    - RED Method: Rate, Errors, Duration per service
    - Flamegraphs for CPU profiling in Java (async-profiler)
    - Thread dump analysis for blocking/deadlock detection

- [ ] **optimization-techniques**
    - Measure before optimizing — premature optimization is the root of all evil
    - Caching, batching, async processing, connection pooling
    - Reduce serialization overhead (binary formats over JSON)
    - Lazy loading vs eager loading trade-offs

- [ ] **database-query-optimization**
    - EXPLAIN ANALYZE: read query plans, find seq scans
    - Index usage: ensure queries use indexes (avoid full table scans)
    - N+1 problem: detect with Hibernate statistics, fix with JOIN FETCH
    - Read replicas for offloading heavy read queries

- [ ] **async-vs-blocking-io**
    - Blocking: thread waits idle during I/O — wastes thread resources
    - Non-blocking: thread freed during I/O wait, handles other requests
    - Java NIO, Netty, Spring WebFlux (Reactor) for non-blocking
    - Use async for: high-concurrency, many slow I/O calls

- [ ] **reactive-vs-imperative**
    - Imperative: sequential, blocking, easy to reason about
    - Reactive: event-driven, non-blocking, backpressure support
    - Project Reactor: `Mono` (0-1) and `Flux` (0-N) types
    - Use reactive only when you need async non-blocking throughout

---

### Cost Architecture & FinOps

- [ ] **finops-fundamentals**
    - FinOps = financial accountability for cloud spend
    - Three phases: Inform (visibility), Optimize (efficiency), Operate (culture)
    - Unit economics: cost per request, cost per user, cost per transaction
    - Cloud cost anomaly detection and alerting

- [ ] **cost-modeling-for-cloud-resources**
    - Compute: instance type × hours × quantity
    - Storage: GB stored + GB transferred
    - Network egress is often the hidden cost — model it separately
    - Build cost model before architecture decisions, not after

- [ ] **cost-vs-performance-trade-offs**
    - More caching = lower DB cost but higher Redis cost
    - Precomputed aggregates = lower query cost, higher storage cost
    - Reserved capacity (1–3 year) vs on-demand: 40–70% savings
    - Right-sizing: don't run m5.2xlarge for 10% CPU workload

- [ ] **reserved-vs-on-demand-vs-spot**
    - On-Demand: pay as you go, no commitment, most expensive
    - Reserved: 1-3 year commitment, 40-60% discount
    - Spot/Preemptible: 80-90% discount, can be terminated anytime
    - Use Spot for: batch jobs, stateless workers, non-critical tasks

- [ ] **cost-tagging-and-showback**
    - Tag every resource: team, environment, feature, cost-center
    - Showback: report costs per team without charging
    - Chargeback: allocate costs back to teams/products
    - AWS Cost Explorer, GCP Billing Reports with label filters

- [ ] **right-sizing-services**
    - Monitor actual CPU/memory utilization (Cloudwatch, Prometheus)
    - Downsize over-provisioned instances (common: 80% at 10% utilization)
    - AWS Compute Optimizer, GCP Recommender
    - Scale-to-zero for dev/staging environments

📌 **Goal**: Architects must justify systems to business stakeholders with cost data.

---

## Phase 9: Java-Specific Deep Dive

📁 `java-specific/`

- [ ] **spring-boot-architecture**
    - Auto-configuration: `@SpringBootApplication` + classpath scanning
    - Application context lifecycle and bean lifecycle hooks
    - `@ConditionalOnProperty`, `@ConditionalOnBean` for config control
    - Actuator endpoints, banner, startup events

- [ ] **spring-cache-internals**
    - `@Cacheable`, `@CacheEvict`, `@CachePut` annotations
    - Cache abstraction: pluggable backends (Caffeine, Redis, EhCache)
    - Proxy-based: self-invocation won't hit cache
    - Serialization of keys and values to Redis

- [ ] **microservices-with-spring-cloud**
    - Service discovery: Eureka / Consul
    - Config server: centralized configuration management
    - Spring Cloud Gateway: routing, filtering, rate limiting
    - Spring Cloud LoadBalancer: client-side load balancing

- [ ] **java-concurrency**
    - `synchronized`, `ReentrantLock`, `ReadWriteLock`
    - `volatile` keyword: visibility, not atomicity
    - Atomic classes: `AtomicInteger`, `AtomicReference`, `LongAdder`
    - `CompletableFuture`: async composition and exception handling

- [ ] **thread-pools**
    - `ThreadPoolExecutor`: corePoolSize, maxPoolSize, queue, keep-alive
    - Queue types: `LinkedBlockingQueue` (unbounded — dangerous) vs `ArrayBlockingQueue`
    - Virtual threads (Java 21): lightweight, no need to pool
    - Sizing: I/O-bound = larger pool, CPU-bound = core count + 1

- [ ] **jvm-memory-model**
    - Heap: Young (Eden + Survivor) + Old Gen
    - Non-Heap: Metaspace, Code Cache, Direct Buffers
    - Stack: per-thread, stores frames and local variables
    - JVM Happens-Before guarantees for thread safety

- [ ] **gc-algorithms**
    - Serial, Parallel GC: throughput-focused, stop-the-world
    - G1GC: default since Java 9, region-based, predictable pause
    - ZGC / Shenandoah: sub-millisecond pauses, large heaps
    - GC log analysis: `GCViewer`, `GCEasy`

- [ ] **jvm-performance-tuning**
    - `-Xms` / `-Xmx`: set equal to avoid heap resizing
    - GC tuning flags: `-XX:+UseG1GC`, `-XX:MaxGCPauseMillis`
    - JIT compiler: C1 (fast compile) + C2 (optimized compile)
    - JMH for microbenchmarking Java code

- [ ] **memory-leaks**
    - Common causes: static collections, unclosed streams, listener leaks, cache without TTL
    - Detection: heap dump analysis with MAT / VisualVM / IntelliJ
    - `WeakReference` / `SoftReference` for cache implementations
    - Monitor GC overhead: if >5% CPU on GC, investigate

📌 **Goal**: Connect JVM internals to system behavior.

---

## Phase 10: Distributed Systems

📁 `distributed-systems/`

- [ ] **consensus-algorithms**
    - Problem: nodes must agree on a value despite failures
    - Paxos: foundational but complex (multi-paxos for log replication)
    - Raft: simpler, understandable, widely implemented (etcd, CockroachDB)
    - Leader election is a consensus problem

- [ ] **leader-election**
    - Bully algorithm: highest ID wins (simple, expensive)
    - Raft leader election: candidate requests votes
    - Zookeeper ephemeral nodes for distributed leader election
    - Must handle split-brain: fencing tokens, epoch numbers

- [ ] **quorum**
    - Quorum = majority: N/2 + 1 nodes must agree
    - Read quorum + Write quorum > N → strong consistency
    - Tunable consistency: R=1, W=1 (fast), R=N, W=N (strong)
    - Quorum prevents split-brain in distributed writes

- [ ] **split-brain**
    - Network partition causes nodes to think others are down
    - Two leaders can independently accept writes → divergence
    - Prevention: quorum requirement, STONITH (shoot the other node)
    - Fencing tokens: monotonically increasing, reject old tokens

- [ ] **distributed-locks**
    - Redis SETNX + TTL: simple but requires Redlock for safety
    - Redlock: acquire lock on N/2+1 Redis nodes
    - Zookeeper ephemeral sequential nodes for fair locking
    - Always set TTL — leaked locks cause deadlocks

- [ ] **distributed-transactions**
    - Two-Phase Commit (2PC): coordinator + participants, blocking
    - Saga: compensating transactions, no coordinator needed
    - TCC (Try-Confirm-Cancel): explicit phases in application code
    - Best-effort eventual consistency for most microservices

- [ ] **vector-clocks** *(see Phase 1 — advanced application here)*
    - Used for conflict detection in Dynamo-style DBs
    - Amazon shopping cart: vector clock per cart item
    - Pruning vector clocks: remove old entries
    - Dotted version vectors: improvement over vector clocks

- [ ] **gossip-protocol**
    - Peer-to-peer information dissemination
    - Each node periodically selects random peers to exchange state
    - Convergence: O(log N) rounds to reach all nodes
    - Used in: Cassandra (ring topology), Redis Cluster, Consul

📌 **Goal**: Think like a distributed systems engineer.

---

## Phase 11: Real-World System Design Cases

📁 `system-design-cases/`

- [ ] **url-shortener**
    - Hash function for generating short codes (base62)
    - 301 (permanent) vs 302 (temporary) redirect strategy
    - Read-heavy system: cache popular URLs aggressively
    - Custom aliases, expiry, analytics counting

- [ ] **rate-limiter-design**
    - Token bucket vs sliding window log vs sliding window counter
    - Distributed rate limiting with Redis (INCR + EXPIRE)
    - Per-user, per-IP, per-API-key limits
    - HTTP 429 with `Retry-After` header

- [ ] **notification-system**
    - Channels: push, email, SMS, in-app
    - Fan-out to millions of users: partition by user_id
    - Template rendering service for personalized content
    - Delivery tracking, retry on failure, DLQ for failed deliveries

- [ ] **file-upload-download-system**
    - Pre-signed URLs for direct S3/GCS upload (bypass backend)
    - Chunked upload for large files with resume support
    - Virus scanning and validation in async pipeline
    - CDN for download acceleration

- [ ] **distributed-cache-design**
    - Consistent hashing for data distribution across cache nodes
    - Eviction policies and memory management
    - Cache replication for HA
    - Client-side vs proxy vs sidecar cache topology

- [ ] **payment-processing-system**
    - Idempotency keys for all payment operations
    - Exactly-once processing for financial transactions
    - Reconciliation process: detect and fix discrepancies
    - PCI DSS compliance requirements

- [ ] **bank-statement-processing-system**
    - Batch processing: scheduled jobs for end-of-day statement generation
    - Pagination for large statement data
    - Audit trail: immutable event log of all transactions
    - Async PDF generation with polling or webhook notification

- [ ] **multi-tenant-saas-design**
    - Isolation models: shared DB, schema-per-tenant, DB-per-tenant
    - Tenant context propagation through request pipeline
    - Rate limiting and resource quotas per tenant
    - Data isolation: row-level security or separate schemas

---

### Advanced Cases

- [ ] **social-media-feed-design**
    - Fan-out on write (push) vs fan-out on read (pull)
    - Hybrid: push for regular users, pull for celebrities
    - Feed ranking: ML-based, reverse-chronological fallback
    - Cache user feeds, evict on new post from followees

- [ ] **search-engine-design**
    - Crawler → Parser → Indexer → Ranker → Query Processor
    - Inverted index: term → list of documents
    - TF-IDF and PageRank for relevance scoring
    - Elasticsearch as implementation reference

- [ ] **real-time-collaborative-editing**
    - Operational Transformation (OT): Google Docs approach
    - CRDTs: conflict-free replicated data types, no server coordination
    - WebSocket for real-time sync between clients
    - Cursor presence: broadcast cursor positions

- [ ] **ride-sharing-platform**
    - Geospatial indexing: geohash or quadtree for driver lookup
    - Matching algorithm: nearest available driver
    - Real-time location tracking: WebSocket or Server-Sent Events
    - Surge pricing: supply/demand ratio computation

- [ ] **food-delivery-system**
    - Order state machine: Placed → Confirmed → Preparing → Picked Up → Delivered
    - Three-sided marketplace: customer, restaurant, driver
    - ETA calculation: ML model with live traffic data
    - Notification at each state transition

📌 **Goal**: Practice end-to-end designs across multiple dimensions.

---

## Phase 12: Cloud & Infrastructure

📁 `cloud-infrastructure/`

### Containers & Orchestration

- [ ] **docker-fundamentals**
    - Overview, images, containers, layers, and registries
    - Java Dockerfile patterns, multi-stage builds, and `.dockerignore`
    - Storage, networking, Compose, runtime config, health, and logging
    - Security, troubleshooting, and Docker in system design

- [ ] **kubernetes-basics**
    - Pod, Deployment, Service, ConfigMap, Secret, Namespace
    - kubectl: apply, get, describe, logs, exec
    - ReplicaSet ensures desired pod count
    - Service types: ClusterIP, NodePort, LoadBalancer

- [ ] **kubernetes-advanced**
    - HPA (Horizontal Pod Autoscaler): scale on CPU/custom metrics
    - PodDisruptionBudget for safe rolling updates
    - Network Policies for pod-level firewall rules
    - StatefulSets for databases, stable pod identities

- [ ] **helm-charts**
    - Package Kubernetes manifests as reusable charts
    - Values files for environment-specific config
    - `helm install`, `upgrade`, `rollback`
    - Helm hooks for pre/post install jobs

---

### Cloud Providers

- [ ] **aws-fundamentals**
    - Core: EC2, S3, RDS, VPC, IAM, CloudWatch
    - Load Balancing: ALB (L7), NLB (L4), CLB (legacy)
    - Auto Scaling Groups with launch templates
    - Well-Architected Framework: 6 pillars

- [ ] **azure-basics**
    - Compute: AKS, App Service, Azure Functions, VMs
    - Storage: Blob Storage, Azure SQL, Cosmos DB
    - Identity: Azure AD, Managed Identities
    - Azure DevOps for CI/CD pipelines

- [ ] **gcp-overview**
    - Compute: GKE, Cloud Run, Cloud Functions, Compute Engine
    - Storage: Cloud Storage, BigQuery, Cloud SQL, Spanner
    - Network: Cloud Load Balancing, Cloud CDN
    - Anthos for hybrid/multi-cloud management

- [ ] **cloud-databases**
    - Managed RDBMS: RDS, Cloud SQL, Azure SQL (patching, backups handled)
    - Serverless: Aurora Serverless, DynamoDB, Firestore
    - NewSQL: Spanner, CockroachDB (distributed + ACID)
    - Multi-region replication and consistency trade-offs

- [ ] **cloud-storage-patterns**
    - Object storage: S3, GCS — immutable, cheap, eventually consistent
    - Block storage: EBS, Persistent Disk — attached to single VM, low latency
    - File storage: EFS, Filestore — shared across instances, NFS-based
    - Storage classes/tiers for cost optimization (Standard, IA, Glacier)

---

### Infrastructure as Code & DevOps

- [ ] **terraform**
    - HCL declarative syntax: resources, variables, outputs, modules
    - State management: remote state in S3, state locking with DynamoDB
    - Plan → Apply → Destroy workflow
    - Modules for reusable infrastructure components

- [ ] **cloudformation**
    - AWS-native IaC: JSON/YAML templates
    - Stacks and nested stacks
    - Change sets: preview changes before applying
    - StackSets for multi-account/multi-region deployment

- [ ] **ansible**
    - Agentless: SSH-based push model
    - Playbooks → Roles → Tasks structure
    - Idempotent operations: safe to run multiple times
    - Use for: configuration management, app deployment

- [ ] **gitops**
    - Git as single source of truth for infrastructure state
    - ArgoCD / Flux: auto-sync cluster to Git state
    - Pull-based model: cluster pulls from Git (not push)
    - Drift detection: alert when cluster diverges from Git

- [ ] **ci-cd-pipelines**
    - CI: build → test → static analysis → security scan → package
    - CD: deploy to dev → integration tests → staging → canary → production
    - Pipeline as code: Jenkinsfile, GitHub Actions, GitLab CI
    - Artifact versioning and promotion strategy

- [ ] **blue-green-deployment**
    - Two identical environments: Blue (live), Green (new version)
    - Cut traffic to Green instantly via load balancer
    - Instant rollback: switch back to Blue
    - Doubles infrastructure cost during transition

- [ ] **canary-releases**
    - Route small % of traffic to new version (1% → 10% → 100%)
    - Monitor error rates and latency before full rollout
    - Automatic rollback if metrics degrade
    - Feature-level canary vs service-level canary

- [ ] **feature-flags**
    - Decouple deployment from release
    - Kill switch: disable feature instantly without redeploy
    - A/B testing and gradual rollout by user segment
    - LaunchDarkly, Unleash, Spring Cloud Feature Flags

- [ ] **auto-scaling**
    - Reactive scaling: scale out when CPU > 70%, scale in when < 30%
    - Predictive scaling: ML-based, scale before load hits
    - Cooldown periods to prevent flapping
    - Scale-to-zero for serverless (Lambda, Cloud Run)

- [ ] **elastic-load-balancing**
    - ALB: HTTP/HTTPS, path-based routing, WAF integration
    - NLB: TCP/UDP, ultra-low latency, static IPs
    - Health checks: interval, threshold, healthy/unhealthy criteria
    - Connection draining: graceful in-flight request completion

📌 **Goal**: Build cloud-native systems.

---
