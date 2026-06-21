# Docker Command Workflows

This note explains practical Docker commands in a step-by-step way. The goal is not to memorize every command, but to understand the workflow: build an image, run a container, inspect it, debug it, clean it up, and publish it.

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

---

## 2. Image Workflow

### Build an Image

```bash
docker build -t my-app:1.0 .
```

Meaning:

- `docker build`: build an image
- `-t my-app:1.0`: name and tag the image
- `.`: use the current directory as build context

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

Use `--rm` for temporary containers.

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

Containers on the same user-defined bridge network can reach each other by container name.

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

Be careful: removing a volume can delete persistent data.

---

## 7. Docker Compose Workflow

Compose runs multi-container applications.

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
docker compose exec api sh
```

---

## 8. Command Chaining

Command chaining runs multiple commands together.

| Operator | Meaning |
| --- | --- |
| `&&` | Run next command only if previous command succeeds |
| `||` | Run next command only if previous command fails |
| `;` | Run next command regardless of success or failure |
| `|` | Send output of one command into another command |
| `$()` | Use output of a command as an argument |

### Safe Build and Run

```bash
docker build -t my-app:dev . && docker run --rm -p 8080:8080 my-app:dev
```

This runs the container only if the build succeeds.

### Build, Tag, and Push

```bash
docker build -t my-app:1.0 . && docker tag my-app:1.0 registry.example.com/my-app:1.0 && docker push registry.example.com/my-app:1.0
```

### Stop and Remove Container

```bash
docker stop my-app && docker rm my-app
```

### Remove All Stopped Containers

```bash
docker ps -aq -f status=exited | xargs docker rm
```

### Remove Dangling Images

```bash
docker images -f dangling=true -q | xargs docker rmi
```

---

## 9. Cleanup Commands

### Remove Stopped Containers, Unused Networks, and Dangling Images

```bash
docker system prune
```

### Aggressive Cleanup

```bash
docker system prune -a
```

This removes unused images too. Be careful because future builds may need to download layers again.

### Remove Unused Volumes

```bash
docker volume prune
```

Be very careful with volume cleanup. Volumes can contain database data.

---

## 10. Practical Debug Checklist

When a container does not work:

1. Check if it is running:

```bash
docker ps -a
```

2. Check logs:

```bash
docker logs <container>
```

3. Check port mapping:

```bash
docker port <container>
```

4. Inspect configuration:

```bash
docker inspect <container>
```

5. Enter the container:

```bash
docker exec -it <container> sh
```

6. Check resource usage:

```bash
docker stats
```

Most local Docker issues come from port conflicts, missing environment variables, wrong network names, file permission problems, or containers exiting immediately because the main process failed.
