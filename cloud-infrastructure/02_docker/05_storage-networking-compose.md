# Docker Storage, Networking, and Compose

Containers are disposable. Storage and network design decide whether a local environment behaves predictably.

## Storage

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

Types of storage:

| Type         | Meaning                          | Use Case                             |
|--------------|----------------------------------|--------------------------------------|
| Named volume | Managed by Docker                | Local databases, persistent app data |
| Bind mount   | Host path mounted into container | Local development                    |
| tmpfs        | In-memory mount                  | Temporary sensitive data             |

Use named volumes for local databases. Use bind mounts when you want live file changes from the host.

## Networking

Containers need networking to talk to users, other containers, and external services.

Common Docker network types:

| Network | Meaning                         | Use Case                                |
|---------|---------------------------------|-----------------------------------------|
| bridge  | Default local container network | Local development                       |
| host    | Container shares host network   | Performance or special networking cases |
| overlay | Multi-host network              | Swarm or orchestrated environments      |
| none    | No network                      | Isolated workloads                      |

## Port Mapping

Container ports are internal unless published.

```bash
docker run -p 8080:8080 my-app:1.0
```

Meaning:

```text
host port 8080 -> container port 8080
```

If host port `8080` is already used:

```bash
docker run -p 8081:8080 my-app:1.0
```

## Container-to-Container DNS

On a user-defined bridge network, containers can reach each other by name.

```bash
docker network create app-network
docker run -d --name redis --network app-network redis:7
docker run -d --name api --network app-network my-api:1.0
```

The `api` container can reach Redis using hostname `redis`.

Important:

```text
localhost inside a container means that same container.
```

It does not mean the host machine and does not mean another container.

## Docker Compose

Docker Compose runs multiple containers from one YAML file.

Example:

```yaml
services:
  api:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: local
      REDIS_HOST: redis
    depends_on:
      redis:
        condition: service_healthy

  redis:
    image: redis:7
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
```

Start:

```bash
docker compose up
```

Start in background:

```bash
docker compose up -d
```

Stop:

```bash
docker compose down
```

Compose is excellent for local development and integration testing. On AWS, production containers commonly run on ECS Fargate, ECS on EC2, or EKS.

## Compose Notes for Java Developers

- `depends_on` controls start order, not application readiness, unless health conditions are used.
- Application code should still retry database and cache connections.
- Use service names as hostnames: `postgres`, `redis`, `kafka`.
- Keep local credentials fake and isolated from production.
- Avoid committing real `.env` files.
