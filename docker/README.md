# Docker Cheatsheet

Docker is an open-source platform that packages applications and their dependencies into lightweight, portable containers. Containers share the host OS kernel but run in isolated user-space environments, making them faster and more resource-efficient than virtual machines. Docker standardises the build, ship, and run lifecycle — from a developer's laptop through staging to production — and underpins modern CI/CD, microservices, and cloud-native architectures.

---

## Table of Contents

1. [Installation & Setup](#installation--setup)
2. [Images](#images)
3. [Containers](#containers)
4. [Networking](#networking)
5. [Volumes & Storage](#volumes--storage)
6. [Dockerfile Best Practices](#dockerfile-best-practices)
7. [Docker Compose](#docker-compose)
8. [Registry & Distribution](#registry--distribution)
9. [Logging & Monitoring](#logging--monitoring)
10. [Resource Management](#resource-management)
11. [Security](#security)
12. [Cleanup](#cleanup)
13. [Common Patterns & Tips](#common-patterns--tips)

---

## Installation & Setup

### Installing Docker Engine

```bash
# Ubuntu / Debian (official convenience script)
curl -fsSL https://get.docker.com | sh

# Add current user to docker group (avoids sudo)
sudo usermod -aG docker $USER
newgrp docker

# Verify installation
docker version
docker info
```

### `docker version`

Displays client and server version information.

```bash
docker version
docker version --format '{{.Server.Version}}'
```

### `docker info`

Shows system-wide information: number of containers and images, storage driver, kernel version, and resource limits.

```bash
docker info
docker info --format '{{.MemTotal}}'
```

### Context Management

Docker contexts allow you to switch between different Docker endpoints (local daemon, remote host, Kubernetes).

```bash
docker context ls                                   # List contexts
docker context use default                          # Switch context
docker context create remote --docker "host=ssh://user@host"
```

---

## Images

Images are read-only, layered file-system snapshots. Each layer represents a Dockerfile instruction. Layers are cached and shared across images that use the same base.

### `docker build`

Builds a Docker image from a Dockerfile.

| Flag / Option | Description |
|---|---|
| `-t <name>:<tag>` | Name and optionally tag the image |
| `-f <Dockerfile>` | Specify an alternative Dockerfile path |
| `--build-arg <k=v>` | Pass build-time variables |
| `--no-cache` | Do not use layer cache |
| `--target <stage>` | Build up to a specific multi-stage target |
| `--platform <platform>` | Set target platform (e.g., `linux/amd64,linux/arm64`) |
| `--squash` | Squash all new layers into one (experimental) |
| `--pull` | Always attempt to pull a newer base image |

```bash
docker build -t myapp:1.0 .
docker build -t myapp:1.0 -f docker/Dockerfile .
docker build --no-cache -t myapp:latest .
docker build --build-arg NODE_ENV=production -t myapp:prod .
docker build --target builder -t myapp:build-stage .

# Multi-platform build (requires buildx)
docker buildx build --platform linux/amd64,linux/arm64 \
  -t myorg/myapp:latest --push .
```

### `docker images` / `docker image ls`

Lists locally available images.

| Flag / Option | Description |
|---|---|
| `-a` | Show all images including intermediates |
| `-q` | Only display image IDs |
| `--filter <key=val>` | Filter output (e.g., `dangling=true`) |
| `--format` | Format output with a Go template |

```bash
docker images
docker images -a
docker images --filter "dangling=true"
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
```

### `docker pull`

Downloads an image from a registry.

```bash
docker pull nginx:1.25-alpine
docker pull --platform linux/arm64 nginx:alpine
```

### `docker tag`

Creates an additional tag pointing to the same image layers.

```bash
docker tag myapp:1.0 myorg/myapp:1.0
docker tag myapp:1.0 myapp:latest
docker tag myapp:1.0 registry.example.com/myapp:1.0
```

### `docker image inspect`

Returns low-level JSON details about an image.

```bash
docker image inspect nginx:alpine
docker image inspect nginx:alpine --format '{{.Config.Cmd}}'
```

### `docker image history`

Shows the build history (layers) of an image.

```bash
docker image history myapp:1.0
docker image history --no-trunc myapp:1.0
```

### `docker save` / `docker load`

Export and import images as tar archives (useful for air-gapped environments).

```bash
docker save myapp:1.0 | gzip > myapp-1.0.tar.gz
docker load < myapp-1.0.tar.gz
```

---

## Containers

A container is a running instance of an image — it adds a writable layer on top of the read-only image layers.

### `docker run`

Creates and starts a container from an image. This is the single most important Docker command.

| Flag / Option | Description |
|---|---|
| `-d` | Run in detached (background) mode |
| `-it` | Interactive terminal (attach stdin + pseudo-TTY) |
| `--name <name>` | Assign a name to the container |
| `-p <host>:<container>` | Publish container port to host |
| `-P` | Publish all exposed ports to random host ports |
| `-v <host>:<container>` | Bind-mount a host path |
| `-e <KEY=val>` | Set environment variable |
| `--env-file <file>` | Load environment variables from a file |
| `--network <name>` | Connect to a specific network |
| `--rm` | Automatically remove container on exit |
| `--restart <policy>` | Restart policy: `no`, `always`, `on-failure`, `unless-stopped` |
| `--memory <limit>` | Memory limit (e.g., `512m`) |
| `--cpus <n>` | Limit CPU count |
| `-u <user>` | Run as a specific user |
| `--read-only` | Mount the root filesystem as read-only |
| `--cap-drop ALL` | Drop all Linux capabilities |
| `--entrypoint <cmd>` | Override the default entrypoint |

```bash
# Run Nginx in background, publish port 80
docker run -d --name web -p 8080:80 --restart unless-stopped nginx:alpine

# Interactive Ubuntu shell; auto-remove on exit
docker run -it --rm ubuntu:22.04 bash

# App container with environment and volume
docker run -d \
  --name api \
  -p 3000:3000 \
  -e NODE_ENV=production \
  --env-file .env \
  -v $(pwd)/logs:/app/logs \
  --memory 512m \
  --cpus 1.5 \
  myapp:1.0
```

### `docker ps`

Lists containers.

| Flag / Option | Description |
|---|---|
| `-a` | Show all containers (including stopped) |
| `-q` | Display only IDs |
| `--filter <key=val>` | Filter containers (e.g., `status=running`) |
| `--format` | Go template formatting |
| `-s` | Display total file sizes |

```bash
docker ps
docker ps -a
docker ps --filter "status=exited" -q
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

### `docker exec`

Runs a command inside a running container.

| Flag / Option | Description |
|---|---|
| `-it` | Interactive terminal |
| `-e <KEY=val>` | Set environment variable for this exec |
| `-u <user>` | Run as specified user |
| `-d` | Detached mode |

```bash
docker exec -it web bash                        # Open shell
docker exec -it web sh                          # Alpine / minimal images
docker exec web cat /etc/nginx/nginx.conf       # One-off command
docker exec -it -u root web bash                # As root
docker exec -e DEBUG=1 api node -e "require('./debug')"
```

### `docker stop` / `docker start` / `docker restart`

```bash
docker stop web                         # Graceful stop (SIGTERM, then SIGKILL after 10s)
docker stop -t 30 web                   # Give 30s grace period
docker start web                        # Start stopped container
docker restart web                      # Stop then start
```

### `docker rm`

Removes one or more stopped containers.

```bash
docker rm web
docker rm -f web                        # Force remove running container
docker rm $(docker ps -aq -f status=exited)  # Remove all exited containers
```

### `docker logs`

Fetches logs from a container.

| Flag / Option | Description |
|---|---|
| `-f` | Follow log output (stream) |
| `--tail <n>` | Show last `n` lines |
| `--since <time>` | Show logs since timestamp or duration (e.g., `1h`) |
| `-t` | Show timestamps |

```bash
docker logs web
docker logs -f --tail 100 web
docker logs --since 30m api
docker logs -t api
```

### `docker inspect`

Returns detailed JSON metadata about a container or image.

```bash
docker inspect web
docker inspect web --format '{{.State.Status}}'
docker inspect web --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
```

### `docker cp`

Copies files between a container and the local filesystem.

```bash
docker cp web:/etc/nginx/nginx.conf ./nginx.conf
docker cp ./nginx.conf web:/etc/nginx/nginx.conf
```

### `docker stats`

Displays live resource usage statistics for containers.

```bash
docker stats
docker stats --no-stream                # Single snapshot, no live update
docker stats web api                    # Specific containers
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

### `docker top`

Shows the running processes inside a container.

```bash
docker top web
docker top web aux
```

### `docker diff`

Shows filesystem changes made to a container since it started.

```bash
docker diff web           # A=added, C=changed, D=deleted
```

### `docker commit`

Creates a new image from a container's current state. Prefer Dockerfiles for reproducibility; use `commit` for quick debugging snapshots.

```bash
docker commit web myapp:debug-snapshot
docker commit -m "add curl" -a "engineer" web myapp:with-curl
```

---

## Networking

Docker provides several built-in network drivers and lets you create isolated networks to control container communication.

### Network Drivers

| Driver | Use Case |
|---|---|
| `bridge` | Default for standalone containers; containers communicate via virtual bridge |
| `host` | Container shares the host's network namespace |
| `none` | No networking |
| `overlay` | Multi-host networking for Docker Swarm / distributed setups |
| `macvlan` | Assign a MAC address; container appears as a physical device on the LAN |

### `docker network`

| Subcommand | Description |
|---|---|
| `ls` | List networks |
| `create` | Create a network |
| `inspect` | Show network details |
| `connect` | Connect a container to a network |
| `disconnect` | Disconnect a container from a network |
| `rm` | Remove a network |
| `prune` | Remove all unused networks |

```bash
docker network ls
docker network create --driver bridge app-net
docker network create --subnet 192.168.100.0/24 custom-net

# Run container attached to custom network
docker run -d --name api --network app-net myapp:1.0

# Connect existing container to another network
docker network connect app-net web

# Inspect to find container IPs
docker network inspect app-net

# DNS-based communication: containers on same network resolve each other by name
docker run -it --rm --network app-net alpine ping api
```

### DNS in User-Defined Networks

Containers in a user-defined bridge network can resolve each other by container name automatically. The default bridge network does NOT support this — always create named networks for multi-container apps.

---

## Volumes & Storage

Volumes persist data beyond the container lifecycle and are the recommended mechanism for sharing data between containers or with the host.

### Storage Options

| Type | Description |
|---|---|
| **Volume** | Managed by Docker; stored in `/var/lib/docker/volumes/` |
| **Bind Mount** | Maps a host path directly into the container |
| **tmpfs Mount** | In-memory; not persisted to disk |

### `docker volume`

| Subcommand | Description |
|---|---|
| `create` | Create a named volume |
| `ls` | List volumes |
| `inspect` | Show volume details |
| `rm` | Remove a volume |
| `prune` | Remove all unused volumes |

```bash
docker volume create pgdata
docker volume ls
docker volume inspect pgdata
docker volume rm pgdata
docker volume prune -f                              # Remove all unused volumes

# Named volume in run
docker run -d --name db \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:16-alpine

# Bind mount (host path)
docker run -d --name web \
  -v $(pwd)/html:/usr/share/nginx/html:ro \
  nginx:alpine

# Read-only volume
docker run -d -v config-vol:/etc/app:ro myapp:1.0

# tmpfs for sensitive in-memory data
docker run -d --tmpfs /run:rw,noexec,nosuid,size=64m myapp:1.0
```

---

## Dockerfile Best Practices

A Dockerfile is a text document containing instructions that assemble a Docker image. Well-written Dockerfiles produce small, secure, cacheable images.

```dockerfile
# ── Stage 1: Builder ─────────────────────────────────────────────
FROM node:20-alpine AS builder

# Set non-root working directory
WORKDIR /app

# Copy dependency manifests first to leverage layer cache
COPY package*.json ./
RUN npm ci --omit=dev

# Copy source after dependencies (cache bust only on src changes)
COPY . .
RUN npm run build

# ── Stage 2: Production image ─────────────────────────────────────
FROM node:20-alpine AS production

# Create non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy only build output and runtime deps from builder
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules

# Drop to non-root user
USER appuser

# Document the port the application uses
EXPOSE 3000

# Use exec form to receive signals properly
ENTRYPOINT ["node"]
CMD ["dist/server.js"]

# Build metadata labels (OCI standard)
LABEL org.opencontainers.image.title="My App"
LABEL org.opencontainers.image.version="1.0.0"
```

### Key Dockerfile Instructions

| Instruction | Description |
|---|---|
| `FROM` | Base image. Always pin to a specific digest or version tag |
| `RUN` | Execute a shell command during build |
| `COPY` | Copy files from build context into the image |
| `ADD` | Like COPY but also handles URLs and tar extraction (prefer COPY) |
| `ENV` | Set environment variables persisted into the image |
| `ARG` | Build-time variable; not persisted in the final image |
| `EXPOSE` | Document which ports the container listens on (informational) |
| `WORKDIR` | Set the working directory for subsequent instructions |
| `USER` | Switch to a different user |
| `ENTRYPOINT` | Configure the executable (prefer exec form `["cmd"]`) |
| `CMD` | Default arguments to ENTRYPOINT; overridable at runtime |
| `HEALTHCHECK` | Define a command to test container health |
| `ONBUILD` | Trigger instructions in child images |
| `STOPSIGNAL` | Override the stop signal sent to the container |

```dockerfile
# HEALTHCHECK example
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1
```

### `.dockerignore`

Exclude files from the build context to reduce image size and prevent leaking secrets.

```
.git
.gitignore
node_modules
*.log
.env
.env.*
coverage/
dist/
*.md
Dockerfile*
docker-compose*
```

---

## Docker Compose

Docker Compose defines and runs multi-container applications using a YAML file. Compose V2 (`docker compose`) is built into the Docker CLI; V1 (`docker-compose`) is deprecated.

### `docker compose` Key Commands

| Command | Description |
|---|---|
| `up` | Create and start all services |
| `up -d` | Detached mode |
| `up --build` | Rebuild images before starting |
| `down` | Stop and remove containers and networks |
| `down -v` | Also remove named volumes |
| `ps` | List service containers |
| `logs` | View output from services |
| `exec` | Run a command in a running service container |
| `run` | Run a one-off command on a service |
| `build` | Build or rebuild service images |
| `pull` | Pull service images |
| `config` | Validate and display the merged config |
| `scale` | Scale services (or use `--scale` with `up`) |

```bash
docker compose up -d
docker compose up -d --build
docker compose down -v
docker compose logs -f api
docker compose exec api bash
docker compose run --rm api npm run migrate
docker compose config                   # Validate config
docker compose up --scale worker=3     # Run 3 worker replicas
```

### Example `docker-compose.yml`

```yaml
version: "3.9"

services:
  api:
    build:
      context: .
      target: production
    image: myapp/api:latest
    container_name: api
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
      DATABASE_URL: postgres://user:secret@db:5432/appdb
    env_file:
      - .env
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app-net
    volumes:
      - logs:/app/logs

  db:
    image: postgres:16-alpine
    container_name: db
    restart: unless-stopped
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - app-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    networks:
      - app-net

volumes:
  pgdata:
  logs:

networks:
  app-net:
    driver: bridge
```

---

## Registry & Distribution

### `docker login` / `docker logout`

```bash
docker login                                    # Docker Hub
docker login registry.example.com              # Private registry
docker login -u $CI_USER -p $CI_TOKEN ghcr.io  # GitHub Container Registry
docker logout registry.example.com
```

### `docker push`

Uploads a tagged image to a registry.

```bash
docker tag myapp:1.0 myorg/myapp:1.0
docker push myorg/myapp:1.0
docker push myorg/myapp:latest
```

### `docker search`

Searches Docker Hub for public images.

```bash
docker search nginx
docker search --filter stars=100 node
docker search --filter is-official=true python
```

### Running a Private Registry

```bash
docker run -d \
  --name registry \
  -p 5000:5000 \
  --restart always \
  -v registry-data:/var/lib/registry \
  registry:2

# Push to local registry
docker tag myapp:1.0 localhost:5000/myapp:1.0
docker push localhost:5000/myapp:1.0
docker pull localhost:5000/myapp:1.0
```

---

## Logging & Monitoring

### Logging Drivers

| Driver | Description |
|---|---|
| `json-file` | Default; logs stored as JSON on disk |
| `syslog` | Forward to syslog daemon |
| `journald` | Forward to systemd journal |
| `gelf` | Graylog Extended Log Format |
| `awslogs` | Amazon CloudWatch Logs |
| `fluentd` | Forward to Fluentd / Fluent Bit |
| `splunk` | Forward to Splunk HTTP Event Collector |
| `none` | Disable logging |

```bash
# Set logging driver globally in /etc/docker/daemon.json
# { "log-driver": "json-file", "log-opts": { "max-size": "10m", "max-file": "3" } }

# Per-container logging
docker run -d \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=5 \
  nginx:alpine

# Forward to syslog
docker run -d \
  --log-driver syslog \
  --log-opt syslog-address=udp://logs.example.com:514 \
  myapp:1.0
```

### `docker events`

Streams real-time events from the Docker daemon.

```bash
docker events
docker events --filter event=start
docker events --filter container=web
docker events --since "2024-01-01T00:00:00"
```

---

## Resource Management

### CPU and Memory Limits

```bash
# Hard memory limit (OOM kill if exceeded)
docker run -d --memory 512m --memory-swap 1g myapp:1.0

# CPU limits
docker run -d --cpus 1.5 myapp:1.0                   # 1.5 CPU cores
docker run -d --cpu-shares 512 myapp:1.0              # Relative weight (default 1024)
docker run -d --cpuset-cpus "0,1" myapp:1.0           # Pin to CPUs 0 and 1

# Soft memory reservation (for scheduling hints)
docker run -d --memory-reservation 256m myapp:1.0

# View live stats
docker stats --no-stream
```

### Updating Limits on a Running Container

```bash
docker update --memory 1g --cpus 2 web
```

---

## Security

### Running as Non-Root

```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

```bash
docker run -u 1000:1000 myapp:1.0
```

### Capabilities

```bash
# Drop all capabilities and add only what's needed
docker run -d \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  nginx:alpine
```

### Read-Only Root Filesystem

```bash
docker run -d \
  --read-only \
  --tmpfs /tmp \
  --tmpfs /run \
  myapp:1.0
```

### No New Privileges

```bash
docker run -d --security-opt no-new-privileges myapp:1.0
```

### Scanning Images for Vulnerabilities

```bash
# Docker Scout (built-in, replaces Snyk)
docker scout cves myapp:1.0
docker scout recommendations myapp:1.0

# Trivy (open-source)
trivy image myapp:1.0
```

### Secrets (Docker Swarm / Compose)

```bash
# Create a secret
echo "supersecret" | docker secret create db_password -

# Use in Compose
# secrets:
#   db_password:
#     external: true
# services:
#   api:
#     secrets:
#       - db_password
```

---

## Cleanup

Docker accumulates unused images, containers, volumes, and networks over time. Regular cleanup prevents disk exhaustion.

### `docker system prune`

Removes all stopped containers, dangling images, unused networks, and optionally volumes.

| Flag / Option | Description |
|---|---|
| `-a` / `--all` | Also remove images not referenced by any container |
| `--volumes` | Also remove unused volumes |
| `-f` / `--force` | Skip confirmation prompt |

```bash
docker system prune                     # Safe: removes dangling resources
docker system prune -a                  # Aggressive: all unused images too
docker system prune -a --volumes -f     # Nuclear option: everything unused

# Show disk usage before pruning
docker system df
docker system df -v                     # Verbose breakdown
```

### Targeted Cleanup

```bash
# Remove all stopped containers
docker container prune -f

# Remove dangling images (no tag, not referenced)
docker image prune -f

# Remove all unused images
docker image prune -a -f

# Remove unused volumes
docker volume prune -f

# Remove unused networks
docker network prune -f

# Force-remove all containers (running and stopped)
docker rm -f $(docker ps -aq)

# Remove all images
docker rmi $(docker images -q)
```

---

## Common Patterns & Tips

1. **Always pin image tags in production.** `nginx:latest` can silently introduce breaking changes. Use `nginx:1.25.3-alpine` or pin to a digest (`nginx@sha256:abc123...`) for reproducible builds.

2. **Use multi-stage builds to minimise image size.** Separate build-time tools (compilers, test frameworks) from the runtime image. A Go binary compiled in a builder stage can be copied into a `scratch` or `distroless` image, resulting in images under 20 MB.

3. **Put `COPY` and `RUN` instructions that change frequently at the bottom of the Dockerfile.** Docker caches layers in order; placing stable instructions (e.g., installing OS packages) before frequently-changing ones (e.g., copying source code) maximises cache reuse and speeds up builds dramatically.

4. **Use health checks in production containers.** Define `HEALTHCHECK` in your Dockerfile or `healthcheck` in Compose so that orchestrators (Swarm, Kubernetes) can detect unhealthy containers and restart or replace them automatically.

5. **Never store secrets in environment variables in images.** Secrets passed via `-e` or `ENV` are visible in `docker inspect` and the image history. Use Docker secrets, a vault solution, or runtime injection (e.g., AWS Secrets Manager with an init container).

6. **Use `.dockerignore` religiously.** Failing to exclude `node_modules`, `.git`, and local build artefacts inflates the build context, slowing every build and potentially leaking sensitive files into the image.

7. **Prefer user-defined bridge networks over the default bridge.** Containers on user-defined networks can resolve each other by name (DNS). The default bridge network lacks this feature and should only be used for quick experiments.

8. **Set memory and CPU limits on all production containers.** Without limits, a runaway container can starve the host. Setting `--memory` and `--cpus` provides predictable performance and prevents cascading failures in co-located services.

9. **Use `docker compose` override files for environment-specific configuration.** Keep `docker-compose.yml` as the base and provide `docker-compose.override.yml` for local development overrides (e.g., bind mounts, debug ports) that are never committed to CI.

10. **Combine `RUN` commands with `&&` and clean caches in the same layer.** Every `RUN` instruction creates a layer; combining multiple commands in one and removing package manager caches (e.g., `rm -rf /var/cache/apk/*`) keeps the layer as small as possible.

11. **Use `--restart unless-stopped` instead of `always`.** The `always` policy restarts containers even after a manual `docker stop`. `unless-stopped` respects intentional stops while still recovering from crashes and reboots.

12. **Run `docker system df` regularly in CI/CD agents.** Build agents accumulate images and volumes rapidly. Integrate `docker system prune -a -f` at the end of CI pipelines to prevent agents from filling their disks.
