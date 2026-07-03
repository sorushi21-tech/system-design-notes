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

- A container is a running instance of an image.
- It is encapsulated environment in which application runs.
- It packages the application and its dependencies into a single executable unit.

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

## Image Tags, Digests, and Immutability

Tags are mutable labels for images. In production, prefer image digests (`@sha256:...`) or immutable tags tied to a build or commit.

A digest uniquely identifies the image content, while a tag is a convenient alias. For example:

```text
my-app:1.0
my-app@sha256:abcdef123456...
```

This is important for reproducible deployments and rollbacks.

## How Image Layers Work

Images are built as a stack of immutable layers. Each Dockerfile instruction creates a layer. When you start a container, Docker mounts a new writable layer on top of those read-only layers.

That means:

- layers can be shared across images and containers
- small changes produce smaller deltas
- the writable layer only contains runtime changes

For Java apps, this is why layered jars and careful layer ordering matter: dependency and runtime layers can be reused across builds.
