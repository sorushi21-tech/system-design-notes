# Addressing and DNS

Before two systems communicate, the sender must know:

- Which machine or endpoint should receive traffic.
- Which process on that endpoint should receive traffic.
- Which protocol both sides will speak.
- Which stable name should be used instead of a fragile raw address.

## IP Address

An IP address identifies a machine or network endpoint.

```text
api.example.com -> 203.0.113.10
```

Simplified flow:

```text
User enters URL
  -> DNS resolves domain to IP or target
  -> client opens TCP/UDP connection
  -> TLS handshake happens for HTTPS
  -> HTTP request is sent
  -> server sends response
```

## IPv4 and IPv6

IPv4 is the older and still very common IP format.

```text
192.168.1.10
8.8.8.8
10.0.1.25
```

IPv6 is the newer format with a much larger address space.

```text
2001:db8::1
```

IPv4 has limited address space, so private networks, NAT, and load balancers are heavily used.

## Public IP vs Private IP

| Type       | Meaning                                 | Example Use                                        |
|------------|-----------------------------------------|----------------------------------------------------|
| Public IP  | Reachable from the internet             | Internet-facing load balancer, public EC2 instance |
| Private IP | Reachable only inside a private network | ECS task, RDS database, Redis node                 |

Common private IPv4 ranges:

| Range                          | CIDR             |
|--------------------------------|------------------|
| 10.0.0.0 to 10.255.255.255     | `10.0.0.0/8`     |
| 172.16.0.0 to 172.31.255.255   | `172.16.0.0/12`  |
| 192.168.0.0 to 192.168.255.255 | `192.168.0.0/16` |

## CIDR Notation

CIDR notation defines a range of IP addresses.

```text
10.0.0.0/16
```

Meaning:

- Network starts at `10.0.0.0`.
- `/16` means the first 16 bits are fixed.
- The range contains many addresses, such as `10.0.1.10` and `10.0.2.20`.

AWS VPC example:

```text
VPC CIDR:        10.0.0.0/16
Public subnet:  10.0.1.0/24
Private app:    10.0.10.0/24
Private data:   10.0.20.0/24
```

CIDR planning matters because overlapping CIDR ranges create problems for VPC peering, VPNs, Transit Gateway, and hybrid cloud networking.

## Port

A port is a logical number used by the operating system to deliver network traffic to the right process.

```text
10.0.1.25:8080
```

Here, `10.0.1.25` is the IP address and `8080` is the port.

Common ports:

| Port | Common Use |
|------|------------|
| 22 | SSH |
| 53 | DNS |
| 80 | HTTP |
| 443 | HTTPS |
| 3306 | MySQL |
| 5432 | PostgreSQL |
| 6379 | Redis |
| 8080 | Common application port |

Client-side ephemeral port example:

```text
Client 192.168.1.10:51544 -> Server 203.0.113.10:443
```

The server listens on port `443`. The client uses temporary port `51544` so the operating system can match the response to the correct connection.

## Protocol

A protocol defines message format, connection behavior, reliability expectations, and error handling.

| Protocol | Layer | Common Use |
|----------|-------|------------|
| IP | Network | Addressing and routing packets |
| TCP | Transport | Reliable ordered connections |
| UDP | Transport | Lightweight connectionless traffic |
| QUIC | Transport/Application-adjacent | HTTP/3 over UDP |
| DNS | Application | Domain name resolution |
| HTTP | Application | Web/API requests |
| HTTPS | Application | HTTP protected by TLS |
| TLS | Security | Encryption and identity validation |

Important distinction:

- IP answers: which machine or network endpoint?
- Port answers: which process on that endpoint?
- Protocol answers: what rules should both sides follow?

Opening a port is not enough. The client and server must also speak the same protocol.

```text
Correct:   HTTP client -> HTTP server on port 80
Incorrect: HTTP client -> PostgreSQL server on port 5432
```

## DNS

DNS converts a domain name into an IP address or another DNS name.

```text
api.example.com -> 203.0.113.10
```

DNS matters because users and services should depend on stable names, not raw infrastructure addresses.

## DNS Resolution Flow

```text
Browser cache
  -> Operating system cache
  -> Recursive resolver
  -> Root DNS server
  -> TLD DNS server
  -> Authoritative DNS server
  -> IP address or target name
```

Explanation:

- Browser and OS check local cache first.
- Recursive resolver is usually provided by ISP, company network, or a public DNS provider.
- Root server points to the top-level domain server, such as `.com`.
- TLD server points to the authoritative name server.
- Authoritative server returns the final DNS answer.

## Common DNS Records

| Record | Purpose | Example |
|--------|---------|---------|
| A | Hostname to IPv4 | `api.example.com -> 203.0.113.10` |
| AAAA | Hostname to IPv6 | `api.example.com -> 2001:db8::1` |
| CNAME | Alias to another hostname | `www -> example.com` |
| MX | Mail server | Email delivery |
| TXT | Text metadata | SPF, DKIM, verification |
| NS | Name server delegation | Authoritative DNS |

In Route 53, Alias records are commonly used to point a domain to AWS resources such as ALB, CloudFront, or API Gateway.

## TTL

TTL means Time To Live. It controls how long DNS answers can be cached.

| TTL Choice | Benefit | Tradeoff |
|------------|---------|----------|
| Low TTL | Faster changes and failover | More DNS queries |
| High TTL | Better caching and lower query volume | Slower changes and failover |

DNS failover is not instant because resolvers and clients may cache records longer than expected.

## DNS-Based Load Balancing

DNS can return different answers based on policy:

- Round robin.
- Weighted routing.
- Latency-based routing.
- Geographic routing.
- Failover routing.

Limitations:

- DNS does not know if every request succeeded.
- Cached answers reduce control.
- Some clients ignore TTL.
- DNS failover is usually slower than load balancer failover.

## Avoid Hardcoded IPs

Hardcoded IPs are fragile. They break when instances are replaced, ECS tasks restart, EKS pods are rescheduled, RDS fails over, load balancer infrastructure changes, or services move.

Better patterns:

- Use DNS names.
- Use Route 53 records.
- Use ALB/NLB DNS names.
- Use Kubernetes Services in EKS.
- Use ECS Cloud Map service discovery.
- Use RDS/Aurora endpoints.
