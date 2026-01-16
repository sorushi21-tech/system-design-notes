# System Design Study Plan

> As java backend developer, I want extend my knowledge and master System Design concepts from basics to advanced.

## Overview

This repository I have created to track my learning and notes.

## Learning Objectives

- Understand core system design principles
- Understand scalability, reliability, and performance optimization
- Learn architectural patterns for building distributed systems
- Apply Java/Spring Boot best practices in system design

---

## Study Plan

### Foundations

#### Core Fundamentals
1. [Scalability](fundamentals/scalability.md)
2. [Latency vs Throughput](fundamentals/latency-vs-throughput.md)
3. [Horizontal vs Vertical Scaling](fundamentals/horizontal-vs-vertical-scaling.md)
4. [Availability and Reliability](fundamentals/availability-reliability.md)
5. [Fault Tolerance](fundamentals/fault-tolerance.md)

#### Data & Consistency
1. [CAP Theorem](fundamentals/cap-theorem.md)
2. [Consistency Models](fundamentals/consistency-models.md)
3. [Data Partitioning](fundamentals/data-partitioning.md)
4. [Load Balancing Algorithms](fundamentals/load-balancing-algorithms.md)

#### Networking Basics
1. [DNS](networking/dns.md)
2. [HTTP and HTTPS](networking/http-https.md)
3. [TCP vs UDP](networking/tcp-vs-udp.md)
4. [Proxies and Reverse Proxies](networking/proxies-reverse-proxies.md)
5. [CDN](networking/cdn.md)

---

### Storage & Data Management

#### Database Fundamentals
1. [SQL vs NoSQL](databases/sql-vs-nosql.md)
2. [ACID vs BASE](databases/acid-vs-base.md)
3. [Database Transactions](databases/transactions.md)
4. [Indexing](databases/indexing.md)
5. [Normalization](databases/normalization.md)

#### Database Scalability
1. [Replication](databases/replication.md)
2. [Sharding](databases/sharding.md)
3. [Connection Pooling](databases/connection-pooling.md)
4. [NoSQL Types](databases/nosql-types.md)

#### Caching Strategies
1. [Redis Basics](caching/redis-basics.md)
2. [Cache-Aside](caching/cache-aside.md)
3. [Write-Through](caching/write-through.md)
4. [Write-Behind](caching/write-behind.md)
5. [Read-Through](caching/read-through.md)
6. [Cache Invalidation](caching/cache-invalidation.md)
7. [Distributed Caching](caching/distributed-caching.md)
8. [Multi-Tenant Caching](caching/multi-tenant-caching.md)

---

### Communication & Messaging

#### API Design
1. [REST API Design](api-design/rest-api-design.md)
2. [API Versioning](api-design/api-versioning.md)
3. [Pagination](api-design/pagination.md)
4. [GraphQL](api-design/graphql.md)
5. [gRPC](api-design/grpc.md)

#### Asynchronous Communication
1. [Kafka](messaging/kafka.md)
2. [RabbitMQ](messaging/rabbitmq.md)
3. [Pub-Sub Pattern](messaging/pub-sub-pattern.md)
4. [Message Queue vs Event Streaming](messaging/message-queue-vs-event-streaming.md)
5. [Exactly-Once Delivery](messaging/exactly-once.md)
6. [Message Ordering](messaging/message-ordering.md)
7. [Dead Letter Queue](messaging/dead-letter-queue.md)

---

### Design Patterns & Resilience

#### Core Design Patterns
1. [Singleton](design-patterns/singleton.md)
2. [Factory](design-patterns/factory.md)
3. [Strategy](design-patterns/strategy.md)
4. [Circuit Breaker](design-patterns/circuit-breaker.md)

#### Resilience Patterns
1. [Retry Pattern](design-patterns/retry-pattern.md)
2. [Bulkhead](design-patterns/bulkhead.md)
3. [Rate Limiting](design-patterns/rate-limiting.md)
4. [Load Balancing](design-patterns/load-balancing.md)
5. [Backpressure](design-patterns/backpressure.md)

#### Critical Patterns
1. [Idempotency](design-patterns/idempotency.md)
2. [WebSockets](networking/websockets.md)

---

### Architecture Patterns

#### Modern Architecture
1. [Monolith vs Microservices](architecture-patterns/monolith-vs-microservices.md)
2. [Event-Driven Architecture](architecture-patterns/event-driven-architecture.md)
3. [CQRS](architecture-patterns/cqrs.md)
4. [Saga Pattern](architecture-patterns/saga-pattern.md)

#### Advanced Patterns
1. [API Gateway](architecture-patterns/api-gateway.md)
2. [Service Mesh](architecture-patterns/service-mesh.md)
3. [Backend for Frontend](architecture-patterns/backend-for-frontend.md)
4. [Hexagonal Architecture](architecture-patterns/hexagonal-architecture.md)

#### Migration & Strategy
1. [Strangler Fig Pattern](architecture-patterns/strangler-fig-pattern.md)

---

### Security & Observability

#### Security
1. [Authentication vs Authorization](security/authentication-vs-authorization.md)
2. [OAuth2 and JWT](security/oauth2-jwt.md)
3. [API Security](security/api-security.md)
4. [Encryption](security/encryption.md)
5. [DDoS Protection](security/ddos-protection.md)

#### Observability
1. [Logging](observability/logging.md)
2. [Monitoring](observability/monitoring.md)
3. [Metrics](observability/metrics.md)
4. [Distributed Tracing](observability/distributed-tracing.md)
5. [Health Checks](observability/health-checks.md)

---

### Performance & Optimization

1. [Performance Optimization Techniques](performance/optimization-techniques.md)
2. [Database Query Optimization](performance/database-query-optimization.md)
3. [Bottleneck Identification](performance/bottleneck-identification.md)

---

### Storage Systems

1. [Object Storage](storage/object-storage.md)
2. [Blob Storage](storage/blob-storage.md)
3. [Data Lakes vs Data Warehouses](storage/data-lakes-vs-warehouses.md)

---

### Java-Specific Deep Dive

#### Spring Ecosystem
1. [Spring Boot Architecture](java-specific/spring-boot-architecture.md)
2. [Microservices with Spring Cloud](java-specific/microservices-with-spring-cloud.md)

#### JVM & Concurrency
1. [JVM Performance Tuning](java-specific/jvm-performance-tuning.md)
2. [Java Concurrency](java-specific/java-concurrency.md)

---

### Distributed Systems

Advanced topics for building large-scale distributed systems.

1. [Consensus Algorithms](distributed-systems/consensus-algorithms.md)
2. [Distributed Locks](distributed-systems/distributed-locks.md)
3. [Vector Clocks](distributed-systems/vector-clocks.md)
4. [Gossip Protocol](distributed-systems/gossip-protocol.md)

---

### Real-World System Design Cases

Practice designing complete systems from scratch.

#### Fundamental Systems
1. [URL Shortener](system-design-cases/url-shortener.md)
2. [Rate Limiter](system-design-cases/rate-limiter.md)
3. [Distributed Cache](system-design-cases/distributed-cache.md)

#### Communication Systems
1. [Chat System](system-design-cases/chat-system.md)
2. [Notification Service](system-design-cases/notification-service.md)

#### Content Systems
1. [Social Media Feed](system-design-cases/social-media-feed.md)
2. [Search Engine](system-design-cases/search-engine.md)

#### Storage & Media
1. [File Storage System](system-design-cases/file-storage-system.md)
2. [Video Streaming](system-design-cases/video-streaming.md)

#### Business Applications
1. [E-Commerce Platform](system-design-cases/e-commerce-platform.md)
2. [Payment System](system-design-cases/payment-system.md)
3. [Booking System](system-design-cases/booking-system.md)
4. [Multi-Tenant SaaS](system-design-cases/multi-tenant-saas.md)

---


## Progress Tracking

Create a simple checklist to track your progress:

- [ ] Phase 1: Foundations 
- [ ] Phase 2: Storage & Data Management
- [ ] Phase 3: Communication & Messaging
- [ ] Phase 4: Design Patterns & Resilience
- [ ] Phase 5: Architecture Patterns
- [ ] Phase 6: Security & Observability
- [ ] Phase 7: Performance & Optimization
- [ ] Phase 8: Storage Systems
- [ ] Phase 9: Java-Specific Deep Dive
- [ ] Phase 10: Distributed Systems
- [ ] Phase 11: Real-World System Design Cases

---