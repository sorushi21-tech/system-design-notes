# Container Filesystem

A container has a writable layer on top of the image.

Important rule:

**Containers are disposable.** Do not store important long-term data only inside a container filesystem.

If the container is removed, its writable layer is removed too.

## Durable Storage

Use external storage for durable data:

- Docker volumes
- Bind mounts
- Object storage
- Databases
- Managed file storage
