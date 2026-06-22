# Container Filesystem

A container has a writable layer on top of the image.

Important rule:

**Containers are disposable.** Do not store important long-term data only inside a container filesystem.

If the container is removed, its writable layer is removed too.

## Copy-on-Write Semantics

The writable layer is implemented with a copy-on-write filesystem. This means:

- Reads come from lower image layers.
- Writes are stored in the container layer.
- Changing a file copied from the image creates new data in the writable layer.

This is efficient for typical container workloads, but it also means that:

- container filesystem writes are slower than plain host filesystem writes
- large runtime changes can increase container size
- state should not be kept only in the writable layer

## Durable Storage

Use external storage for durable data:

- Docker volumes
- Bind mounts
- Object storage
- Databases
- Managed file storage
