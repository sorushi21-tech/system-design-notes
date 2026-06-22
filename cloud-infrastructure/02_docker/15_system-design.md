# Docker in System Design

Docker is not the whole platform. In system design, it is best explained as the packaging and runtime unit for services.

## When to Mention Docker

Mention Docker when discussing:

- Consistent deployment artifacts
- Microservice packaging
- CI/CD pipelines
- Local development parity
- Horizontal scaling units
- Kubernetes, ECS, or other container platforms
- Blue-green or canary deployments by image version

Good phrasing:

```text
Each service is packaged as an immutable Docker image and deployed by version through the orchestration platform.
```

## What Docker Does Not Solve Alone

Do not claim Docker solves:

- Service discovery by itself
- Multi-host orchestration by itself
- Database replication
- Auto-scaling
- Production secrets management
- Traffic shifting
- Zero-downtime deployments

Those are usually handled by Kubernetes, ECS, service meshes, load balancers, deployment controllers, or cloud services.

## Java Backend Deployment Example

```text
Spring Boot service
  -> Docker image
  -> ECR
  -> ECS Fargate service
  -> ALB target group
  -> CloudWatch logs and metrics
  -> RDS and Redis
```

Important design points:

- The image is stateless.
- Configuration is injected at runtime.
- Secrets come from a secret manager.
- Logs go to stdout/stderr.
- Health checks protect traffic routing.
- The service handles graceful shutdown.
- Rollback means deploying the previous image version.
- The Docker image should be treated as the deployable artifact, not as an environment.

For Java teams, this means the CI pipeline should build the JAR and the image together, preserving artifact immutability and traceability.

## Stateful vs Stateless

Stateless services fit containers well:

- REST APIs
- Background workers
- Event consumers
- WebSocket gateways with external session storage

Stateful systems need extra care:

- Databases
- Message brokers
- Search clusters
- File storage

In interviews, prefer managed stateful services unless there is a reason to operate them yourself.

## Deployment Strategy

Container image versions make deployment strategies easier:

| Strategy | Meaning | Docker Role |
| --- | --- | --- |
| Rolling | Replace instances gradually | New image version rolled out slowly |
| Blue-green | Switch traffic between old and new environments | Old and new image versions run side by side |
| Canary | Send small traffic percentage to new version | New image tested with limited traffic |
| Rollback | Return to previous version | Redeploy previous image tag or digest |

## Senior-Level Tradeoffs

Discuss these tradeoffs when relevant:

- Smaller images improve cold starts and pull time.
- Distroless images improve security but make shell debugging harder.
- Running as non-root improves isolation but may expose file permission issues.
- Aggressive health checks can cause restart loops.
- Too-large JVM heap can trigger container OOM kills.
- Mutable tags like `latest` make debugging and rollback harder.
