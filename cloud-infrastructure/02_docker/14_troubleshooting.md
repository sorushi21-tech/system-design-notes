# Docker Troubleshooting

Most local Docker issues come from port conflicts, missing environment variables, wrong network names, file permission problems, or containers exiting immediately because the main process failed.

## Debug Checklist

1. Check if the container exists:

```bash
docker ps -a
```

2. Check logs:

```bash
docker logs <container>
```

3. Check port mapping:

```bash
docker port <container>
```

4. Inspect configuration:

```bash
docker inspect <container>
```

5. Enter the container:

```bash
docker exec -it <container> sh
```

6. Check resource usage:

```bash
docker stats
```

## Container Exits Immediately

Common causes:

- Main process crashed.
- Wrong command.
- Missing environment variable.
- Application startup failure.
- Java cannot find the JAR file.
- Spring profile points to unavailable dependencies.

Check:

```bash
docker logs <container>
```

For a one-shot debug:

```bash
docker run --rm -it --entrypoint sh my-app:1.0
```

## Port Already in Use

Cause:

- Another process is using the host port.

Fix:

- Stop the other process.
- Use a different host port.

Example:

```bash
docker run -p 8081:8080 my-app:1.0
```

## Cannot Connect to Another Container

Common causes:

- Containers are not on the same network.
- Using `localhost` incorrectly.
- Wrong service name.
- Dependency is running but not ready yet.

Remember:

```text
Inside a container, localhost means that same container.
```

Use container or Compose service names for container-to-container calls.

## Cannot Connect to Host from Container

On Docker Desktop, use:

```text
host.docker.internal
```

For Linux hosts, this may need explicit configuration depending on Docker version and network mode.

## Image Build Is Slow

Common causes:

- Bad layer ordering.
- Large build context.
- Missing `.dockerignore`.
- Dependencies downloaded every build.
- Tests running during every image build.

Fix:

- Copy dependency files before source code.
- Add `.dockerignore`.
- Use BuildKit cache mounts when useful.
- Build application artifacts in CI when appropriate.

## Java Container OOM

Symptoms:

- Container exits with code `137`.
- Logs stop suddenly.
- Orchestrator reports OOMKilled.

Common causes:

- Heap too large for container memory.
- Too many threads.
- Direct buffers or native memory usage.
- Large request payloads.
- Memory leak.

Actions:

- Reduce `MaxRAMPercentage`.
- Increase container memory limit.
- Check thread count and connection pool sizes.
- Capture heap dump in a controlled environment.
- Add memory metrics and alerts.

## Java-Specific Startup Failures

If a Java service starts but exits immediately, common container-related causes include:

- Missing or mislocated JAR file in the image.
- Wrong `ENTRYPOINT` or `CMD` causing the JVM to start with invalid args.
- File permission errors when running as a non-root user.
- `NoSuchFileException` for `application.yml` or config directories.
- `ClassNotFoundException` or `NoClassDefFoundError` when the jar is incorrect.

Check the first lines of `docker logs <container>` and compare the container entrypoint with the expected artifact path.

## Java Networking and Binding Issues

If a Spring Boot app fails to bind:

- Confirm `server.port` matches the exposed container port.
- Avoid binding only to localhost inside the container.
- Use `server.address=0.0.0.0` for containerized apps.
- Ensure no other process in the container is already listening on that port.

If the app can’t reach an external service, verify the JDBC/Redis host and network mode.

## Health Check Fails

Common causes:

- Wrong health endpoint path.
- Application starts slower than health interval.
- Health check depends on a slow external dependency.
- Port mismatch between app and Dockerfile or platform config.

For Spring Boot, verify:

```text
/actuator/health
/actuator/health/liveness
/actuator/health/readiness
```

## Cleanup Commands

Remove stopped containers, unused networks, and dangling images:

```bash
docker system prune
```

Aggressive cleanup:

```bash
docker system prune -a
```

Remove unused volumes:

```bash
docker volume prune
```

Be careful with volume cleanup. Volumes can contain database data.
