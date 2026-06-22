# Dockerfile Basics

A Dockerfile defines how to build an image. For a senior Java developer, the Dockerfile should also reflect the Java build pipeline and production JVM behavior.

## Base Image Selection for Java

Common runtime choices:

- `eclipse-temurin:21-jre` or `eclipse-temurin:21-jre-jammy` for production runtime.
- `eclipse-temurin:21-jdk` for build stages or local debug images.
- `gcr.io/distroless/java17-debian11` for minimal runtime images.
- `adoptopenjdk:21-jre-hotspot` or `amazoncorretto:21-alpine` when you need distribution-specific behavior.

Choose the smallest, well-maintained image that supports your JVM version and native dependencies.

## Simple Runtime Image

If CI already builds the JAR, the Dockerfile can be simple:

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY target/app.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

This pattern is easy to reason about, but it depends on your pipeline to produce `target/app.jar`.

## Multi-Stage Java Build

Multi-stage builds keep Maven or Gradle out of the final runtime image.

### Maven example

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

### Gradle example

```dockerfile
FROM gradle:8.4-jdk21 AS build
WORKDIR /workspace
COPY build.gradle settings.gradle .
COPY gradle ./gradle
RUN gradle --no-daemon dependencies
COPY src ./src
RUN gradle --no-daemon bootJar

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /workspace/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## `.dockerignore`

Always control the build context. Docker sends the build context to the daemon, so large or sensitive files slow builds and can leak into images.

Example:

```
.git
.idea
target
build
.gradle
*.log
*.iml
.env
node_modules
```

If the Dockerfile expects `target/app.jar`, do not ignore `target`. If the Dockerfile builds the artifact inside Docker, ignoring `target` is usually correct.

## Dockerfile Metadata and Build Arguments

Use `LABEL` for image metadata and `ARG` for build-time parameters.

```dockerfile
ARG APP_VERSION=1.0
LABEL maintainer="team@example.com" \
      org.opencontainers.image.version="$APP_VERSION" \
      org.opencontainers.image.source="https://github.com/org/repo"
```

`ARG` values are not secret-safe. Do not use them for credentials.

Use `COPY --chown=app:app` when building a non-root runtime image to avoid file permission issues.

## ENTRYPOINT vs CMD

Use `ENTRYPOINT` for the executable and `CMD` for default runtime arguments.

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]
CMD ["--spring.profiles.active=local"]
```

For production Java services, keep environment-specific configuration outside the image. Do not hardcode `spring.profiles.active=prod` in the image.

## JDK vs JRE vs Distroless

- Use the JDK only during build stages.
- Use a JRE or runtime-only base image in production.
- Consider distroless or slim base images for security, but remember they make container debugging harder.

## Advanced Java Image Tooling

For build pipelines, consider using Jib when you want Java-aware layering without a Docker daemon. Jib integrates with Maven and Gradle and creates reproducible images that mirror Java dependency changes.

If you stay with Dockerfiles, keep image layers aligned with dependency change frequency and avoid copying the entire repository before dependency resolution.

## Java Runtime Options in Dockerfiles

Do not bake environment-specific JVM flags into the image. Prefer runtime injection via `JAVA_OPTS` or platform config.

For example, set heap behavior at runtime:

```bash
docker run -e JAVA_OPTS='-XX:MaxRAMPercentage=75 -XX:+ExitOnOutOfMemoryError' my-app:1.0
```

## Dockerfile Best Practices

- Use a specific base image tag, not `latest`.
- Keep the runtime image small.
- Avoid embedding secrets in the Dockerfile.
- Use `.dockerignore` to keep build context small.
- Prefer `ENTRYPOINT` for the executable and `CMD` for default arguments.
- Separate build tooling from runtime via multi-stage builds.
- Use `COPY --chown=` to avoid permission issues when running as non-root.
- Expose only the application port, not internal build or probe ports.
