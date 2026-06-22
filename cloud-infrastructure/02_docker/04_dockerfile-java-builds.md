# Dockerfile for Java Services

This note focuses on Dockerfile design for Java and Spring Boot services.

## Simple Runtime Image

If CI already builds the JAR, the Dockerfile can be simple:

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY target/app.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

This is easy to understand, but it depends on the CI pipeline building `target/app.jar` before `docker build`.

## Multi-Stage Maven Build

Multi-stage builds keep Maven and source code out of the final runtime image.

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /workspace

COPY pom.xml .
RUN mvn -B dependency:go-offline

COPY src ./src
RUN mvn -B clean package -DskipTests

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /workspace/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Benefits:

- Smaller runtime image
- Build tools stay out of production
- Better layer caching
- Fewer runtime vulnerabilities

## Build Cache Pattern

Docker caches layers by instruction and content. Put slow-changing dependency files before fast-changing source code.

Good:

```dockerfile
COPY pom.xml .
RUN mvn -B dependency:go-offline
COPY src ./src
RUN mvn -B package
```

Bad:

```dockerfile
COPY . .
RUN mvn package
```

In the bad version, every source change invalidates dependency download.

## BuildKit Cache Mount

BuildKit can cache Maven dependencies across builds without baking them into the image.

```dockerfile
# syntax=docker/dockerfile:1.7

FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /workspace

COPY pom.xml .
RUN --mount=type=cache,target=/root/.m2 mvn -B dependency:go-offline

COPY src ./src
RUN --mount=type=cache,target=/root/.m2 mvn -B clean package -DskipTests
```

This is useful in CI and local development when image builds happen often.

## Spring Boot Layered JAR

Spring Boot can split a fat JAR into logical layers:

```bash
java -Djarmode=layertools -jar app.jar extract
```

Typical layers:

- Dependencies
- Spring Boot loader
- Snapshot dependencies
- Application classes

This can improve Docker cache reuse because dependencies change less often than application code.

## `.dockerignore`

Always control the build context. Docker sends the build context to the daemon, so large or sensitive files slow builds and can leak into images accidentally.

Example:

```gitignore
.git
.idea
target
build
*.log
*.iml
.env
node_modules
```

If the Dockerfile expects `target/app.jar`, do not ignore `target`. If the Dockerfile builds the JAR inside Docker, ignoring `target` is usually correct.

## ENTRYPOINT vs CMD

Use `ENTRYPOINT` for the executable and `CMD` for default arguments.

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]
CMD ["--spring.profiles.active=local"]
```

For production Java services, keep runtime configuration in environment variables or platform config rather than hardcoding profiles in the image.

## Java Runtime Options

Common JVM options:

```text
-XX:MaxRAMPercentage=75
-XX:InitialRAMPercentage=50
-XX:+ExitOnOutOfMemoryError
-Dfile.encoding=UTF-8
```

Modern JVMs understand container CPU and memory limits, but production services should still set heap behavior intentionally.

## Production Dockerfile Checklist

- Uses a specific base image tag
- Uses multi-stage build or a CI-built artifact
- Keeps final image small
- Runs as non-root user
- Does not contain secrets
- Uses `.dockerignore`
- Has clear `ENTRYPOINT` or `CMD`
- Exposes expected port
- Writes logs to stdout/stderr
- Supports health checks or orchestrator probes
- Uses immutable image tags or digests in production
