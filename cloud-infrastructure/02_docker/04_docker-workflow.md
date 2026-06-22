# Docker Workflow

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

## Build Once, Deploy the Same Artifact

A senior delivery pipeline builds the container image once, scans it once, and deploys the same image digest through staging and production.

This avoids drift between environments and makes rollbacks deterministic:

- CI builds the image and tags it with a commit SHA or version.
- CI scans and pushes the image to a registry.
- Deployments reference the same image digest.
- Promotion is a registry-level event, not a rebuild.

Use this pattern for Java services as well: build the jar and the image in the same pipeline, then deploy the resulting image unchanged.
