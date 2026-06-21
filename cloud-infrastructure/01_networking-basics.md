# Networking Basics

Networking explains how a request travels from a user to a service and how the response comes back. In cloud infrastructure, networking is the foundation for understanding latency, availability, load balancing, CDNs, timeouts, retries, service discovery, and production failures.

## 1. Request Flow From User to Application

A common production request path looks like this:

```text
Browser or mobile app
  -> DNS resolver
  -> CDN or edge network
  -> Load balancer
  -> Reverse proxy or API gateway
  -> Application service
  -> Cache/database/downstream service
  -> Response back to user
```

Each layer can add latency, improve reliability, enforce security, cache data, route traffic, or fail independently.

When debugging a request, ask:

- Did DNS resolve correctly?
- Did the request reach the edge/CDN?
- Did the load balancer find a healthy backend?
- Did TLS terminate correctly?
- Did the proxy route to the correct service?
- Did the application timeout while calling another dependency?
- Did a retry create duplicate work?

![Request Flow Diagram](../images/production_request_flow_polished.png)
---

## 2. Network

A network is a group of devices that can communicate with each other. These devices can be laptops, phones, servers, containers, databases, load balancers, routers, or cloud services.

In backend systems, a network is what allows:

- A browser to call an API.
- One microservice to call another microservice.
- An application to connect to a database.
- A private service to call an external third-party API.
- AWS services inside a VPC to communicate safely.

### Simple Network Example

```text
Laptop
  -> Wi-Fi router
  -> Internet
  -> Load balancer
  -> Backend server
```

Each device must know where to send traffic next. This is handled through IP addresses, routing, DNS, and network devices.

### Local Network

A local network connects devices in a small area.

Examples:

- Home Wi-Fi network
- Office network
- Local Docker network
- Kubernetes pod network

Example:

```text
Laptop: 192.168.1.10
Phone:  192.168.1.11
Router: 192.168.1.1
```

These devices can communicate privately inside the same network.

### Internet

The internet is a global network of networks.

When your browser calls a public API, traffic may pass through:

- Your local router
- ISP network
- Internet backbone
- CDN edge
- Cloud provider network
- Load balancer
- Backend service

This is why latency is affected by geography, routing, congestion, TLS handshakes, and the number of network hops.

### AWS VPC as a Private Network

In AWS, a VPC is your private cloud network.

Example:

```text
VPC: 10.0.0.0/16
  Public subnet:      10.0.1.0/24
  Private app subnet: 10.0.10.0/24
  Private DB subnet:  10.0.20.0/24
```

Common AWS network layout:

```text
Internet
  -> Route 53
  -> CloudFront, optional
  -> Public ALB subnet
  -> Private ECS/EKS service subnet
  -> Private RDS/ElastiCache subnet
```

This keeps backend services and databases away from direct internet exposure.

### Public Network vs Private Network

| Type            | Meaning                     | Example                                      |
|-----------------|-----------------------------|----------------------------------------------|
| Public network  | Reachable from the internet | Public API endpoint, CloudFront, ALB         |
| Private network | Reachable only internally   | ECS service subnet, RDS subnet, Redis subnet |

Production backend rule:

Public traffic should enter through controlled public entry points. Internal services should stay private.

### Subnet

A subnet is a smaller range inside a larger network.

Example:

```text
VPC:    10.0.0.0/16
Subnet: 10.0.1.0/24
Subnet: 10.0.2.0/24
```

Subnets help separate different types of resources:

- Public subnet for load balancers and NAT gateways
- Private app subnet for ECS/EKS/EC2 services
- Private data subnet for RDS and ElastiCache

### Router and Route Table

A router decides where traffic should go next.

In AWS, route tables define routing rules for subnets.

Example:

```text
0.0.0.0/0 -> Internet Gateway
10.0.0.0/16 -> local VPC routing
```

Common AWS route targets:

- Internet Gateway for public internet access
- NAT Gateway for private subnet outbound internet access
- VPC peering connection
- Transit Gateway
- VPC endpoint

### Firewall

A firewall controls allowed and blocked traffic.

In AWS:

- Security Groups are stateful firewalls attached to resources.
- Network ACLs are stateless firewalls attached to subnets.

Example security group rule:

```text
Allow ALB security group -> ECS service on port 8080
Allow ECS service security group -> RDS on port 5432
```

This is better than opening database access to a wide IP range.

### Network Hop

A network hop is one step traffic takes between source and destination.

Example:

```text
Client
  -> router
  -> ISP
  -> CDN
  -> load balancer
  -> backend service
```

More hops can mean:

- More latency
- More places where failure can happen
- More logs needed for debugging

### Network Latency

Network latency is the time taken for data to travel between two points.

It is affected by:

- Physical distance
- Number of hops
- Packet loss
- TLS handshakes
- DNS lookup time
- Load balancer/proxy processing
- Congestion

Backend design impact:

- Put services close to users when latency matters.
- Reuse connections with keep-alive.
- Use CDNs for static/cacheable content.
- Avoid unnecessary synchronous service calls.

### Network Bandwidth

Bandwidth is how much data can be transferred per second.

High bandwidth matters for:

- File uploads/downloads
- Video/image delivery
- Data replication
- Large analytics exports
- Backup and restore

Backend design impact:

- Use S3 pre-signed URLs for large uploads.
- Use CloudFront for downloads.
- Compress responses where useful.
- Avoid moving large data through application servers unnecessarily.

### Network Best Practices for Backend Developers

- Understand whether a service is public or private.
- Do not expose databases directly to the internet.
- Use Security Groups with least privilege.
- Use private subnets for application and data layers.
- Use load balancers as controlled entry points.
- Use DNS/service discovery instead of hardcoded IPs.
- Set timeouts for all network calls.
- Add retries carefully with backoff and jitter.
- Log request IDs so calls can be traced across services.
- Know which AWS route a request takes before debugging production issues.

---

## 3. IP Address

An IP address is the network address of a device or service. If DNS is like a contact name, the IP address is the actual reachable network location.

Example:

```text
api.example.com -> 203.0.113.10
```

Here:

- `api.example.com` is the domain name.
- `203.0.113.10` is the IP address.
- DNS connects the human-friendly name to the network address.

### Why IP Address Matters

Every network request needs a destination IP address.

When a browser calls:

```text
https://api.example.com/orders
```

the system must first discover the IP address behind `api.example.com`. Only then can it open a network connection.

Simplified flow:

```text
User enters URL
  -> DNS resolves domain to IP
  -> Client opens TCP/UDP connection to IP
  -> TLS handshake happens for HTTPS
  -> HTTP request is sent
  -> Server sends response
```

### IPv4

IPv4 is the older and most common IP format.

Example:

```text
192.168.1.10
```

IPv4 has four numbers separated by dots. Each number is from 0 to 255.

Examples:

```text
8.8.8.8
10.0.1.25
172.16.4.10
203.0.113.10
```

IPv4 has limited address space, so private networks, NAT, and load balancers are heavily used.

### IPv6

IPv6 is the newer format with much larger address space.

Example:

```text
2001:db8::1
```

IPv6 uses hexadecimal groups separated by colons.

Why IPv6 exists:

- IPv4 address exhaustion
- More available public addresses
- Better long-term internet scalability

In backend system design, you should know IPv6 exists, but most internal AWS backend discussions still focus heavily on IPv4 and VPC CIDR ranges.

### Public IP Address

A public IP address is reachable from the internet.

Examples of resources that may have public IPs:

- Internet-facing load balancer
- Public EC2 instance
- NAT Gateway Elastic IP
- Public API endpoint

Production rule:

Backend services should usually not be directly exposed through public IPs. Put them behind controlled entry points such as CloudFront, ALB, NLB, or API Gateway.

### Private IP Address

A private IP address is reachable only inside a private network, such as a home network, office network, or AWS VPC.

Common private IPv4 ranges:

| Range                          | CIDR             |
|--------------------------------|------------------|
| 10.0.0.0 to 10.255.255.255     | `10.0.0.0/8`     |
| 172.16.0.0 to 172.31.255.255   | `172.16.0.0/12`  |
| 192.168.0.0 to 192.168.255.255 | `192.168.0.0/16` |

AWS backend systems commonly use private IPs inside a VPC.

Example:

```text
ALB public endpoint
  -> private ECS task IP
  -> private RDS endpoint
```

### Static IP vs Dynamic IP

Static IP:

- Stays the same until intentionally changed.
- Useful for allowlists and fixed endpoints.
- Requires careful ownership and documentation.

Dynamic IP:

- Can change when a resource restarts, scales, or is recreated.
- Common for containers, pods, and cloud-managed infrastructure.
- Should not be hardcoded in application code.

Senior backend rule:

Use stable DNS names and service discovery instead of depending on dynamic IPs.

### AWS Elastic IP

Elastic IP is a static public IPv4 address in AWS.

Use it when:

- A third-party partner requires IP allowlisting.
- NAT Gateway needs stable outbound IP.
- A special legacy endpoint needs a fixed IP.

Avoid using Elastic IP as the default way to expose applications. For application traffic, prefer:

- CloudFront
- API Gateway
- Application Load Balancer
- Network Load Balancer

### CIDR Notation

CIDR notation defines a range of IP addresses.

Example:

```text
10.0.0.0/16
```

Meaning:

- Network starts at `10.0.0.0`.
- `/16` means the first 16 bits are fixed.
- The range can contain many addresses, such as `10.0.1.10`, `10.0.2.20`, etc.

AWS VPC example:

```text
VPC CIDR:        10.0.0.0/16
Public subnet:  10.0.1.0/24
Private app:    10.0.10.0/24
Private data:   10.0.20.0/24
```

CIDR planning matters because overlapping CIDR ranges create problems for VPC peering, VPNs, Transit Gateway, and hybrid networking.

### NAT

NAT means Network Address Translation.

It allows private IP addresses to communicate with the internet through a public IP.

AWS example:

```text
Private ECS task
  -> NAT Gateway
  -> Elastic IP
  -> Internet / third-party API
```

Why NAT is useful:

- Backend services stay private.
- Outbound traffic uses a stable public IP.
- Third-party APIs can allowlist the NAT Gateway Elastic IP.

Important cost note:

NAT Gateway can become expensive because AWS charges for hourly usage and data processing. For AWS services such as S3, DynamoDB, SQS, and Secrets Manager, VPC endpoints can reduce NAT dependency.

### IP Address and Ports

An IP address identifies the machine or network endpoint. A port identifies the application process or service on that endpoint.

Example:

```text
10.0.1.25:8080
```

Here:

- `10.0.1.25` is the IP address.
- `8080` is the port.

Common ports:

| Port | Protocol/Use |
| --- | --- |
| 80 | HTTP |
| 443 | HTTPS |
| 22 | SSH |
| 5432 | PostgreSQL |
| 3306 | MySQL |
| 6379 | Redis |

Security Groups commonly allow traffic by port and source.

Example:

```text
Allow ALB security group -> ECS service port 8080
Allow ECS security group -> RDS port 5432
```

### Why We Avoid Hardcoding IPs

Hardcoded IPs are fragile.

They break when:

- EC2 instance is replaced.
- ECS task is restarted.
- EKS pod is rescheduled.
- RDS fails over.
- Load balancer infrastructure changes.
- Service moves to another subnet or region.

Better patterns:

- Use DNS names.
- Use Route 53 records.
- Use ALB/NLB DNS names.
- Use Kubernetes Services in EKS.
- Use ECS Cloud Map service discovery.
- Use RDS/Aurora endpoints.

### IP Address Best Practices on AWS

- Keep backend services in private subnets.
- Expose public traffic through CloudFront, ALB, NLB, or API Gateway.
- Use Route 53 DNS names instead of raw IPs.
- Use Security Group references instead of broad IP ranges.
- Use Elastic IP only when stable public IP is required.
- Use NAT Gateway for controlled outbound internet from private subnets.
- Use VPC endpoints for private access to AWS services.
- Plan VPC and subnet CIDR ranges carefully.
- Document any third-party IP allowlists.

---

## 4. DNS

DNS converts a domain name into an IP address.

Example:

```text
api.example.com -> 203.0.113.10
```

### Resolution Flow

```text
Browser cache
  -> Operating system cache
  -> Recursive resolver
  -> Root DNS server
  -> TLD DNS server
  -> Authoritative DNS server
  -> IP address
```

Explanation:

- Browser and OS check local cache first.
- Recursive resolver is usually provided by ISP, company network, or public DNS provider.
- Root server points to the top-level domain server, such as `.com`.
- TLD server points to the authoritative name server.
- Authoritative server returns the final DNS record.

### Common DNS Records

| Record | Purpose                | Example                           |
|--------|------------------------|-----------------------------------|
| A      | Hostname to IPv4       | `api.example.com -> 203.0.113.10` |
| AAAA   | Hostname to IPv6       | `api.example.com -> 2001:db8::1`  |
| CNAME  | Alias to another name  | `www -> example.com`              |
| MX     | Mail server            | Email delivery                    |
| TXT    | Text metadata          | SPF, DKIM, domain verification    |
| NS     | Name server delegation | Authoritative DNS                 |

### TTL

TTL means Time To Live. It controls how long DNS answers can be cached.

Low TTL:

- Faster changes
- Faster failover
- More DNS queries

High TTL:

- Better caching
- Lower DNS load
- Slower changes and failover

DNS failover is not instant because resolvers and clients may cache records.

### DNS-Based Load Balancing

DNS can return different IPs based on policy:

- Round robin
- Weighted routing
- Latency-based routing
- Geographic routing
- Failover routing

Limitations:

- DNS does not know if every request succeeded.
- Cached answers reduce control.
- Some clients ignore TTL.
- Failover can be slower than load balancer failover.

---

## 5. HTTP and HTTPS

HTTP is the application protocol used by most web systems. HTTPS is HTTP protected by TLS encryption.

### HTTP Request

```http
GET /users/42 HTTP/1.1
Host: api.example.com
Accept: application/json
Authorization: Bearer token
```

A request contains:

- Method
- Path
- Headers
- Optional body

Common methods:

| Method | Meaning                     |
|--------|-----------------------------|
| GET    | Read data                   |
| POST   | Create or submit data       |
| PUT    | Replace a resource          |
| PATCH  | Partially update a resource |
| DELETE | Delete a resource           |

### HTTP Response

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: max-age=60
```

A response contains:

- Status code
- Headers
- Optional body

### Important Status Codes

| Code | Meaning               | Why It Matters                    |
|------|-----------------------|-----------------------------------|
| 200  | OK                    | Success                           |
| 201  | Created               | Resource was created              |
| 204  | No Content            | Success without body              |
| 301  | Permanent redirect    | Can be cached by clients          |
| 302  | Temporary redirect    | Temporary route change            |
| 400  | Bad request           | Invalid client input              |
| 401  | Unauthorized          | Authentication missing or invalid |
| 403  | Forbidden             | Authenticated but not allowed     |
| 404  | Not found             | Resource does not exist           |
| 409  | Conflict              | Version or idempotency conflict   |
| 429  | Too many requests     | Rate limited                      |
| 500  | Internal server error | Application failed                |
| 502  | Bad gateway           | Proxy got bad upstream response   |
| 503  | Service unavailable   | Backend unavailable or overloaded |
| 504  | Gateway timeout       | Upstream timed out                |

### Useful Headers

| Header             | Purpose                            |
|--------------------|------------------------------------|
| `Cache-Control`    | Browser/CDN caching behavior       |
| `ETag`             | Resource version for revalidation  |
| `If-None-Match`    | Ask if cached copy is still valid  |
| `Content-Encoding` | Compression such as gzip or br     |
| `Authorization`    | Credentials                        |
| `Retry-After`      | When a client should retry         |
| `X-Request-ID`     | Request correlation                |
| `traceparent`      | Distributed tracing context        |
| `X-Forwarded-For`  | Original client IP through proxies |

### HTTP Versions

| Version  | Main Feature                         | Notes                                                                |
|----------|--------------------------------------|----------------------------------------------------------------------|
| HTTP/1.1 | Persistent connections               | Simple, widely supported                                             |
| HTTP/2   | Multiplexing over one TCP connection | Better for many parallel requests                                    |
| HTTP/3   | Runs over QUIC/UDP                   | Better connection migration and less transport head-of-line blocking |

### TLS and HTTPS

TLS provides:

- Encryption
- Server identity validation
- Data integrity

Simplified TLS flow:

1. Client connects to server.
2. Server sends certificate.
3. Client validates certificate chain.
4. Client and server agree on encryption keys.
5. HTTP traffic flows securely.

TLS can terminate at:

- CDN
- Load balancer
- Reverse proxy
- Application server

In production, monitor certificate expiry. Expired certificates cause user-visible outages.

---

## 6. TCP, UDP, and QUIC

### TCP

TCP is reliable, ordered, and connection-oriented.

It provides:

- Three-way handshake
- Retransmission of lost packets
- Ordered delivery
- Flow control
- Congestion control

Used by:

- HTTP/1.1
- HTTP/2
- Databases
- SSH
- Email protocols

### UDP

UDP is connectionless and lightweight.

It does not provide built-in:

- Reliability
- Ordering
- Retransmission
- Connection state

Used by:

- DNS
- Streaming
- Gaming
- VoIP
- WebRTC
- QUIC/HTTP/3

### QUIC

QUIC runs over UDP but adds reliability, encryption, multiplexing, and faster connection setup.

HTTP/3 uses QUIC.

Benefits:

- Faster reconnects
- Better behavior on mobile networks
- Avoids some TCP head-of-line blocking problems

---

## 7. TCP Connection Lifecycle

### Connection Establishment

TCP starts with a three-way handshake:

```text
Client -> SYN -> Server
Client <- SYN-ACK <- Server
Client -> ACK -> Server
```

Only after this can data flow.

### Connection Close

A graceful close uses FIN and ACK packets.

```text
Client -> FIN -> Server
Client <- ACK <- Server
Client <- FIN <- Server
Client -> ACK -> Server
```

A connection can also be closed abruptly with RST.

### TIME_WAIT

TIME_WAIT is a state after connection close.

Purpose:

- Prevent delayed packets from corrupting new connections.
- Allow final ACK retransmission.

Problem:

- Too many short-lived connections can create many TIME_WAIT sockets.
- This can cause port exhaustion.

Fix:

- Use keep-alive.
- Use connection pooling.
- Avoid one connection per request.

### Half-Open Connections

A half-open connection happens when one side thinks the connection still exists but the other side is gone.

Causes:

- Network interruption
- Server crash
- NAT timeout
- Mobile client disconnect

Detection:

- TCP keepalive
- Application heartbeat
- Read/write timeout

---

## 8. Keep-Alive and Connection Pooling

Creating new TCP and TLS connections is expensive. Keep-alive reuses connections.

Benefits:

- Lower latency
- Less CPU usage
- Less network overhead
- Better throughput

### HTTP/1.1 Keep-Alive

HTTP/1.1 keeps connections open by default. Multiple requests can reuse the same connection, but parallelism usually needs multiple connections.

### HTTP/2 Multiplexing

HTTP/2 allows multiple streams over one TCP connection. This reduces the number of connections needed.

### Connection Pool Sizing

Too small:

- Requests wait for a free connection.
- Latency increases.

Too large:

- Too many open sockets.
- Downstream service can be overwhelmed.
- Database connection limits can be exceeded.

Sizing depends on:

- Request rate
- Downstream latency
- Number of app instances
- Downstream connection limits
- Timeout settings

---

## 9. Timeouts and Retries

Timeouts prevent requests from waiting forever.

Set timeouts at every layer:

- Client timeout
- Load balancer timeout
- Reverse proxy timeout
- Application timeout
- Database/downstream timeout

Good timeout ordering:

```text
Client timeout > load balancer timeout > application timeout > dependency timeout
```

Retries help with temporary failures, but bad retries can make outages worse.

Good retry behavior:

- Retry only safe operations or idempotent operations.
- Use exponential backoff.
- Add jitter.
- Set a retry budget.
- Do not retry forever.

Bad retry behavior creates retry storms, where extra traffic overloads an already struggling service.

---

## 10. Proxies and Reverse Proxies

### Forward Proxy

A forward proxy sits near the client.

```text
Client -> Forward proxy -> Internet service
```

Uses:

- Corporate internet access
- Content filtering
- Egress logging
- Client anonymity

### Reverse Proxy

A reverse proxy sits in front of servers.

```text
Client -> Reverse proxy -> Backend service
```

Uses:

- Routing
- Load balancing
- TLS termination
- Compression
- Caching
- Rate limiting
- Access logging

Common tools:

- Nginx
- HAProxy
- Envoy
- Traefik

### API Gateway

An API gateway is a reverse proxy specialized for APIs.

Common responsibilities:

- Authentication
- Authorization
- Rate limiting
- Request validation
- API version routing
- Request/response transformation
- Observability

Avoid putting too much business logic in the gateway. Keep domain logic inside services.

---

## 11. CDN

A CDN caches content close to users.

```text
User -> nearest CDN edge -> origin server
```

Benefits:

- Lower latency
- Lower origin load
- Better handling of traffic spikes
- DDoS absorption
- Static asset acceleration

### Cache-Control

Examples:

```http
Cache-Control: public, max-age=31536000, immutable
Cache-Control: no-cache
Cache-Control: no-store
```

Meanings:

- `public`: shared caches can store it.
- `private`: browser can store it, shared caches should not.
- `max-age`: freshness lifetime.
- `immutable`: content will not change during freshness lifetime.
- `no-cache`: can store but must revalidate before reuse.
- `no-store`: should not store.

### ETag

ETag represents a version of a response.

Flow:

```text
Client has cached ETag "v1"
Client sends If-None-Match: "v1"
Server returns 304 Not Modified if unchanged
```

### Cache Invalidation

Ways to update cached content:

- Short TTL
- CDN purge API
- Versioned filenames
- Cache-busting query parameter

Best practice for static assets:

```text
app.a81f3.js
style.19ab2.css
```

Use long TTLs and change the filename when content changes.

Warning: never cache user-specific private data unless cache keys and access rules are designed very carefully.

---

## 12. Load Balancing

Load balancers distribute traffic across healthy backends.

Algorithms:

- Round robin
- Weighted round robin
- Least connections
- Least response time
- IP hash
- Consistent hashing

### Layer 4 Load Balancing

Works at TCP/UDP level.

Good for:

- Very high throughput
- Low latency
- Non-HTTP protocols

### Layer 7 Load Balancing

Works at HTTP/HTTPS level.

Can route by:

- Hostname
- Path
- Header
- Cookie
- Method

Layer 7 load balancers can also handle redirects, TLS termination, WAF integration, and request logs.

---

## 13. Common Failure Modes

### DNS Failure

Symptoms:

- Hostname does not resolve.
- Some users can connect and others cannot.
- Service works in one region but not another.

### Timeout Mismatch

Symptoms:

- 504 errors
- Duplicate retries
- Backend continues processing after client gave up

### Connection Exhaustion

Symptoms:

- New connections fail
- Many TIME_WAIT sockets
- Database connection limit reached

### Retry Storm

Symptoms:

- Dependency slows down
- Clients retry aggressively
- Load multiplies
- System collapses further

### Bad CDN Rule

Symptoms:

- Stale content served too long
- Private content served to wrong user
- Origin changes do not appear

---

## 14. What If IP Address Changes?

IP addresses are not always stable. In cloud systems, instances, containers, pods, NAT gateways, and load balancers can be recreated. When that happens, the IP address may change unless you intentionally design for a stable endpoint.

The senior backend mindset is: clients and services should depend on stable names, not directly on changing machine IPs.

### Public IP vs Private IP

| Type | Meaning | Example Use |
| --- | --- | --- |
| Public IP | Reachable from the internet | Internet-facing load balancer, public EC2 instance |
| Private IP | Reachable only inside private network/VPC | App service to database, internal service calls |

In AWS, backend services should usually run with private IPs inside private subnets. Public traffic should enter through controlled entry points such as CloudFront, ALB, NLB, or API Gateway.

### Static IP vs Dynamic IP

Dynamic IP:

- Can change when resource restarts or is recreated.
- Common for containers, pods, and many cloud-managed resources.
- Should not be hardcoded in application configuration.

Static IP:

- Stays fixed until released or changed intentionally.
- Useful for allowlists, legacy integrations, or fixed outbound identity.
- Adds operational responsibility because the static IP becomes part of the architecture.

### What If an EC2 Public IP Changes?

If an EC2 instance uses an auto-assigned public IP, the public IP can change after stop/start.

Impact:

- DNS records pointing directly to that IP become wrong.
- External clients cannot connect.
- Partner allowlists may break.

Better options:

- Put EC2 behind an ALB/NLB and point DNS to the load balancer.
- Use an Elastic IP only when a fixed public IP is truly required.
- Avoid exposing backend EC2 instances directly to the internet.

### Elastic IP

Elastic IP is a static public IPv4 address in AWS.

Use when:

- A legacy partner requires a fixed source/destination IP.
- You need a fixed public IP for a NAT Gateway.
- You have a small special-purpose public endpoint.

Avoid using Elastic IP as the default design for application services. Load balancers and DNS are usually better abstractions.

### What If a Container or Pod IP Changes?

Container and pod IPs are temporary.

Impact:

- Direct calls to a pod/container IP break after restart or rescheduling.
- Scaling creates many changing IPs.

Better options:

- Docker Compose: use service/container name on the Docker network.
- Kubernetes/EKS: use Kubernetes Service DNS name.
- ECS: use service discovery, Cloud Map, or route through ALB/NLB.

Example:

```text
Bad:  call http://10.0.12.45:8080
Good: call http://order-service.default.svc.cluster.local
Good: call internal-order-service.example.local
```

### What If Load Balancer IP Changes?

Application Load Balancers do not promise fixed IP addresses. Their underlying IPs can change.

Do not point clients to ALB IPs directly.

Use DNS:

```text
api.example.com -> ALB DNS name
```

In Route 53, use an Alias record to point your domain to the ALB.

For fixed IP requirements:

- Use Network Load Balancer, which can support static IPs per Availability Zone.
- Or place AWS Global Accelerator in front for static anycast IPs.

### What If Database IP Changes?

Managed databases such as RDS/Aurora should be accessed through their DNS endpoints, not raw IPs.

Why:

- Failover can move the writer to another instance.
- Maintenance can replace underlying infrastructure.
- Read replicas may change.

Use:

```text
mydb.cluster-xxxxx.ap-south-1.rds.amazonaws.com
```

Not:

```text
10.0.4.17
```

Also make sure Java connection pools handle failover:

- Set connection max lifetime.
- Set connection timeout.
- Retry safely at transaction boundary.
- Avoid infinite retries.

### What If NAT Gateway IP Changes?

NAT Gateway uses an Elastic IP for outbound internet traffic from private subnets.

This matters when:

- A third-party API allowlists your outbound IP.
- Your private services need stable egress identity.

Pattern:

```text
Private ECS/EKS/EC2 service
  -> NAT Gateway
  -> Elastic IP
  -> Third-party API allowlist
```

For high availability, use one NAT Gateway per Availability Zone and allowlist all relevant Elastic IPs.

### DNS and IP Change

DNS hides IP changes from clients when used correctly.

Good pattern:

```text
Client -> api.example.com -> Route 53 -> ALB -> backend service
```

Bad pattern:

```text
Client -> hardcoded backend IP
```

Important:

- DNS TTL controls how long clients/resolvers cache the old answer.
- Low TTL helps faster changes but increases DNS query volume.
- Some clients cache longer than expected.
- DNS is not instant failover.

### What If IP Is Blocked or Allowlists Are Required?

Some third-party systems allow traffic only from approved IPs.

AWS options:

- NAT Gateway with Elastic IP for outbound calls from private subnets.
- Network Load Balancer with static IPs for inbound traffic.
- AWS Global Accelerator for static global entry IPs.
- PrivateLink for private service-to-service connectivity when supported.

Senior backend warning:

IP allowlisting couples your system to network infrastructure. Document it clearly because changing NAT, load balancer, or region can break integrations.

### Best Practices

- Do not hardcode IP addresses in application code.
- Use DNS names for services and databases.
- Put public services behind ALB, NLB, CloudFront, or API Gateway.
- Keep backend services in private subnets.
- Use Security Group references where possible instead of fixed IP ranges.
- Use Elastic IP only when a stable public IP is required.
- Use Route 53 Alias records for AWS load balancers.
- Plan for DNS TTL during migrations and failovers.
- Document any third-party IP allowlists.

---

## 15. Production Checklist

- DNS records are documented.
- TTL values match failover needs.
- HTTPS is enabled for user traffic.
- TLS certificate expiry is monitored.
- Load balancer health checks use readiness semantics.
- Connection pooling is configured.
- Timeouts are explicit at every layer.
- Retries use backoff and jitter.
- Write APIs are idempotent when retried.
- CDN cache rules are reviewed for private data risk.
- Request IDs and trace headers are propagated.
- Edge, proxy, and application logs are available.

---

## 16. AWS Networking Mapping

Use this mapping to connect networking concepts to AWS services.

| Networking Concept | AWS Service |
| --- | --- |
| DNS | Route 53 |
| CDN | CloudFront |
| L7 HTTP/HTTPS load balancing | Application Load Balancer |
| L4 TCP/UDP load balancing | Network Load Balancer |
| API edge/gateway | API Gateway |
| Private network | VPC |
| Network segments | Public/private subnets |
| Firewall at resource level | Security Groups |
| Stateless subnet firewall | Network ACLs |
| Outbound internet from private subnet | NAT Gateway |
| Private AWS service access | VPC endpoints / PrivateLink |
| Network logs | VPC Flow Logs |

Typical AWS public API path:

```text
User
  -> Route 53
  -> CloudFront, optional
  -> ALB or API Gateway
  -> ECS/EKS/EC2/Lambda
  -> RDS/DynamoDB/ElastiCache/SQS/S3
```

Senior backend checklist on AWS:

- Use Route 53 health checks or failover routing only when DNS-level failover is acceptable.
- Put internet-facing ALBs in public subnets.
- Put backend services in private subnets.
- Use Security Group references instead of broad IP ranges.
- Use VPC endpoints for S3, DynamoDB, SQS, Secrets Manager, and other AWS services when private traffic and NAT cost matter.
- Enable ALB access logs and VPC Flow Logs for debugging production traffic.
