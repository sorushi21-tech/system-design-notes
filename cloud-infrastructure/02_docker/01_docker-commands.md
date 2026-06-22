# Docker Commands

## 1. Basic Command Pattern

Most Docker commands follow this pattern:

```bash
docker <object> <action> [options]
```

Examples:

```bash
docker image ls
docker container ls
docker network ls
docker volume ls
```

Common objects:

- `image`: Docker image operations
- `container`: container operations
- `network`: Docker network operations
- `volume`: persistent storage operations
- `compose`: multi-container application operations

For Java services, the most common workflow is build the image, run the container, inspect logs, and iterate on the Dockerfile or JVM options.

---

## 2. Image Workflow

### Build an Image

```bash
docker build -t payment-service:1.0 .
```

Meaning:

- `docker build`: build an image
- `-t payment-service:1.0`: name and tag the image
- `.`: use the current directory as build context

For Java apps, keep the build context narrow and use `.dockerignore` so Maven/Gradle caches, IDE files, and local artifacts do not slow builds.

### Build with Build Args

```bash
docker build --build-arg MAVEN_OPTS='-Xmx2g' -t payment-service:1.0 .
```

Use build args when the image needs build-time configuration, but avoid passing secrets through `--build-arg`.

### List Images

```bash
docker images
```

or:

```bash
docker image ls
```

### Remove an Image

```bash
docker rmi my-app:1.0
```

If a container is using the image, remove the container first.

### Tag an Image for Registry

```bash
docker tag my-app:1.0 registry.example.com/my-app:1.0
```

### Push an Image

```bash
docker push registry.example.com/my-app:1.0
```

Use immutable tags such as Git SHA or version number for production. Avoid relying only on `latest`.

### BuildKit and Buildx

For Java builds with complex caching or multi-platform images, use BuildKit and Buildx:

```bash
docker buildx build --platform linux/amd64 -t my-app:1.0 --load .
```

Use `--pull` to ensure the latest base image and `--progress=plain` to capture logs when debugging build issues.

### Build Args

```bash
docker build --build-arg MAVEN_OPTS='-Xmx2g' -t my-app:1.0 .
```

Use build args for build-time configuration, but never pass secrets through `--build-arg`.

---

## 3. Container Workflow

### Run a Container

```bash
docker run --name my-app -p 8080:8080 my-app:1.0
```

Meaning:

- `--name my-app`: container name
- `-p 8080:8080`: map host port 8080 to container port 8080
- `my-app:1.0`: image to run

### Run in Detached Mode

```bash
docker run -d --name my-app -p 8080:8080 my-app:1.0
```

Detached mode runs the container in the background.

### Environment and JVM Options

```bash
docker run -d --name my-app -p 8080:8080 -e SPRING_PROFILES_ACTIVE=prod -e JAVA_OPTS='-XX:MaxRAMPercentage=75 -XX:+ExitOnOutOfMemoryError' my-app:1.0
```

Pass runtime configuration via environment variables rather than baking profiles or secrets into the image.

### Run Without Port Mapping for Internal Services

```bash
docker run --name batch-worker my-worker:1.0
```

Not every Java container requires published ports. Background jobs and message consumers often run without host port mapping.

### List Running Containers

```bash
docker ps
```

### List All Containers

```bash
docker ps -a
```

### Stop a Container

```bash
docker stop my-app
```

### Start a Stopped Container

```bash
docker start my-app
```

### Remove a Container

```bash
docker rm my-app
```

### Run and Remove Automatically

```bash
docker run --rm my-app:1.0
```

Use `--rm` for temporary containers or command-line utilities.

---

## 4. Logs and Debugging

### View Logs

```bash
docker logs my-app
```

### Follow Logs

```bash
docker logs -f my-app
```

### Execute Command Inside Container

```bash
docker exec -it my-app sh
```

If the image has Bash:

```bash
docker exec -it my-app bash
```

For Java images, using `docker exec` to inspect `ps -ef | grep java` and `cat /app/app.jar` is common when debugging classpath or permission issues.

### Inspect Container Details

```bash
docker inspect my-app
```

Use inspect to check:

- IP address
- Environment variables
- Volume mounts
- Network configuration
- Restart policy

### Check Resource Usage

```bash
docker stats
```

Shows CPU, memory, network, and disk I/O usage.

---

## 5. Networking Commands

### List Networks

```bash
docker network ls
```

### Create Network

```bash
docker network create app-network
```

### Run Container on a Network

```bash
docker run -d --name api --network app-network my-api:1.0
```

### Connect Existing Container to Network

```bash
docker network connect app-network my-app
```

### Inspect Network

```bash
docker network inspect app-network
```

Containers on the same user-defined bridge network can reach each other by container name. See [09_docker-networking.md](09_docker-networking.md) for more.

---

## 6. Volume Commands

### Create Volume

```bash
docker volume create app-data
```

### Use Volume

```bash
docker run -d --name db -v app-data:/var/lib/postgresql/data postgres:16
```

### List Volumes

```bash
docker volume ls
```

### Inspect Volume

```bash
docker volume inspect app-data
```

### Remove Volume

```bash
docker volume rm app-data
```

Be careful: removing a volume can delete persistent data. See [08_docker-storage.md](08_docker-storage.md) for storage patterns.


---

## 7. Docker Compose Workflow

Compose runs multi-container applications. See [03_storage-networking-compose.md](03_storage-networking-compose.md) for full Compose examples.

Start services:

```bash
docker compose up
```

Start in background:

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

Run command in service:

```bash
docker compose exec <service> <command>
```

---

## 8. Registry Commands

### Login to Registry

```bash
docker login registry.example.com
```

### Push Image

```bash
docker push registry.example.com/my-app:1.0
```

### Pull Image

```bash
docker pull registry.example.com/my-app:1.0
```

### Search Registry (Docker Hub)

```bash
docker search redis
```

---

## 9. System Commands

### Show Docker Info

```bash
docker info
```

### Clean Up

Remove stopped containers, unused networks, and dangling images:

```bash
docker system prune
```

Aggressive cleanup (also removes tagged images not used by containers):

```bash
docker system prune -a
```

Remove unused volumes:

```bash
docker volume prune
```

Be careful with volume cleanup. Volumes can contain database data.

### Inspect Builder Cache

```bash
docker buildx du
```

---

## Next

- [00_docker-fundamentals.md](00_docker-fundamentals.md) for core concepts
- [02_dockerfile-java.md](02_dockerfile-java.md) for Java Dockerfile design
- [03_storage-networking-compose.md](03_storage-networking-compose.md) for storage and networking
- [04_runtime-config-health-logging.md](04_runtime-config-health-logging.md) for production behavior
- [05_security-production.md](05_security-production.md) for security practices
- [06_troubleshooting.md](06_troubleshooting.md) for debugging
- [07_system-design-integration.md](07_system-design-integration.md) for system design
