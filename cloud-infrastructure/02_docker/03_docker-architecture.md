# Docker Architecture

Docker uses a client-server architecture.

```text
Docker CLI
  -> Docker daemon
  -> container runtime
  -> containers
```

## Docker Client

The command-line tool you use:

```bash
docker build
docker run
docker ps
```

## Docker Daemon

The background service that manages:

- Images
- Containers
- Networks
- Volumes
- Builds

Daemon functionality:

- The docker daemon also known as **dockerd**. It listens to docker API request from docker client and manage container, images, network and volumes.
- It builds container images as request by the client.
- It interfaces with docker registries to pull or publish images as requested by the client.
- It manages the lifecycle of the container.

Modern Docker implementations delegate container execution to lower-level components such as `containerd` and `runc`. BuildKit is frequently used for build acceleration and secret/SSH mounts.

Docker can also run in rootless mode for improved developer security, though rootless mode may require additional setup for network and filesystem access.

## Registry

A registry stores Docker images.

Examples:

- Docker Hub
- Amazon ECR
- GitHub Container Registry
