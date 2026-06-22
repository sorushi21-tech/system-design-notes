# Troubleshooting and System Design

Networking failures are easier to debug when every layer leaves evidence.

## Default Debugging Order

```text
DNS name
  -> resolved IP or target
  -> route/network path
  -> firewall/security rule
  -> port connectivity
  -> TLS handshake
  -> HTTP request and response
  -> application and dependency behavior
```

## Useful Signals

| Signal | What It Helps Diagnose |
|--------|------------------------|
| DNS query result | Wrong record, stale cache, split-horizon DNS |
| Load balancer access logs | Target status, latency, routing, health |
| Application logs | Handler errors, dependency failures, request IDs |
| Distributed traces | Slow spans, cross-service call paths |
| Metrics | Error rate, latency, saturation, connection count |
| VPC Flow Logs | Accept/reject, source/destination IP, port, protocol |
| Packet captures | Retransmits, resets, handshake failures |

## Common Commands

```bash
nslookup api.example.com
dig api.example.com
curl -v https://api.example.com/health
curl --resolve api.example.com:443:203.0.113.10 https://api.example.com/health
telnet api.example.com 443
nc -vz api.example.com 443
traceroute api.example.com
```

Use equivalent tools for your operating system when a command is unavailable.

## Timeout Troubleshooting

When a request times out, check in this order:

1. DNS resolved to the expected target.
2. Route table sends traffic to the right place.
3. Security group and NACL allow the port.
4. Target process is listening.
5. Load balancer target is healthy.
6. TLS handshake completes.
7. Application logs show the request.
8. Downstream dependency latency is acceptable.
9. Retry behavior is not amplifying load.

## `502` vs `504`

| Status | Meaning | Common Causes |
|--------|---------|---------------|
| `502 Bad Gateway` | Proxy/load balancer got an invalid or failed upstream response | Crashed app, protocol mismatch, connection reset, bad target port |
| `504 Gateway Timeout` | Proxy/load balancer waited too long for upstream | Slow app, slow DB, overloaded service, timeout mismatch |

Simple distinction:

```text
502: upstream response was bad or connection failed.
504: upstream response took too long.
```

## System Design Talking Points

In system design interviews and real architecture discussions, networking is usually about safety, availability, latency, and blast radius.

Mention networking when discussing:

- Public entry points through DNS, CDN, and load balancers.
- Private subnets for application services and databases.
- Multi-AZ deployments.
- Health checks and load balancer failover.
- NAT gateways or VPC endpoints for private outbound access.
- Security group rules between tiers.
- Timeouts, retries, and idempotency.
- CDN caching strategy.
- Observability through logs, metrics, traces, and flow logs.

Avoid claiming:

- DNS failover is instant.
- Private subnets alone make systems secure.
- Retries always improve reliability.
- A load balancer fixes slow downstream dependencies.
- Opening a port is enough if routes, firewalls, and protocols are wrong.

Good production shape:

```text
Route 53
  -> CloudFront
  -> public ALB
  -> private app subnets across multiple AZs
  -> private database/cache subnets
  -> VPC endpoints for AWS services
  -> NAT Gateway only where outbound internet is required
```

## Quick Revision

- DNS maps stable names to IPs or target names.
- IP identifies the endpoint; port identifies the process; protocol defines the rules.
- A VPC is a private cloud network.
- Subnets divide a VPC by availability zone, exposure, and responsibility.
- Public subnets hold controlled entry points such as load balancers and NAT gateways.
- Private subnets hold application services, databases, and caches.
- Route tables decide where traffic goes next.
- Security groups and NACLs control allowed traffic.
- NAT lets private resources make outbound internet calls.
- VPC endpoints let private resources access supported AWS services without public internet.
- HTTP is the application protocol; HTTPS is HTTP protected by TLS.
- TCP is reliable and ordered; UDP is lightweight; QUIC powers HTTP/3.
- Load balancers distribute traffic to healthy targets.
- Reverse proxies and API gateways route, authenticate, limit, and shape traffic.
- CDNs reduce latency and origin load for cacheable content.
- Every network call should have timeouts.
- Retries need backoff, jitter, limits, and idempotency.
- Debug from DNS and routing inward before blaming application code.
