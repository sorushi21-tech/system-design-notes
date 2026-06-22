# Runtime Configuration

Production containers should be configured at runtime.

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

## Java Runtime Options

Java 11+ supports container-aware memory and CPU limits. Still configure JVM options explicitly for production.

Recommended options:

```bash
docker run -e JAVA_OPTS='-XX:+UseContainerSupport -XX:MaxRAMPercentage=75 -XX:+ExitOnOutOfMemoryError -XX:ActiveProcessorCount=2 -Djava.security.egd=file:/dev/./urandom -Djava.io.tmpdir=/tmp' my-app:1.0
```

For Spring Boot services, avoid setting `-Xmx` directly unless you know the full non-heap requirements.

Use environment variables to keep the image generic and let deployment platforms tune heap and CPU independently.

## JVM Memory Behavior

Container memory includes more than heap:

- Heap
- Metaspace
- Thread stacks
- Direct buffers
- Code cache
- JNI native allocations

If the JVM is given the full container memory as heap, the process can still be OOM-killed because of native overhead.
