# Production Traffic Management

Production systems rarely expose application servers directly. Traffic is usually controlled through load balancers, proxies, gateways, and CDNs.

## Load Balancer

A load balancer distributes traffic across healthy targets.

```text
Client
  -> Load balancer
  -> Service instance A
  -> Service instance B
  -> Service instance C
```

Common responsibilities:

- Health checks.
- TLS termination.
- Path or host-based routing.
- Cross-zone distribution.
- Connection handling.
- Access logging.

AWS examples:

| Load Balancer | Best For |
|---------------|----------|
| ALB | HTTP/HTTPS APIs, path routing, host routing |
| NLB | TCP/UDP traffic, static IPs, high throughput, low latency |
| Gateway Load Balancer | Network appliances and inspection |

## Reverse Proxy

A reverse proxy sits in front of application services.

Common examples:

- Nginx.
- Envoy.
- HAProxy.
- Kubernetes Ingress controller.

Common responsibilities:

- Route requests to services.
- Terminate TLS.
- Apply timeouts.
- Add or preserve request IDs.
- Rate limit.
- Compress responses.
- Retry carefully.

## API Gateway

An API gateway is a managed or centralized entry point for APIs.

It commonly handles:

- Authentication and authorization.
- Rate limiting and quotas.
- Request validation.
- Routing to backend services.
- API versioning.
- Usage plans and API keys.

API gateways are useful at public boundaries. Avoid putting too much business logic in the gateway because it can become a hard-to-change bottleneck.

## CDN

A CDN serves content from edge locations near users.

Good uses:

- Static assets.
- Images and videos.
- Public cacheable API responses.
- Global TLS termination.
- DDoS absorption at the edge.

CDN risks:

- Stale content.
- Wrong cache key.
- Accidentally caching private data.
- Origin overload when cache expires or is bypassed.

## Timeouts

Every network call should have timeouts.

Useful timeout types:

- Connect timeout: time allowed to establish a connection.
- Read/request timeout: time allowed to receive a response.
- Idle timeout: time a connection may sit unused.
- Overall deadline: maximum time for the whole operation.

Timeouts should match the caller's latency budget. If an upstream gives up after 2 seconds but a downstream keeps working for 30 seconds, the system wastes capacity.

## Retries

Retries can improve reliability for temporary failures, but they can also multiply load.

Good retry behavior:

- Retry only safe or idempotent operations.
- Use exponential backoff.
- Add jitter.
- Respect `Retry-After`.
- Set a maximum retry budget.
- Use idempotency keys for create/payment/order operations.

Bad retry behavior:

```text
Service is slow
  -> clients retry aggressively
  -> service receives more traffic
  -> service gets slower
  -> more clients retry
```

## Health Checks

Health checks tell load balancers and orchestrators whether a target should receive traffic.

Good health checks:

- Are fast.
- Are reliable.
- Do not overload dependencies.
- Separate "process is alive" from "ready to serve traffic".

Kubernetes usually separates:

- Liveness probe: should this container be restarted?
- Readiness probe: should this pod receive traffic?
- Startup probe: has slow startup finished?
