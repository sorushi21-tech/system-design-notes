# AWS Services, Storage, and Databases

AWS gives backend teams managed building blocks for running systems without owning physical data centers. The goal is not to memorize every AWS service name. The goal is to understand which AWS service fits which backend problem.

## 1. AWS Mental Model

AWS provides APIs for infrastructure.

Instead of buying servers, disks, routers, and data center space, you create resources such as:

- Virtual machines
- Containers
- Serverless functions
- Virtual networks
- Load balancers
- Object storage buckets
- Managed databases
- IAM roles
- Monitoring dashboards

AWS design is about deciding:

- What should be managed by the provider?
- What should your team operate directly?
- How much control do you need?
- How much operational burden can you accept?
- What availability, latency, compliance, and cost requirements exist?

---

## 2. Shared Responsibility Model

Cloud security and reliability are shared between provider and customer.

Provider usually manages:

- Data center buildings
- Physical servers
- Physical networking
- Power and cooling
- Managed service infrastructure
- Hardware replacement

Customer usually manages:

- Application code
- Data and access policies
- Identity and permissions
- Network exposure
- Secrets
- Runtime configuration
- Backups and restore requirements
- OS patches for self-managed VMs

Managed services reduce operational work, but they do not remove architecture responsibility.

---

## 3. Main Cloud Service Categories

| Category | What It Solves | AWS Examples |
| --- | --- | --- |
| Compute | Run code | EC2, ECS, EKS, Lambda |
| Networking | Connect systems | VPC, subnets, route tables, ALB, NLB, Route 53 |
| Storage | Store files/objects/disks | S3, EBS, EFS, FSx |
| Databases | Store structured/queryable data | RDS, Aurora, DynamoDB, ElastiCache, Redshift |
| Messaging | Decouple services | SQS, SNS, EventBridge, MSK |
| Identity | Control access | IAM users, roles, policies, Identity Center |
| Security | Protect systems | KMS, Secrets Manager, WAF, Shield, Security Groups |
| Observability | Understand systems | CloudWatch, X-Ray, CloudTrail, VPC Flow Logs |
| DevOps | Deploy systems | CodePipeline, CodeBuild, CodeDeploy, ECR, CloudFormation |

---

## 4. Compute Options

### Virtual Machines

On AWS, virtual machines are EC2 instances. They give maximum control.

Use when:

- You need OS-level access.
- You run legacy software.
- You need custom agents or networking.
- Managed platforms are too restrictive.

Trade-offs:

- You manage patching.
- You manage runtime installation.
- You manage scaling and recovery patterns.

### Containers

Containers package applications consistently.

Run containers using:

- ECS on Fargate
- ECS on EC2
- EKS on managed node groups
- EKS on Fargate

Use when:

- You have microservices.
- You want portable deployment artifacts.
- You need repeatable CI/CD.

### Serverless

Serverless runs code without managing servers.

AWS examples:

- AWS Lambda
- Lambda with SQS, EventBridge, or S3 triggers
- Lambda behind API Gateway

Use when:

- Traffic is event-driven.
- Workloads are bursty.
- You want scale-to-zero.
- You want minimal infrastructure management.

Watch for:

- Cold starts
- Timeout limits
- AWS service limits
- Observability complexity

---

## 5. AWS Fundamentals for Backend Systems

Important AWS services:

| Category | Services |
| --- | --- |
| Compute | EC2, ECS, EKS, Lambda |
| Storage | S3, EBS, EFS |
| Database | RDS, Aurora, DynamoDB, ElastiCache, Redshift |
| Networking | VPC, ALB, NLB, Route 53, CloudFront |
| Security | IAM, KMS, Secrets Manager, WAF |
| Observability | CloudWatch, CloudTrail, X-Ray |
| Messaging | SQS, SNS, EventBridge |
| Containers | ECS, EKS, ECR |

### VPC

A VPC is a private network in AWS.

Core parts:

- Subnets
- Route tables
- Internet Gateway
- NAT Gateway
- Security Groups
- Network ACLs

Common layout:

```text
Public subnet: load balancers, NAT gateways
Private app subnet: application services
Private data subnet: databases and internal data services
```

### IAM

IAM controls access.

Important objects:

- User
- Group
- Role
- Policy

Best practices:

- Prefer roles over long-lived access keys.
- Use least privilege.
- Enable MFA for human access.
- Separate production and non-production accounts.

### S3

S3 is object storage.

Good for:

- Images
- Videos
- Backups
- Logs
- Data lake files
- Static assets

Key features:

- Versioning
- Lifecycle policies
- Server-side encryption
- Bucket policies
- Event notifications
- Pre-signed URLs

S3 is not a normal filesystem. Design around object keys.

### RDS and Aurora

Managed relational databases.

Provider handles:

- Patching
- Backups
- Failover support
- Monitoring integration
- Read replicas

Your team still owns:

- Schema design
- Indexes
- Query performance
- Connection pooling
- Migration safety

---

## 6. AWS Networking for Backend Systems

### VPC Design

A production VPC usually spans multiple Availability Zones.

Typical layout:

```text
VPC
  Public subnet AZ-a: ALB, NAT Gateway
  Public subnet AZ-b: ALB, NAT Gateway
  Private app subnet AZ-a: ECS/EKS/EC2 services
  Private app subnet AZ-b: ECS/EKS/EC2 services
  Private data subnet AZ-a: RDS/ElastiCache
  Private data subnet AZ-b: RDS/ElastiCache
```

Rules:

- Internet-facing ALB belongs in public subnets.
- Backend services belong in private subnets.
- Databases and caches belong in private data subnets.
- Private subnets use NAT Gateway for outbound internet access.
- Security Groups should allow only required traffic.

### Security Groups

Security Groups are stateful firewalls attached to resources.

Example:

```text
ALB security group allows 443 from internet.
Service security group allows app port only from ALB security group.
Database security group allows 5432/3306 only from service security group.
```

This is better than allowing broad CIDR ranges.

### Route 53

Route 53 provides DNS.

Use for:

- Public DNS records
- Private hosted zones
- Health-check based routing
- Weighted routing
- Latency-based routing
- Failover routing

### CloudFront

CloudFront is AWS CDN.

Use for:

- Static assets from S3
- Public images/files
- Cached API responses when safe
- Edge TLS termination
- DDoS absorption with Shield/WAF

Be careful with authenticated or user-specific responses. A wrong cache key can leak private data.

---

## 7. AWS Compute Choices for Java Backends

### ECS Fargate

Good default for Spring Boot microservices when your team wants containers without Kubernetes.

Pros:

- No node management
- Works well with ALB
- Simple service autoscaling
- Good AWS integration
- Lower platform complexity than EKS

Cons:

- Less portable than Kubernetes
- Task startup can be slower during scale-out
- Advanced networking/service mesh patterns are more limited

### EKS

Use when Kubernetes is a platform requirement.

Use when:

- You need Kubernetes APIs.
- You use Helm, Argo CD, service mesh, or operators.
- You have many services and platform engineering support.
- You need portability or advanced orchestration.

Cons:

- More operational complexity
- Cluster upgrades
- Add-on management
- IAM and networking complexity

### Lambda

Use for event-driven Java workloads where cold start and timeout limits are acceptable.

Good for:

- SQS consumers
- S3 event processing
- EventBridge scheduled jobs
- Lightweight webhooks

Watch for:

- Java cold starts
- Database connection handling
- 15-minute max execution time
- VPC networking setup
- At-least-once event delivery

### EC2

Use EC2 when you need full control or support legacy Java deployments.

For new stateless Java services, ECS Fargate is often the simpler AWS-native choice.

---

## 8. Storage Types

### Object Storage

Examples:

- AWS S3

Best for:

- Images
- Videos
- Backups
- Logs
- Static files
- Data lake files

Characteristics:

- Accessed by object key
- Highly durable
- Cheap at scale
- Not a mounted disk by default
- Great for large unstructured data

Common pattern:

```text
Client -> pre-signed URL -> object storage
```

This avoids sending large uploads through application servers.

### Block Storage

Examples:

- AWS EBS

Best for:

- VM disks
- Database disks
- Low-latency random I/O

Characteristics:

- Attached to compute
- Usually zone-scoped
- Behaves like a disk

### File Storage

Examples:

- AWS EFS
- Amazon FSx

Best for:

- Shared filesystem needs
- Legacy applications
- Media processing workflows
- Tools requiring POSIX-like access

Characteristics:

- Mounted by multiple clients
- NFS/SMB-like behavior
- Easier migration for old apps

---

## 9. Database Categories

### Managed Relational Database

Examples:

- RDS
- Aurora

Use when:

- Data is relational.
- Transactions matter.
- SQL queries are needed.
- Strong consistency is important.

Watch for:

- Slow queries
- Lock contention
- Connection limits
- Replica lag
- Migration risk

### NoSQL Database

Examples:

- DynamoDB

Use when:

- Access patterns are known.
- High scale key-value/document access is needed.
- Flexible schema is useful.
- Horizontal scale matters more than relational joins.

Watch for:

- Partition key design
- Hot partitions
- Query limitations
- Secondary index cost
- Consistency model

### Cache

Examples:

- Redis
- Memcached
- ElastiCache

Use for:

- Hot data
- Sessions
- Rate limiting
- Leaderboards
- Temporary computed results

Watch for:

- Eviction policy
- Memory pressure
- Cache stampede
- Stale data
- Failover behavior

### Data Warehouse

Examples:

- Redshift
- Athena
- OpenSearch for search/analytics use cases

Use for:

- Analytics
- Reporting
- Large scans
- Aggregations
- Business intelligence

Do not use a warehouse as the main transactional database.

---

## 10. AWS Messaging and Event Services

### SQS

SQS is a managed queue.

Use for:

- Background jobs
- Decoupling services
- Traffic buffering
- Retryable async processing

Important concepts:

- Visibility timeout
- Dead-letter queue
- Message retention
- Long polling
- Standard vs FIFO queue

For Java consumers, always make processing idempotent because messages can be delivered more than once.

### SNS

SNS is pub/sub fanout.

Use when one event should notify multiple subscribers.

Example:

```text
OrderCreated
  -> inventory subscriber
  -> email subscriber
  -> analytics subscriber
```

### EventBridge

EventBridge routes events using rules.

Use for:

- Domain events
- Scheduled events
- Cross-account event routing
- SaaS integration

EventBridge is useful when you want loose coupling and rule-based routing without direct service-to-service calls.

### MSK

MSK is managed Apache Kafka.

Use when:

- You need Kafka semantics.
- You need high-throughput event streaming.
- You need replayable logs.
- Multiple consumers process the same event stream independently.

SQS is simpler for queues. MSK/Kafka is more powerful but operationally heavier.

---

## 11. Single-AZ, Multi-AZ, and Multi-Region

### Single-AZ

Simplest and cheapest.

Risk:

- Availability Zone failure can take service down.

Use for:

- Development
- Testing
- Non-critical workloads

### Multi-AZ

Runs across multiple availability zones in one region.

Benefits:

- Survives one zone failure.
- Lower latency than multi-region.
- Common production baseline.

### Multi-Region

Runs across multiple geographic regions.

Benefits:

- Regional disaster recovery
- Lower latency for global users
- Higher availability target

Costs:

- More expensive
- More operational complexity
- Harder data consistency
- Harder deployments

---

## 12. Active-Passive vs Active-Active

### Active-Passive

One region serves traffic. Another waits as standby.

Pros:

- Simpler
- Cheaper than active-active
- Easier conflict handling

Cons:

- Failover takes time
- Standby may be under-tested

### Active-Active

Multiple regions serve traffic simultaneously.

Pros:

- Lower user latency
- Better regional resilience
- More available capacity

Cons:

- Complex data consistency
- More expensive
- Harder operations

---

## 13. RPO and RTO

RPO means Recovery Point Objective.

It answers: how much data can we lose?

Example:

```text
RPO = 5 minutes
```

Losing up to 5 minutes of data is acceptable.

RTO means Recovery Time Objective.

It answers: how long can recovery take?

Example:

```text
RTO = 30 minutes
```

The service should recover within 30 minutes.

---

## 14. Choosing an AWS Service

Ask:

- Is the workload stateless or stateful?
- Does it need low latency?
- Does it need strong consistency?
- What is the expected traffic pattern?
- What operational skills does the team have?
- What compliance requirements exist?
- What are the cost constraints?
- Is AWS service lock-in acceptable?

Prefer managed services when:

- The service is standard infrastructure.
- Operating it yourself is risky.
- The AWS managed service is mature.
- Your team gains more by focusing on product code.

Self-manage only when:

- Managed service does not meet requirements.
- You need deep customization.
- Cost at scale justifies operational burden.
- You have the expertise to operate it safely.

---

## 15. Production Checklist

- Network boundaries are defined.
- Public and private subnets are separated.
- IAM follows least privilege.
- Secrets are stored in a secret manager.
- Data is encrypted at rest and in transit.
- Backups are enabled.
- Restore process is tested.
- Monitoring and logging are enabled.
- Cost tags or labels are applied.
- Autoscaling policies are reviewed.
- Disaster recovery plan is documented.
- Database connection limits are understood.
- Managed services are preferred where they reduce risk.
