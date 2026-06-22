# Docker Networking

Containers need networking to talk to users, other containers, and external services. For Java microservices, networking also shapes how JDBC URLs, cache hosts, and service discovery are configured.

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

Java services often expose application metrics and health endpoints on the same port. Make sure the Dockerfile `EXPOSE` matches the port your Spring Boot app uses.

## Container DNS and Service Names

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

For a Java app, this means:

```properties
spring.datasource.url=jdbc:postgresql://postgres:5432/mydb
```

not `jdbc:postgresql://localhost:5432/mydb`.

## Host and Container Access

On Docker Desktop, containers can reach the host via:

```text
host.docker.internal
```

For Linux, use network mode or a host gateway configuration when you need the container to access host services.

## Debugging Networking for Java Apps

Useful checks:

- `docker network inspect app-network`
- `docker exec -it api ping redis`
- `docker exec -it api curl -f http://redis:6379`
- `docker logs api`

If a Spring Boot service cannot connect to a database, verify the JDBC URL, network, service name, and environment variables.

## Service Discovery and Local Development

For local Compose networks, use service names as hostnames and keep configuration environment-driven.

Example:

```bash
SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/mydb
```

For cloud or Kubernetes, these names are replaced by platform DNS names or service discovery mechanisms.
