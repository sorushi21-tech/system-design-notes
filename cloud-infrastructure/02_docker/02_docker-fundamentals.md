# Docker Fundamentals

Docker is a container platform used to package an application with everything it needs to run: application code, runtime, libraries, environment variables, and operating-system level dependencies.

This file is the core concept note. Related senior Java Docker topics are split into smaller files:

- `01_docker-overview.md`
- `03_docker-command-workflows.md`
- `04_dockerfile-java-builds.md`
- `05_storage-networking-compose.md`
- `06_runtime-config-health-logging.md`
- `07_security-production.md`
- `08_troubleshooting-docker.md`
- `09_docker-in-system-design.md`

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

## 9. Next Topics

Continue with the focused notes in this folder:

- `03_docker-command-workflows.md` for daily Docker commands.
- `04_dockerfile-java-builds.md` for Java image builds.
- `05_storage-networking-compose.md` for volumes, networks, and Compose.
- `06_runtime-config-health-logging.md` for production runtime behavior.
- `07_security-production.md` for security and release practices.
- `08_troubleshooting-docker.md` for debugging.
- `09_docker-in-system-design.md` for interview and architecture framing.

---

## 10. Quick Revision

- Image: read-only packaged application template.
- Container: running instance of an image.
- Dockerfile: instructions to build an image.
- Registry: stores images.
- Layer: reusable filesystem delta from a Dockerfile instruction.
- Multi-stage build: separates build environment from runtime image.
- Containers are disposable; durable state belongs outside them.
