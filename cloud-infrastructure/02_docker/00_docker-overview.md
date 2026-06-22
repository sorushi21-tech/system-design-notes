# Docker Overview

Docker packages an application and its runtime dependencies into a container image. The same image can run on a developer laptop, CI runner, test environment, or production container platform.

For a Java backend developer, Docker matters because it turns a Spring Boot service into a repeatable deployment artifact.

A Docker image should be treated as an immutable artifact: build it once, promote it through environments, and run it unchanged. The container is the ephemeral process that executes that image with a writable runtime layer.

Docker does not replace good application design. It packages and runs processes; orchestration, service discovery, autoscaling, traffic routing, and secrets management are usually handled by Kubernetes, ECS, Nomad, or cloud services.

## Quick Mental Model

- Image: read-only packaged application template.
- Container: running instance of an image.
- Dockerfile: instructions used to build an image.
- Registry: remote storage for images.
- Volume: persistent storage outside the container writable layer.
- Network: communication boundary between containers.
- Compose: local multi-container environment.
- Orchestrator: production system that schedules and manages containers.
