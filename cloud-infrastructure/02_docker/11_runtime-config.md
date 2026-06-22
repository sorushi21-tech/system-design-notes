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
