# Docker Security and Production Practices

Docker security is mostly about reducing what is inside the image, limiting what the container can do, and keeping secrets out of build artifacts.

## Image Security

Basic rules:

- Use trusted base images.
- Keep images small.
- Run as non-root user.
- Do not store secrets in images.
- Pin image versions.
- Scan images for vulnerabilities.
- Remove package managers and build tools from runtime images when possible.
- Use read-only filesystems when possible.
- Drop unnecessary Linux capabilities.

## Non-Root Containers

Example:

```dockerfile
RUN addgroup --system app && adduser --system --ingroup app app
USER app
```

For Java images, make sure the non-root user can read the application files and write only to intended temporary directories.

## Base Image Choice

Common Java base image options:

| Image Type | Use Case | Watch For |
| --- | --- | --- |
| Full JDK | Build stage | Larger image |
| JRE | Runtime stage | Smaller runtime |
| Alpine | Very small images | musl compatibility surprises |
| Distroless | Minimal production runtime | Harder shell debugging |

For production, prefer a small, maintained runtime image over a full build image.

## Tags and Digests

Avoid relying on `latest` in production.

Better:

```text
payment-service:1.7.3
payment-service:git-abc1234
payment-service@sha256:...
```

Immutable tags or digests make rollbacks and audits much clearer.

## Secrets

Never put secrets in:

- Dockerfile `ENV`
- Image layers
- Build arguments
- Git-tracked `.env` files
- Application config baked into the image

Use runtime secret injection through the platform:

- Kubernetes Secrets with external secret managers
- AWS Secrets Manager
- AWS SSM Parameter Store
- ECS task secrets
- CI/CD secret stores

## Runtime Hardening

Useful runtime options:

```bash
docker run --read-only --cap-drop=ALL --security-opt no-new-privileges my-app:1.0
```

Not every application works with these options immediately. Java apps may need writable temp space:

```bash
docker run --read-only --tmpfs /tmp my-app:1.0
```

## Supply Chain Checklist

- Build images in CI, not manually on a laptop.
- Scan images before deployment.
- Generate SBOM when required.
- Sign images when the platform supports it.
- Keep base images patched.
- Use immutable deploy references.
- Restrict who can push to production registries.

## Production Readiness Checklist

- Image has no secrets.
- Runtime image does not include Maven, Gradle, or source code.
- Container runs as non-root.
- CPU and memory limits are set in the platform.
- JVM memory leaves room for non-heap usage.
- Logs go to stdout/stderr.
- Health endpoints are available.
- Graceful shutdown is configured.
- Image tag or digest is traceable to a commit.
