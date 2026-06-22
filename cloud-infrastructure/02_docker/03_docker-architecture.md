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

## Registry

A registry stores Docker images.

Examples:

- Docker Hub
- Amazon ECR
- GitHub Container Registry
