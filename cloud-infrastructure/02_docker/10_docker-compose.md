# Docker Compose

Docker Compose runs multiple containers from one YAML file. It is a powerful local development and integration test tool for Java microservices.

Use `docker compose -f docker-compose.yml -f docker-compose.dev.yml up` to separate local overrides from reusable service definitions.

## Example: Spring Boot API with Postgres and Redis

```yaml
services:
  api:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: local
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/myapp
      SPRING_REDIS_HOST: redis
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 5
    volumes:
      - ./logs:/app/logs

  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: example
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "myapp"]
      interval: 10s
      timeout: 5s
      retries: 5
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
```

## Compose Commands

Start services:

```bash
docker compose up
```

Start in the background:

```bash
docker compose up -d
```

Stop services:

```bash
docker compose down
```

Rebuild and start:

```bash
docker compose up --build
```

View logs:

```bash
docker compose logs -f
```

Run a command in a service:

```bash
docker compose exec api sh
```

## Java-Specific Compose Notes

- `depends_on` controls container start order, but it does not guarantee the application is ready. Use health checks or application retry logic.
- Keep local credentials isolated from production. Use `.env` or Compose secrets for local development if needed.
- Use service names as hostnames: `postgres`, `redis`, `kafka`, etc.
- Map logs or temp files only for local development; production platforms often handle logs and storage externally.
- Use `SPRING_PROFILES_ACTIVE` or `SPRING_APPLICATION_JSON` to keep config environment-driven.
- Avoid binding the entire source directory into a production image. For local dev, use bind mounts carefully to preserve build consistency.

## When to Use Compose

Compose is best for:

- Local integration testing
- Quick multi-service Java development environments
- Reproducing production-like dependency topology

Compose is not a production scheduler. On AWS, production containers commonly run on ECS Fargate, ECS on EC2, or EKS.
