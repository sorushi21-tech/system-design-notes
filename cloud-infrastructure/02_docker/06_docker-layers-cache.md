# Docker Layers and Cache

Each Dockerfile instruction creates a layer.

Example:

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY target/app.jar app.jar
```

Docker caches layers. If a layer does not change, Docker reuses it during the next build.

## Layer Design

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
