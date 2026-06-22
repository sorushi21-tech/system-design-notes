# Health Checks and Logging

## Health Checks

A health check tells Docker or an orchestrator whether the container is healthy.

Dockerfile example:

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1
```

Good health checks:

- Are fast
- Are reliable
- Do not overload dependencies
- Reflect whether the app can serve traffic

In Kubernetes, health is usually split into:

- Liveness probe: should this container be restarted?
- Readiness probe: should this pod receive traffic?
- Startup probe: has slow startup finished?

For Spring Boot, expose health through Actuator and keep readiness separate from liveness.

When using Docker health checks, avoid probes that exercise slow downstream dependencies. The container may be restarted before the application finishes warmup if the health check is too aggressive.

Use `management.endpoints.web.exposure.include=health,info,metrics` and configure the actuator path explicitly so health probes target the right endpoint.

Example:

```properties
management.endpoints.web.exposure.include=health,info,metrics
management.endpoint.health.probes.enabled=true
management.endpoints.web.base-path=/actuator
```

By separating health and metrics, you avoid startup readiness failures from slow or optional monitoring endpoints.

## Graceful Shutdown

Containers are stopped with signals. A Java service should handle shutdown gracefully:

```text
SIGTERM received
  -> stop accepting new requests
  -> finish in-flight requests
  -> close database/cache clients
  -> exit before grace period expires
```

Spring Boot settings:

```properties
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s
```

In production, align application shutdown time with orchestrator termination grace periods and load balancer deregistration delay.

## Logging

Containers should write logs to stdout and stderr.

Good:

```text
Application
  -> stdout/stderr
  -> container runtime
  -> log collector
  -> searchable log backend
```

For Docker-specific logging behavior, choose a driver that matches your environment and production architecture.

Common drivers:

- `json-file` for local development
- `local` for high-performance local storage
- `journald` for systems using systemd
- `gelf`, `fluentd`, `awslogs` for cloud log collection

Use log rotation options in production if the local driver is in use:

```bash
docker run --log-driver json-file --log-opt max-size=10m --log-opt max-file=3 my-app:1.0
```

Avoid writing important logs only inside container files because containers are temporary.

In production, logs are usually collected by:

- Fluent Bit
- Fluentd
- Logstash
- CloudWatch Logs
- Datadog or similar tools

For Java services, structured JSON logs make filtering easier during incidents.

## Observability Checklist

- Logs include trace ID, request ID, service name, and environment.
- Metrics expose JVM, HTTP, database pool, and business counters.
- Traces show downstream calls and latency.
- Health endpoints are separate from metrics endpoints.
- Startup, shutdown, and OOM events are visible in logs.
