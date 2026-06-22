# Runtime Configuration, Health, and Logging

Production containers should be configured at runtime, checked by health probes, and observable through stdout/stderr logs and metrics.

## Environment Variables

Containers should receive environment-specific configuration at runtime.

Example:

```bash
docker run -e SPRING_PROFILES_ACTIVE=prod my-app:1.0
```

Good configuration examples:

- Environment name
- Log level
- Database URL
- Cache URL
- Feature flags
- JVM memory options

Do not bake environment-specific secrets into Docker images.

Bad:

```dockerfile
ENV DB_PASSWORD=secret
```

Better:

- Use Docker secrets for local or special setups.
- Use Kubernetes Secrets or external secret managers in production.
- On AWS, prefer AWS Secrets Manager or SSM Parameter Store.

## Resource Limits

Containers should have CPU and memory limits.

Example:

```bash
docker run --memory=512m --cpus=1 my-app:1.0
```

Why limits matter:

- Prevent one container from consuming the whole host.
- Make capacity planning predictable.
- Reveal memory leaks earlier.
- Match production behavior more closely.

## JVM Memory in Containers

Modern JVMs detect container memory limits, but you should still tune intentionally.

Example:

```bash
docker run --memory=1g -e JAVA_OPTS="-XX:MaxRAMPercentage=75 -XX:+ExitOnOutOfMemoryError" my-app:1.0
```

Remember that container memory is not only Java heap. It also includes:

- Metaspace
- Thread stacks
- Direct buffers
- Code cache
- Native libraries
- JVM overhead

For Spring Boot services, avoid setting heap equal to the whole container limit.

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
