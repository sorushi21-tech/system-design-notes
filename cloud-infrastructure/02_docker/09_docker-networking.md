# Docker Networking

Containers need networking to talk to users, other containers, and external services.

## Network Types

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

## Container DNS

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
