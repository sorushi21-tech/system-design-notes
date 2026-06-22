# HTTP, HTTPS, and TLS

HTTP is the application protocol used by most web systems. HTTPS is HTTP protected by TLS encryption.

## HTTP Request

```http
GET /users/42 HTTP/1.1
Host: api.example.com
Accept: application/json
Authorization: Bearer token
```

A request contains:

- Method.
- Path.
- Headers.
- Optional body.

Common methods:

| Method | Meaning |
|--------|---------|
| GET | Read data |
| POST | Create or submit data |
| PUT | Replace a resource |
| PATCH | Partially update a resource |
| DELETE | Delete a resource |

## HTTP Response

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: max-age=60
```

A response contains:

- Status code.
- Headers.
- Optional body.

## Important Status Codes

| Code | Meaning | Why It Matters |
|------|---------|----------------|
| 200 | OK | Success |
| 201 | Created | Resource was created |
| 204 | No Content | Success without response body |
| 301 | Permanent redirect | Can be cached by clients |
| 302 | Temporary redirect | Temporary route change |
| 400 | Bad request | Invalid client input |
| 401 | Unauthorized | Authentication missing or invalid |
| 403 | Forbidden | Authenticated but not allowed |
| 404 | Not found | Resource or route does not exist |
| 409 | Conflict | Version or idempotency conflict |
| 429 | Too many requests | Rate limited |
| 500 | Internal server error | Application failed |
| 502 | Bad gateway | Proxy got bad upstream response |
| 503 | Service unavailable | Backend unavailable or overloaded |
| 504 | Gateway timeout | Upstream timed out |

## Useful Headers

| Header | Purpose |
|--------|---------|
| `Host` | Target hostname |
| `Authorization` | Credentials |
| `Content-Type` | Request/response body format |
| `Cache-Control` | Browser/CDN caching behavior |
| `ETag` | Resource version for revalidation |
| `If-None-Match` | Ask if cached copy is still valid |
| `Content-Encoding` | Compression such as gzip or br |
| `Retry-After` | When a client should retry |
| `X-Request-ID` | Request correlation |
| `traceparent` | Distributed tracing context |
| `X-Forwarded-For` | Original client IP through proxies |

## HTTP Versions

| Version | Main Feature | Notes |
|---------|--------------|-------|
| HTTP/1.1 | Persistent connections | Simple and widely supported |
| HTTP/2 | Multiplexing over one TCP connection | Better for many parallel requests |
| HTTP/3 | Runs over QUIC/UDP | Better connection migration and less transport head-of-line blocking |

## TLS and HTTPS

TLS provides:

- Encryption.
- Server identity validation.
- Data integrity.

Simplified TLS flow:

1. Client connects to server.
2. Server sends certificate.
3. Client validates certificate chain and hostname.
4. Client and server agree on encryption keys.
5. HTTP traffic flows securely.

TLS can terminate at:

- CDN.
- Load balancer.
- Reverse proxy.
- Application server.

Production rule:

```text
Monitor certificate expiry.
Expired certificates cause immediate user-visible outages.
```

## TLS Termination

TLS termination means encrypted traffic is decrypted at a specific infrastructure layer.

Common patterns:

| Termination Point | Use Case |
|-------------------|----------|
| CDN | Global edge TLS, caching, DDoS absorption |
| Load balancer | Central certificate management for services |
| Reverse proxy | Internal routing and policy enforcement |
| Application | End-to-end encryption to the app process |

Choose termination based on security requirements, operational simplicity, and whether internal traffic must also stay encrypted.
