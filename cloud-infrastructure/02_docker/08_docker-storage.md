# Docker Storage

A container has a writable layer on top of the image. That writable layer disappears when the container is removed.

Use external storage for durable data:

- Docker volumes
- Bind mounts
- Managed databases
- Object storage
- Persistent volumes in Kubernetes

## Volumes

Volumes persist data outside the container writable layer.

Create volume:

```bash
docker volume create app-data
```

Use volume:

```bash
docker run -v app-data:/data my-app:1.0
```

### Types of storage

| Type         | Meaning                          | Use Case                             |
|--------------|----------------------------------|--------------------------------------|
| Named volume | Managed by Docker                | Local databases, persistent app data |
| Bind mount   | Host path mounted into container | Local development                    |
| tmpfs        | In-memory mount                  | Temporary sensitive data             |

Use named volumes for local databases. Use bind mounts when you want live file changes from the host.
