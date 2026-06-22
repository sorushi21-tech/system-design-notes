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
