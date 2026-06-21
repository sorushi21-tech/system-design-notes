# Docker Fundamentals

Docker is a container platform used to package an application with everything it needs to run: application code, runtime, libraries, environment variables, and operating-system level dependencies.

The main idea is simple: build the application once as an image, then run the same image consistently on a laptop, CI server, test environment, or cloud platform.

## 1. Why Docker Exists

Before containers, teams often had problems like:

- Application works on one machine but fails on another.
- Different projects need different Java, Node, Python, or system library versions.
- Deployments require long manual setup instructions.
- Production servers drift from each other over time.
- Scaling requires configuring many machines by hand.

Docker solves these problems by packaging the application and its runtime dependencies into a standard unit called a container.

Docker helps with:

- Environment consistency
- Faster local setup
- Reproducible builds
- Easier CI/CD
- Isolated services
- Simpler scaling
- Cloud-native deployment

---

## 2. Image vs Container

### Image

An image is a read-only template.

It contains:

- Application code
- Runtime, such as JDK or Node.js
- System libraries
- Default command
- Filesystem layers
- Metadata

Think of an image as a packaged blueprint.

Example:

```bash
docker build -t payment-service:1.0 .
```

This builds an image named `payment-service` with tag `1.0`.

### Container

A container is a running instance of an image.

Example:

```bash
docker run payment-service:1.0
```

If an image is like a class, a container is like an object created from that class.

Multiple containers can run from the same image:

```text
payment-service:1.0 image
  -> container A
  -> container B
  -> container C
```

---

## 3. Containers vs Virtual Machines

| Area      | Virtual Machine                   | Container                 |
|-----------|-----------------------------------|---------------------------|
| Isolation | Full guest OS                     | Process-level isolation   |
| Startup   | Slower                            | Faster                    |
| Size      | Larger                            | Smaller                   |
| Kernel    | Separate guest kernel             | Shares host kernel        |
| Use case  | Strong isolation, full OS control | App packaging and scaling |

Containers are lighter because they do not run a full guest operating system. They share the host kernel but isolate processes, filesystems, networks, and resources.

---

## 4. Docker Architecture

Docker uses a client-server architecture.

```text
Docker CLI
  -> Docker daemon
  -> container runtime
  -> containers
```

### Docker Client

The command-line tool you use:

```bash
docker build
docker run
docker ps
```

### Docker Daemon

The background service that manages:

- Images
- Containers
- Networks
- Volumes
- Builds

### Registry

A registry stores Docker images.

Examples:

- Docker Hub
- Amazon ECR
- GitHub Container Registry

---

## 5. Dockerfile

A Dockerfile defines how to build an image.

Simple example:

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY target/app.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Explanation:

- `FROM`: base image
- `WORKDIR`: working directory inside image
- `COPY`: copy files into image
- `EXPOSE`: document expected port
- `ENTRYPOINT`: command that starts the container

---

## 6. Image Layers

Each Dockerfile instruction creates a layer.

Example:

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY target/app.jar app.jar
```

Docker caches layers. If a layer does not change, Docker reuses it during the next build.

Good layer design makes builds faster.

Bad pattern:

```dockerfile
COPY . .
RUN mvn package
```

Any code change invalidates dependency download and rebuilds too much.

Better pattern:

```dockerfile
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package
```

Dependencies are cached separately from source code.

---

## 7. Multi-Stage Builds

Multi-stage builds keep build tools out of the final image.

Example for Java:

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/target/app.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Benefits:

- Smaller runtime image
- Fewer security vulnerabilities
- Faster image pulls
- Cleaner production artifact

---

## 8. Container Filesystem

A container has a writable layer on top of the image.

Important rule:

Containers are disposable. Do not store important long-term data only inside a container filesystem.

If the container is removed, its writable layer is removed too.

Use external storage for durable data:

- Docker volumes
- Bind mounts
- Object storage
- Databases
- Managed file storage

---

## 9. Volumes

Volumes persist data outside the container writable layer.

Create volume:

```bash
docker volume create app-data
```

Use volume:

```bash
docker run -v app-data:/data my-app:1.0
```

Types of storage:

| Type         | Meaning                          | Use Case                             |
|--------------|----------------------------------|--------------------------------------|
| Named volume | Managed by Docker                | Local databases, persistent app data |
| Bind mount   | Host path mounted into container | Local development                    |
| tmpfs        | In-memory mount                  | Temporary sensitive data             |

In production Kubernetes or cloud environments, persistent data usually lives in managed databases, persistent volumes, object storage, or file storage.

---

## 10. Docker Networking

Containers need networking to talk to users, other containers, and external services.

Common Docker network types:

| Network | Meaning                         | Use Case                                |
|---------|---------------------------------|-----------------------------------------|
| bridge  | Default local container network | Local development                       |
| host    | Container shares host network   | Performance or special networking cases |
| overlay | Multi-host network              | Swarm or orchestrated environments      |
| none    | No network                      | Isolated workloads                      |

### Port Mapping

Container ports are internal unless published.

```bash
docker run -p 8080:8080 my-app:1.0
```

Meaning:

```text
host port 8080 -> container port 8080
```

### Container-to-Container DNS

On a user-defined bridge network, containers can reach each other by name.

```bash
docker network create app-network
docker run -d --name redis --network app-network redis
docker run -d --name api --network app-network my-api:1.0
```

The `api` container can reach Redis using hostname `redis`.

---

## 11. Environment Variables and Configuration

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

Do not bake environment-specific secrets into Docker images.

Bad:

```dockerfile
ENV DB_PASSWORD=secret
```

Better:

- Use Docker secrets for local/special setups.
- Use Kubernetes Secrets or external secret managers in production.
- On AWS, prefer AWS Secrets Manager or SSM Parameter Store.

---

## 12. Resource Limits

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

For Java applications, make sure the JVM understands container limits. Modern JVMs do this well, but still configure heap intentionally for production.

---

## 13. Health Checks

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

---

## 14. Docker Compose

Docker Compose runs multiple containers from one YAML file.

Example:

```yaml
services:
  api:
    build: .
    ports:
      - "8080:8080"
    environment:
      REDIS_HOST: redis
    depends_on:
      - redis

  redis:
    image: redis:7
```

Start:

```bash
docker compose up
```

Start in background:

```bash
docker compose up -d
```

Stop:

```bash
docker compose down
```

Compose is excellent for local development and integration testing. On AWS, production containers commonly run on ECS Fargate, ECS on EC2, or EKS.

---

## 15. Logging

Containers should write logs to stdout and stderr.

Good:

```text
Application -> stdout/stderr -> container runtime -> log collector
```

Avoid writing important logs only inside container files because containers are temporary.

In production, logs are usually collected by:

- Fluent Bit
- Fluentd
- Logstash
- CloudWatch Logs
- Datadog or similar tools

---

## 16. Security Best Practices

Basic security rules:

- Use trusted base images.
- Keep images small.
- Run as non-root user.
- Do not store secrets in images.
- Pin image versions.
- Scan images for vulnerabilities.
- Remove package managers and build tools from runtime images when possible.
- Use read-only filesystems when possible.
- Drop unnecessary Linux capabilities.

Example non-root user:

```dockerfile
RUN addgroup --system app && adduser --system --ingroup app app
USER app
```

---

## 17. Production Dockerfile Checklist

- Uses a specific base image tag
- Uses multi-stage build
- Keeps final image small
- Runs as non-root user
- Does not contain secrets
- Uses `.dockerignore`
- Has clear `ENTRYPOINT` or `CMD`
- Exposes expected port
- Writes logs to stdout/stderr
- Has a health check or supports orchestrator probes
- Uses immutable image tags in production

---

## 18. Common Docker Problems

### Container Exits Immediately

Common causes:

- Main process crashed
- Wrong command
- Missing environment variable
- Application startup failure

Check:

```bash
docker logs <container>
```

### Port Already in Use

Cause:

- Another process is using the host port.

Fix:

- Stop the other process.
- Use a different host port.

Example:

```bash
docker run -p 8081:8080 my-app:1.0
```

### Cannot Connect to Another Container

Common causes:

- Containers are not on the same network.
- Using `localhost` incorrectly.
- Wrong service name.

Remember:

Inside a container, `localhost` means that same container, not your host machine and not another container.

### Image Build Is Slow

Common causes:

- Bad layer ordering
- Large build context
- Missing `.dockerignore`
- Dependencies downloaded every build

Fix:

- Copy dependency files before source code.
- Add `.dockerignore`.
- Use BuildKit cache mounts when useful.

---

## 19. Docker in System Design

In system design interviews, mention Docker when discussing:

- Consistent deployment artifacts
- Microservice packaging
- CI/CD pipelines
- Local development parity
- Horizontal scaling
- Kubernetes or cloud container platforms

Do not claim Docker solves:

- Service discovery by itself
- Multi-host orchestration by itself
- Database replication
- Auto-scaling
- Production secrets management

Docker packages and runs containers. Orchestrators and cloud platforms handle broader production operations.

---

## 20. Quick Revision

- Image: read-only packaged application template.
- Container: running instance of an image.
- Dockerfile: instructions to build an image.
- Registry: stores images.
- Volume: persistent data outside container writable layer.
- Network: allows containers to communicate.
- Compose: runs multi-container apps locally.
- Multi-stage build: separates build environment from runtime image.
- Containers are disposable; durable state belongs outside them.
