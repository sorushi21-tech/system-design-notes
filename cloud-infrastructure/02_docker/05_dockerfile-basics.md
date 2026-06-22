# Dockerfile Basics

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

## Dockerfile Best Practices

- Use a specific base image tag, not `latest`.
- Keep the runtime image small.
- Avoid embedding secrets in the Dockerfile.
- Use `.dockerignore` to keep build context small.
- Prefer `ENTRYPOINT` for the executable and `CMD` for default arguments.
