# Senior Java Backend on AWS

These notes focus on AWS decisions that matter for a senior Java backend developer. The goal is to connect Java and Spring Boot engineering choices with AWS infrastructure choices.

## 1. Senior Backend Mental Model

A senior backend developer is expected to think beyond code correctness.

You should be able to reason about:

- Runtime behavior under load
- Database connection limits
- Thread pools and blocking I/O
- Retry storms and timeout chains
- Idempotency and duplicate message handling
- Deployment rollback safety
- Observability during incidents
- Cost impact of architecture choices
- Security boundaries and IAM permissions

On AWS, the backend system is usually a combination of:

```text
Spring Boot service
  -> ALB/API Gateway
  -> ECS/EKS/Lambda/EC2
  -> RDS/Aurora/DynamoDB
  -> ElastiCache Redis
  -> SQS/SNS/EventBridge
  -> S3
  -> CloudWatch/X-Ray/OpenTelemetry
```

---

## 2. Choosing AWS Compute for Java Services

### ECS Fargate

Best default choice for many Spring Boot microservices.

Use when:

- You want containers without managing Kubernetes.
- Team is AWS-first.
- Services are stateless.
- You want simpler operations than EKS.

Watch for:

- Task CPU/memory sizing
- Cold image pulls during scale-out
- ALB health check configuration
- Container startup time
- CloudWatch log volume and cost

### EKS

Use when:

- Your organization standardizes on Kubernetes.
- You need Kubernetes ecosystem tools.
- You need advanced scheduling, service mesh, or custom controllers.
- You run many services with platform engineering support.

Watch for:

- Cluster upgrade burden
- IAM integration through IRSA
- Ingress controller configuration
- Pod resource requests and limits
- Operational complexity

### Lambda

Use for event-driven Java workloads only when the constraints fit.

Good for:

- SQS consumers
- EventBridge handlers
- Lightweight async jobs
- Scheduled tasks

Watch for:

- Java cold starts
- Package size
- Max execution timeout
- Database connection handling
- VPC cold start and networking configuration

For latency-sensitive Java APIs, ECS Fargate or EKS is often simpler than Lambda.

### EC2

Use when:

- You need full OS control.
- You run legacy Java apps.
- You need custom agents or special networking.

Watch for:

- AMI patching
- Auto Scaling Groups
- JVM and OS tuning
- Deployment automation

---

## 3. Spring Boot Runtime on AWS

Important production settings:

- Set active profile through environment variable: `SPRING_PROFILES_ACTIVE`.
- Expose health endpoints through Spring Boot Actuator.
- Separate liveness and readiness checks.
- Configure graceful shutdown.
- Use structured JSON logs.
- Propagate request IDs and trace IDs.
- Set JVM memory based on container memory.

Recommended Actuator endpoints:

```text
/actuator/health/liveness
/actuator/health/readiness
/actuator/metrics
/actuator/prometheus
```

Graceful shutdown matters because ECS/EKS/ALB may stop routing traffic before the process exits.

Example concerns:

- Stop accepting new requests.
- Finish in-flight requests.
- Stop message consumers.
- Close database connections.
- Flush logs and metrics.

---

## 4. RDS and Aurora for Java Backends

RDS/Aurora is the common choice for transactional Java systems.

Use for:

- Orders
- Payments
- Users
- Inventory
- Audit records
- Strong consistency workflows

### Connection Pooling

Java services usually use HikariCP.

Connection pool size must consider total application instances.

Example:

```text
20 ECS tasks x 20 DB connections = 400 DB connections
```

If the database supports 500 connections, this leaves little room for migrations, admin tools, replicas, and failover.

Guidelines:

- Keep pool size intentionally small.
- Use read replicas for heavy read workloads.
- Use RDS Proxy for Lambda or highly bursty workloads.
- Monitor active connections and wait time.
- Set connection timeout, idle timeout, and max lifetime carefully.

### Migration Safety

Use Flyway or Liquibase.

Rules:

- Migrations must be backward compatible.
- Avoid long locking migrations during peak traffic.
- Split large migrations into expand and contract phases.
- Deploy app code that supports old and new schema.
- Remove old columns only after all old code is gone.

### Aurora

Aurora is useful when:

- You need better read scaling.
- You want faster failover than standard RDS.
- You need managed replication and storage scaling.

Watch for:

- Writer/reader endpoint behavior
- Replica lag
- Failover impact on connection pools
- Query plan regressions

---

## 5. DynamoDB for Java Backends

DynamoDB is a managed NoSQL database.

Use when:

- Access patterns are known.
- You need high-scale key-value/document access.
- Single-digit millisecond latency is useful.
- You can avoid joins and complex ad hoc queries.

Important design points:

- Partition key decides data distribution.
- Bad partition key creates hot partitions.
- Model tables around access patterns.
- Use conditional writes for concurrency control.
- Use TTL for automatic item expiry.
- Use Streams for change events.

Java backend examples:

- Idempotency key table
- Rate limit counters
- User session metadata
- Event processing state
- High-scale lookup table

Do not choose DynamoDB just because it scales. Choose it when the query model fits.

---

## 6. ElastiCache Redis

Redis is commonly used with Java services for:

- Cache-aside
- Distributed rate limiting
- Short-lived sessions
- Leaderboards
- Temporary locks, with caution
- Hot reference data

Java/Spring concerns:

- Choose serialization carefully.
- Avoid huge keys and huge values.
- Set TTLs.
- Protect against cache stampede.
- Monitor memory, evictions, and connection count.
- Use timeouts on Redis calls.

Cache-aside pattern:

```text
Read request
  -> check Redis
  -> if hit, return
  -> if miss, read DB
  -> write Redis with TTL
  -> return
```

Failure rule:

For many systems, Redis failure should degrade performance, not take down the entire service. Decide which cache operations are critical and which can fail open.

---

## 7. Messaging on AWS

### SQS

SQS is a queue for asynchronous processing.

Use for:

- Background jobs
- Decoupling services
- Retryable work
- Buffering traffic spikes

Important concepts:

- Visibility timeout
- Dead-letter queue
- Message retention
- Long polling
- FIFO vs standard queue

Java consumer rules:

- Make message processing idempotent.
- Set visibility timeout longer than processing time.
- Send failed messages to DLQ after limited retries.
- Monitor age of oldest message.
- Use backpressure by limiting consumer concurrency.

### SNS

SNS is pub/sub fanout.

Use when one event should notify multiple subscribers.

Example:

```text
OrderCreated event
  -> billing subscriber
  -> email subscriber
  -> analytics subscriber
```

### EventBridge

EventBridge is event routing with rules.

Use for:

- Domain events
- SaaS integration
- Scheduled events
- Cross-account event routing

EventBridge is often better than direct service-to-service calls when events should be loosely coupled.

---

## 8. S3 for Java Backends

Use S3 for:

- File uploads
- Generated reports
- PDFs
- Images
- Backups
- Data exports
- Audit archives

Best pattern for large files:

```text
Client asks backend for pre-signed URL
Backend returns pre-signed URL
Client uploads directly to S3
S3 event triggers async processing
```

Benefits:

- Backend does not stream large files.
- Upload scales with S3.
- Processing can be asynchronous.

Security rules:

- Use private buckets by default.
- Use pre-signed URLs with short expiry.
- Enable encryption.
- Use bucket policies carefully.
- Avoid public access unless explicitly required.

---

## 9. IAM for Backend Services

Use IAM roles, not static keys.

Patterns:

- ECS task role for ECS services
- IRSA for EKS pods
- Lambda execution role for Lambda functions
- EC2 instance profile for EC2 apps

Least privilege examples:

- Service that reads one S3 bucket should not have `s3:*`.
- SQS consumer should receive/delete from only its queue.
- App should read only required secrets.
- CI role should deploy only approved stacks/services.

Senior expectation:

You should review IAM policies as part of backend design, not treat them as deployment-only details.

---

## 10. Observability on AWS

Minimum observability stack:

- CloudWatch metrics
- CloudWatch logs
- X-Ray or OpenTelemetry traces
- ALB access logs
- RDS Performance Insights
- CloudTrail for audit

Application logs should include:

- Timestamp
- Request ID
- Trace ID
- User/tenant identifier when safe
- Endpoint
- Status code
- Latency
- Error details

Golden signals:

- Latency
- Traffic
- Errors
- Saturation

Java-specific metrics:

- JVM heap/non-heap usage
- GC pauses
- Thread pool usage
- HikariCP active/idle/waiting connections
- HTTP client pool usage
- Kafka/SQS consumer lag or queue age

---

## 11. Resilience Patterns

Senior Java backend systems on AWS should use:

- Timeouts on every network call
- Retries with exponential backoff and jitter
- Circuit breakers for unstable dependencies
- Bulkheads for thread pool isolation
- Idempotency keys for retryable writes
- DLQs for failed async processing
- Graceful degradation for non-critical dependencies

Important AWS examples:

- Payment API calls need idempotency keys.
- SQS consumers need duplicate handling.
- S3 event processing can receive duplicate events.
- Lambda retries can invoke the same event more than once.
- ALB retries/client retries can duplicate POST effects unless protected.

---

## 12. Cost Awareness for Java Backends

Watch these cost drivers:

- NAT Gateway data processing
- Cross-AZ data transfer
- CloudWatch log volume
- RDS instance size and storage IOPS
- Aurora read replicas
- DynamoDB read/write capacity
- ElastiCache node size
- ALB LCUs
- EKS cluster and node costs

Optimization examples:

- Avoid noisy debug logs in production.
- Use S3 lifecycle policies.
- Right-size ECS task CPU/memory.
- Use Graviton where compatible.
- Use RDS reserved instances/savings plans for steady workloads.
- Cache expensive reads, but monitor Redis cost.

---

## 13. Senior Backend Design Checklist

- Does the service have explicit timeouts?
- Are retries bounded and using jitter?
- Are write operations idempotent?
- Is database pool size safe across all replicas?
- Are migrations backward compatible?
- Is Redis failure behavior defined?
- Does every async consumer have a DLQ?
- Are SQS visibility timeout and processing time aligned?
- Are IAM permissions least privilege?
- Are secrets stored in Secrets Manager or Parameter Store?
- Are logs structured and searchable?
- Are JVM, DB pool, and dependency metrics visible?
- Is rollback safe if schema changed?
- Is the cost impact understood?
- Is the design multi-AZ by default for production?

