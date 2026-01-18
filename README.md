# System Design Study Plan 

> As java backend developer, I want extend my knowledge and master System Design concepts from basics to advanced.

---

## Learning Goals

* Understand **core system design principles**
* Design **scalable, reliable, and high-performance systems**
* Learn **distributed systems internals**
* Apply **Java, Spring Boot, and cloud-native best practices**
* Be confident in **System Design and real production systems**

---

## Phase 1: Foundations

### Core Fundamentals

📁 `fundamentals/`

* [ ] `scalability`
* [ ] `latency-vs-throughput`
* [ ] `horizontal-vs-vertical-scaling`
* [ ] `availability-reliability`
* [ ] `fault-tolerance`
* [ ] `backpressure-basics`
* [ ] `graceful-degradation`

📌 **Goal**: Understand *why* systems fail and *how* they scale.

---

### Capacity Estimation & System Math

📁 `fundamentals/`

* [ ] `capacity-estimation`
* [ ] `traffic-estimation`
* [ ] `storage-estimation`
* [ ] `qps-calculation`
* [ ] `latency-budgeting`
* [ ] `back-of-the-envelope-calculations`

📌 **Goal**: Size systems realistically and justify design decisions.

---

### Data & Consistency

📁 `fundamentals/`

* [ ] `cap-theorem`
* [ ] `consistency-models`
* [ ] `eventual-consistency-patterns`
* [ ] `data-partitioning`
* [ ] `hot-keys-problem`
* [ ] `denormalization-at-scale`
* [ ] `load-balancing-algorithms`

📌 **Goal**: Balance correctness, availability, and performance.

---

### Time, Ordering & Clocks

📁 `fundamentals/`

* [ ] `clock-skew`
* [ ] `ntp-time-sync`
* [ ] `lamport-clocks`
* [ ] `vector-clocks`
* [ ] `event-time-vs-processing-time`
* [ ] `out-of-order-events`

📌 **Goal**: Handle time correctly in distributed systems.

---

### Networking Basics

📁 `networking/`

* [ ] `dns`
* [ ] `http-https`
* [ ] `tcp-vs-udp`
* [ ] `connection-lifecycle`
* [ ] `keep-alive`
* [ ] `proxies-reverse-proxies`
* [ ] `cdn`

📌 **Goal**: Understand how requests flow across networks.

---

## Phase 2: Storage & Data Management

### Database Fundamentals

📁 `databases/`

* [ ] `sql-vs-nosql`
* [ ] `acid-vs-base`
* [ ] `transactions`
* [ ] `indexing`
* [ ] `normalization`
* [ ] `read-vs-write-optimized-models`

---

### Database Scalability

📁 `databases/`

* [ ] `replication`
* [ ] `sharding`
* [ ] `leader-election`
* [ ] `failover-strategies`
* [ ] `connection-pooling`
* [ ] `nosql-types`

📌 **Goal**: Scale data without breaking consistency.

---

## Phase 3: Caching

📁 `caching/`

* [✅] `redis-basics`
* [✅] `local-vs-distributed-cache`
* [✅] `two-level-cache`
* [✅] `cache-aside`
* [✅] `read-through`
* [✅] `write-through`
* [✅] `write-behind`
* [✅] `cache-invalidation`
* [✅] `cache-eviction-policies`
* [✅] `cache-stampede`
* [✅] `cache-penetration`
* [✅] `multi-tenant-caching`

📌 **Goal**: Reduce latency and DB load safely.

---

## Phase 4: Communication & Messaging

### API Design

📁 `api-design/`

* [ ] `rest-api-design`
* [ ] `api-versioning`
* [ ] `pagination`
* [ ] `idempotency`
* [ ] `graphql`
* [ ] `grpc`

---

### Asynchronous Communication

📁 `messaging/`

* [ ] `kafka`
* [ ] `rabbitmq`
* [ ] `pub-sub-pattern`
* [ ] `message-queue-vs-event-streaming`
* [ ] `at-least-once-vs-at-most-once`
* [ ] `exactly-once`
* [ ] `message-ordering`
* [ ] `consumer-groups`
* [ ] `event-schema-evolution`
* [ ] `outbox-pattern`
* [ ] `transactional-messaging`
* [ ] `dead-letter-queue`
* [ ] `replayability`

📌 **Goal**: Build resilient, decoupled systems.

---

## Phase 5: Design Patterns & Resilience

📁 `design-patterns/`

* [ ] `singleton`
* [ ] `factory`
* [ ] `strategy`
* [ ] `circuit-breaker`
* [ ] `retry-pattern`
* [ ] `bulkhead`
* [ ] `rate-limiting`
* [ ] `load-balancing`
* [ ] `backpressure`

📌 **Goal**: Survive partial failures.

---

## Phase 6: Architecture Patterns

📁 `architecture-patterns/`

* [ ] `monolith-vs-microservices`
* [ ] `database-per-service`
* [ ] `shared-database-anti-pattern`
* [ ] `sync-vs-async-communication`
* [ ] `event-driven-architecture`
* [ ] `cqrs`
* [ ] `saga-pattern`
* [ ] `choreography-vs-orchestration`
* [ ] `api-gateway`
* [ ] `service-mesh`
* [ ] `backend-for-frontend`
* [ ] `hexagonal-architecture`
* [ ] `strangler-fig-pattern`

📌 **Goal**: Choose the right architecture at the right time.

---

## Phase 7: Security & Observability

### Security

📁 `security/`

* [ ] `authentication-vs-authorization`
* [ ] `oauth2-jwt`
* [ ] `token-expiry-refresh`
* [ ] `mTLS`
* [ ] `api-security`
* [ ] `secrets-management`
* [ ] `encryption`
* [ ] `zero-trust-architecture`
* [ ] `ddos-protection`

---

### Observability

📁 `observability/`

* [ ] `structured-logging`
* [ ] `log-correlation`
* [ ] `monitoring`
* [ ] `metrics`
* [ ] `golden-signals`
* [ ] `distributed-tracing`
* [ ] `health-checks`
* [ ] `alert-fatigue`
* [ ] `slo-error-budgets`

📌 **Goal**: Understand production behavior.

---

## Phase 8: Performance & Optimization

📁 `performance/`

* [ ] `bottleneck-identification`
* [ ] `optimization-techniques`
* [ ] `database-query-optimization`
* [ ] `async-vs-blocking-io`
* [ ] `reactive-vs-imperative`

---

## Phase 9: Java-Specific Deep Dive

📁 `java-specific/`

* [ ] `spring-boot-architecture`
* [ ] `spring-cache-internals`
* [ ] `microservices-with-spring-cloud`
* [ ] `java-concurrency`
* [ ] `thread-pools`
* [ ] `jvm-memory-model`
* [ ] `gc-algorithms`
* [ ] `jvm-performance-tuning`
* [ ] `memory-leaks`

📌 **Goal**: Connect JVM internals to system behavior.

---

## Phase 10: Distributed Systems

📁 `distributed-systems/`

* [ ] `consensus-algorithms`
* [ ] `leader-election`
* [ ] `quorum`
* [ ] `split-brain`
* [ ] `distributed-locks`
* [ ] `distributed-transactions`
* [ ] `vector-clocks`
* [ ] `gossip-protocol`

📌 **Goal**: Think like a distributed systems engineer.

---

## Phase 11: Real-World System Design Cases

📁 `system-design-cases/`

* [ ] `url-shortener`
* [ ] `rate-limiter-design`
* [ ] `notification-system`
* [ ] `file-upload-download-system`
* [ ] `distributed-cache-design`
* [ ] `payment-processing-system`
* [ ] `bank-statement-processing-system`
* [ ] `multi-tenant-saas-design`

📌 **Goal**: Practice end-to-end designs.

---

## Phase 12: Cloud & Infrastructure

📁 `cloud-infrastructure/`

### Containers & Orchestration

* [ ] `docker-fundamentals`
* [ ] `kubernetes-basics`
* [ ] `kubernetes-advanced`
* [ ] `helm-charts`

### Cloud Providers

* [ ] `aws-fundamentals`
* [ ] `azure-basics`
* [ ] `gcp-overview`
* [ ] `cloud-databases`
* [ ] `cloud-storage-patterns`

### Infrastructure as Code & DevOps

* [ ] `terraform`
* [ ] `cloudformation`
* [ ] `ansible`
* [ ] `gitops`
* [ ] `ci-cd-pipelines`
* [ ] `blue-green-deployment`
* [ ] `canary-releases`
* [ ] `feature-flags`
* [ ] `auto-scaling`
* [ ] `elastic-load-balancing`

📌 **Goal**: Build cloud-native systems.

---

## Progress Checklist

* [ ] Phase 1: Foundations
* [ ] Phase 2: Storage & Data
* [ ] Phase 3: Caching
* [ ] Phase 4: Messaging
* [ ] Phase 5: Design Patterns
* [ ] Phase 6: Architecture
* [ ] Phase 7: Security & Observability
* [ ] Phase 8: Performance
* [ ] Phase 9: Java Deep Dive
* [ ] Phase 10: Distributed Systems
* [ ] Phase 11: System Design Cases
* [ ] Phase 12: Cloud & Infrastructure

---

