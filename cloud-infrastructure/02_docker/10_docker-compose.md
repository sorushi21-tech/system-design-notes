# Docker Compose

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
