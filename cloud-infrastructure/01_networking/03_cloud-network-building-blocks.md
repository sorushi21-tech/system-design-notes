# Cloud Network Building Blocks

A cloud network connects resources such as load balancers, containers, virtual machines, databases, caches, NAT gateways, and managed services.

Backend systems use networks so that:

- A browser can call an API.
- One service can call another service.
- An application can connect to a database.
- A private service can call an external API.
- Cloud services inside a VPC can communicate safely.

## Local Network and Internet

A local network connects nearby devices.

```text
Laptop: 192.168.1.10
Phone:  192.168.1.11
Router: 192.168.1.1
```

The internet is a global network of networks. A public API request may pass through a home router, ISP, internet backbone, CDN edge, cloud provider network, load balancer, and backend service.

## VPC

In AWS, a VPC is your private cloud network.

```text
VPC: 10.0.0.0/16
  Public subnet:      10.0.1.0/24
  Private app subnet: 10.0.10.0/24
  Private DB subnet:  10.0.20.0/24
```

Typical AWS layout:

```text
Internet
  -> Route 53
  -> CloudFront, optional
  -> public ALB subnet
  -> private ECS/EKS service subnet
  -> private RDS/ElastiCache subnet
```

Production rule:

```text
Public traffic enters through controlled public entry points.
Internal services and databases stay private.
```

## Public vs Private Networks

| Type            | Meaning                     | Example                                              |
|-----------------|-----------------------------|------------------------------------------------------|
| Public network  | Reachable from the internet | CloudFront, public API endpoint, internet-facing ALB |
| Private network | Reachable only internally   | ECS service subnet, RDS subnet, Redis subnet         |

Good pattern:

```text
Public ALB endpoint
  -> private application tasks
  -> private database/cache endpoints
```

Bad pattern:

```text
Internet
  -> public database endpoint
```

## Subnet

A subnet is a smaller IP range inside a larger network.

```text
VPC:    10.0.0.0/16
Subnet: 10.0.1.0/24
Subnet: 10.0.2.0/24
```

Subnets separate resources by responsibility and exposure:

- Public subnet for load balancers and NAT gateways.
- Private app subnet for ECS, EKS, EC2, or Lambda VPC access.
- Private data subnet for RDS and ElastiCache.

In AWS, a subnet belongs to one Availability Zone. For high availability, place application resources across multiple subnets in multiple Availability Zones.

## Route Table

A route table decides where traffic should go next.

```text
10.0.0.0/16 -> local VPC routing
0.0.0.0/0  -> Internet Gateway
```

Common AWS route targets:

| Target           | Purpose                                       |
|------------------|-----------------------------------------------|
| Internet Gateway | Public internet access for public subnets     |
| NAT Gateway      | Outbound internet access for private subnets  |
| VPC peering      | Private VPC-to-VPC communication              |
| Transit Gateway  | Hub-and-spoke networking across many networks |
| VPC endpoint     | Private access to supported AWS services      |

## Security Groups and NACLs

In AWS:

- Security Groups are stateful firewalls attached to resources.
- Network ACLs are stateless firewalls attached to subnets.

Example security group rules:

```text
Allow ALB security group -> ECS service on port 8080
Allow ECS service security group -> RDS on port 5432
```

Best practices:

- Allow the smallest required port range.
- Prefer security group references over raw IP ranges.
- Do not expose databases directly to the internet.
- Separate public entry rules from internal service rules.
- Document third-party allowlists.

## NAT

NAT means Network Address Translation. It allows private resources to make outbound internet calls through a public IP.

```text
Private ECS task
  -> NAT Gateway
  -> Elastic IP
  -> Internet / third-party API
```

NAT is useful because:

- Backend services stay private.
- Outbound traffic can use a stable public IP.
- Third-party APIs can allowlist the NAT Gateway Elastic IP.

NAT Gateway can become expensive because AWS charges for hourly usage and data processing. For AWS services such as S3, DynamoDB, SQS, Secrets Manager, and CloudWatch Logs, VPC endpoints can reduce NAT dependency.

## VPC Endpoints

A VPC endpoint lets private resources reach supported AWS services without going through the public internet.

Common uses:

- S3 access from private subnets.
- DynamoDB access from private subnets.
- Pulling secrets from Secrets Manager.
- Sending logs to CloudWatch.
- Calling SQS or SNS privately.

VPC endpoints improve security and can reduce NAT data processing cost.

## Hops, Latency, and Bandwidth

A network hop is one step traffic takes between source and destination.

```text
Client
  -> router
  -> ISP
  -> CDN
  -> load balancer
  -> backend service
```

Latency is the time it takes data to travel between two points. It is affected by physical distance, number of hops, packet loss, DNS lookup time, TCP handshake, TLS handshake, proxy processing, and congestion.

Bandwidth is how much data can be transferred per second. It matters for uploads, downloads, media delivery, replication, backups, and analytics exports.

Backend design impact:

- Put services close to users when latency matters.
- Reuse connections with keep-alive.
- Use CDNs for static and cacheable content.
- Avoid unnecessary synchronous service calls.
- Use S3 pre-signed URLs for large uploads.
- Avoid moving large files through application servers unnecessarily.
