# Docker Overview

Docker packages an application and its runtime dependencies into a container image. The same image can run on a developer laptop, CI runner, test environment, or production container platform.

For a Java backend developer, Docker matters because it turns a Spring Boot service into a repeatable deployment artifact.

```text
Java source code
  -> Maven or Gradle build
  -> JAR file
  -> Docker image
  -> Container
  -> ECS, EKS, Kubernetes, or another runtime
```

## Why Docker Exists

Before containers, teams often had problems like:

- Application works on one machine but fails on another.
- Different projects need different Java, Node, Python, or system library versions.
- Deployments require long manual setup instructions.
- Production servers drift from each other over time.
- Scaling requires configuring many machines by hand.

Docker helps with:

- Environment consistency
- Faster local setup
- Reproducible builds
- Easier CI/CD
- Isolated services
- Cloud-native deployment

Docker does not replace good application design. It packages and runs processes; orchestration, service discovery, autoscaling, traffic routing, and secrets management are usually handled by Kubernetes, ECS, Nomad, or cloud services.

## Core Workflow

The basic Docker workflow is:

```text
Write Dockerfile
  -> build image
  -> run container locally
  -> test and inspect
  -> tag image
  -> push image to registry
  -> deploy image through orchestrator
```

Example:

```bash
docker build -t payment-service:1.0 .
docker run --rm -p 8080:8080 payment-service:1.0
docker tag payment-service:1.0 registry.example.com/payment-service:1.0
docker push registry.example.com/payment-service:1.0
```

## Where Docker Fits in Production

Typical production flow:

```text
Developer commit
  -> CI build and test
  -> build container image
  -> scan image
  -> push image to registry
  -> deploy by image tag or digest
  -> collect logs, metrics, and traces
```

On AWS:

```text
Spring Boot image
  -> Amazon ECR
  -> ECS Fargate or EKS
  -> ALB
  -> CloudWatch/OpenTelemetry
  -> RDS, Redis, SQS, and other dependencies
```

## Quick Mental Model

- Image: read-only packaged application template.
- Container: running instance of an image.
- Dockerfile: instructions used to build an image.
- Registry: remote storage for images.
- Volume: persistent storage outside the container writable layer.
- Network: communication boundary between containers.
- Compose: local multi-container environment.
- Orchestrator: production system that schedules and manages containers.
