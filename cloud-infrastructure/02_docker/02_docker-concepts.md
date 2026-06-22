# Docker Concepts

## Image vs Container

### Image

An image is a read-only template.

It contains:

- Application code
- Runtime, such as JDK or Node.js
- System libraries
- Default command
- Filesystem layers
- Metadata

Think of an image as a packaged blueprint.

### Container

A container is a running instance of an image.

Example:

```bash
docker run payment-service:1.0
```

If an image is like a class, a container is like an object created from that class.

### Multiple Containers from One Image

Multiple containers can run from the same image:

```text
payment-service:1.0 image
  -> container A
  -> container B
  -> container C
```

## Containers vs Virtual Machines

| Area      | Virtual Machine                   | Container                 |
|-----------|-----------------------------------|---------------------------|
| Isolation | Full guest OS                     | Process-level isolation   |
| Startup   | Slower                            | Faster                    |
| Size      | Larger                            | Smaller                   |
| Kernel    | Separate guest kernel             | Shares host kernel        |
| Use case  | Strong isolation, full OS control | App packaging and scaling |

Containers are lighter because they do not run a full guest operating system. They share the host kernel but isolate processes, filesystems, networks, and resources.
