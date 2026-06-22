# Docker Layers and Cache

Each Dockerfile instruction creates a layer. Good layer design is essential for Java builds, because Maven and Gradle dependency resolution can be expensive.

## How Docker Caches Layers

Docker caches each layer by instruction and the contents of the files used by that instruction. If the instruction and input files do not change, Docker reuses the cached layer.

Example:

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY target/app.jar app.jar
```

If `target/app.jar` does not change, that layer is reused.

## Layer Design for Java

Good layer design makes builds faster and more predictable.

Bad pattern:

```dockerfile
COPY . .
RUN mvn package
```

Any source code change invalidates the dependency download and rebuilds too much.

Better pattern for Maven:

```dockerfile
COPY pom.xml .
RUN mvn -B dependency:go-offline
COPY src ./src
RUN mvn -B package -DskipTests
```

Better pattern for Gradle:

```dockerfile
COPY build.gradle settings.gradle .
RUN gradle --no-daemon dependencies
COPY src ./src
RUN gradle --no-daemon bootJar
```

This separates slow-changing dependency resolution from fast-changing application source.

## BuildKit Cache Mounts

With BuildKit, you can cache Maven or Gradle dependencies without baking the cache into the image.

```dockerfile
# syntax=docker/dockerfile:1.7

FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /workspace
COPY pom.xml .
RUN --mount=type=cache,target=/root/.m2 mvn -B dependency:go-offline
COPY src ./src
RUN --mount=type=cache,target=/root/.m2 mvn -B clean package -DskipTests
```

For Gradle:

```dockerfile
# syntax=docker/dockerfile:1.7

FROM gradle:8.4-jdk21 AS build
WORKDIR /workspace
COPY build.gradle settings.gradle gradle.properties .
RUN --mount=type=cache,target=/home/gradle/.gradle gradle --no-daemon dependencies
COPY src ./src
RUN --mount=type=cache,target=/home/gradle/.gradle gradle --no-daemon bootJar
```

## Reproducible Builds

To keep Java images reproducible:

- Pin base image versions.
- Pin dependency versions in `pom.xml` or `build.gradle`.
- Use a deterministic build stage.
- Avoid extracting build output from local host directories into the image.

## Multi-Stage Builds

Multi-stage builds keep build tools out of the final runtime image.

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
- Build tools and source code remain out of runtime

## Spring Boot Layered JARs

Spring Boot can create layered JARs that align well with Docker build cache.

If you build a layered jar, copy only the immutable dependency layers first and application classes last:

```dockerfile
FROM eclipse-temurin:21-jre AS runtime
WORKDIR /app
COPY --from=build /workspace/target/dependency/ /app/dependency/
COPY --from=build /workspace/target/spring-boot-loader/ /app/spring-boot-loader/
COPY --from=build /workspace/target/snapshot-dependencies/ /app/snapshot-dependencies/
COPY --from=build /workspace/target/application/ /app/application/
ENTRYPOINT ["java", "-cp", "/app/dependency/*:/app/application", "org.springframework.boot.loader.JarLauncher"]
```

This allows the dependency layers to remain cached when only application code changes.

## Jib and Java Image Tooling

For Java developers, Jib is an alternative to manual Dockerfiles. It builds optimized Java container images directly from Maven or Gradle and manages layers automatically.

If you prefer standard Dockerfiles, ensure your layer ordering mirrors how Java dependencies change:

1. OS and runtime base image
2. Build dependencies
3. Application dependencies
4. Application classes

## Common Cache Pitfalls

- Copying the entire build context too early.
- Including `.git` or IDE files in build context.
- Not using `.dockerignore`.
- Running `mvn package` before copying `pom.xml` and dependencies.
- Relying on host-specific temporary directories.
