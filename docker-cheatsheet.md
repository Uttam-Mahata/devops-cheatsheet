# Docker Cheatsheet

> A comprehensive reference for Docker — from container basics to multi-service orchestration.

---

## Table of Contents

- [Installation & Setup](#installation--setup)
- [Images](#images)
- [Containers](#containers)
- [Networking](#networking)
- [Volumes & Storage](#volumes--storage)
- [Dockerfile](#dockerfile)
- [Docker Compose](#docker-compose)
- [Registry & Distribution](#registry--distribution)
- [Logging & Monitoring](#logging--monitoring)
- [Resource Management](#resource-management)
- [Security](#security)
- [Docker System & Cleanup](#docker-system--cleanup)
- [Advanced Scenarios](#advanced-scenarios)

---

## Installation & Setup

```bash
# Check Docker version
docker --version
docker version          # full client + server info

# Check system-wide info
docker info

# Run hello-world to verify installation
docker run hello-world

# Add current user to docker group (avoid sudo)
sudo usermod -aG docker $USER
newgrp docker
```

---

## Images

### Pulling & Listing

```bash
# Pull an image from Docker Hub
docker pull nginx

# Pull a specific tag
docker pull node:20-alpine

# Pull from a private registry
docker pull registry.example.com/myapp:1.0.0

# List local images
docker images
docker image ls

# List with filter
docker images --filter "dangling=true"

# Inspect image metadata
docker image inspect nginx

# Show image layers/history
docker image history nginx
```

### Building

```bash
# Build from Dockerfile in current directory
docker build -t myapp:latest .

# Build with a specific Dockerfile
docker build -f Dockerfile.prod -t myapp:prod .

# Build with build arguments
docker build --build-arg ENV=production -t myapp:prod .

# Build with no cache
docker build --no-cache -t myapp:latest .

# Multi-platform build (requires buildx)
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest --push .
```

### Tagging & Removing

```bash
# Tag an existing image
docker tag myapp:latest myapp:1.2.0

# Remove an image
docker rmi myapp:latest

# Force remove (even if container uses it)
docker rmi -f myapp:latest

# Remove all dangling images
docker image prune

# Remove all unused images
docker image prune -a
```

---

## Containers

### Running Containers

```bash
# Run a container (foreground)
docker run nginx

# Run in detached (background) mode
docker run -d nginx

# Run with a name
docker run -d --name my-nginx nginx

# Run with port mapping  host:container
docker run -d -p 8080:80 nginx

# Run with multiple ports
docker run -d -p 8080:80 -p 8443:443 nginx

# Run with environment variables
docker run -d -e NODE_ENV=production -e PORT=3000 myapp

# Run with env file
docker run -d --env-file .env myapp

# Run interactively with terminal
docker run -it ubuntu bash

# Run and remove container after exit
docker run --rm alpine echo "hello"

# Run with volume mount
docker run -d -v /host/data:/app/data nginx

# Run with resource limits
docker run -d --memory="256m" --cpus="0.5" myapp
```

### Managing Containers

```bash
# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# Start / stop / restart
docker start my-nginx
docker stop my-nginx
docker restart my-nginx

# Pause / unpause
docker pause my-nginx
docker unpause my-nginx

# Kill (sends SIGKILL)
docker kill my-nginx

# Remove a stopped container
docker rm my-nginx

# Force remove a running container
docker rm -f my-nginx

# Remove all stopped containers
docker container prune
```

### Interacting with Containers

```bash
# Execute a command in a running container
docker exec my-nginx ls /etc/nginx

# Open interactive shell in running container
docker exec -it my-nginx bash
docker exec -it my-nginx sh          # for alpine-based images

# Copy files to/from container
docker cp ./config.json my-nginx:/etc/nginx/config.json
docker cp my-nginx:/var/log/nginx/error.log ./error.log

# View container logs
docker logs my-nginx

# Stream logs in real time
docker logs -f my-nginx

# Last 100 lines of logs
docker logs --tail 100 my-nginx

# Logs with timestamps
docker logs -t my-nginx

# Inspect container details (IP, mounts, env, etc.)
docker inspect my-nginx

# Get container IP address
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my-nginx

# View live resource usage
docker stats

# View stats for a specific container
docker stats my-nginx

# View running processes inside a container
docker top my-nginx
```

---

## Networking

```bash
# List networks
docker network ls

# Create a bridge network
docker network create my-network

# Create with custom subnet
docker network create --subnet=192.168.1.0/24 my-network

# Connect a running container to a network
docker network connect my-network my-nginx

# Disconnect a container from a network
docker network disconnect my-network my-nginx

# Inspect a network
docker network inspect my-network

# Remove a network
docker network rm my-network

# Remove all unused networks
docker network prune

# Run container on a specific network
docker run -d --network my-network --name api myapp

# Run container with a custom hostname
docker run -d --network my-network --hostname api-server myapp
```

### Network Types

| Driver    | Use Case                                      |
|-----------|-----------------------------------------------|
| `bridge`  | Default. Isolated network on a single host    |
| `host`    | Container shares host's network stack         |
| `none`    | No networking (fully isolated)                |
| `overlay` | Multi-host networking (Docker Swarm)          |
| `macvlan` | Assign a MAC address to the container         |

---

## Volumes & Storage

```bash
# Create a named volume
docker volume create my-data

# List volumes
docker volume ls

# Inspect a volume
docker volume inspect my-data

# Remove a volume
docker volume rm my-data

# Remove all unused volumes
docker volume prune

# Mount a named volume into a container
docker run -d -v my-data:/app/data myapp

# Bind mount a host directory
docker run -d -v $(pwd)/data:/app/data myapp

# Read-only bind mount
docker run -d -v $(pwd)/config:/app/config:ro myapp

# tmpfs mount (in-memory, not persisted)
docker run -d --tmpfs /tmp myapp
```

### Volume vs Bind Mount

| Feature            | Named Volume          | Bind Mount                  |
|--------------------|-----------------------|-----------------------------|
| Managed by Docker  | Yes                   | No                          |
| Portable           | Yes                   | No (host path dependent)    |
| Performance        | Optimized             | Depends on OS               |
| Best for           | App data, databases   | Dev hot-reload, config files|

---

## Dockerfile

```dockerfile
# ── Base image ──────────────────────────────────────────────
FROM node:20-alpine

# ── Metadata ────────────────────────────────────────────────
LABEL maintainer="jane@example.com"
LABEL version="1.0.0"

# ── Build arguments (available during build only) ───────────
ARG NODE_ENV=production

# ── Environment variables (available at runtime too) ────────
ENV PORT=3000 \
    NODE_ENV=${NODE_ENV}

# ── Set working directory ───────────────────────────────────
WORKDIR /app

# ── Copy dependency files first (layer cache optimization) ──
COPY package*.json ./

# ── Install dependencies ────────────────────────────────────
RUN npm ci --only=production

# ── Copy application source ─────────────────────────────────
COPY . .

# ── Expose port ─────────────────────────────────────────────
EXPOSE 3000

# ── Non-root user (security best practice) ──────────────────
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# ── Health check ────────────────────────────────────────────
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

# ── Start command ────────────────────────────────────────────
CMD ["node", "server.js"]
```

### Multi-Stage Build

```dockerfile
# ── Stage 1: Build ──────────────────────────────────────────
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# ── Stage 2: Production ─────────────────────────────────────
FROM node:20-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json ./
RUN npm ci --only=production
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

### .dockerignore

```
node_modules
npm-debug.log
.git
.gitignore
.env
*.md
dist
coverage
.DS_Store
```

### Dockerfile Instruction Reference

| Instruction  | Purpose                                              |
|--------------|------------------------------------------------------|
| `FROM`       | Base image                                           |
| `RUN`        | Execute command during build (creates a layer)       |
| `CMD`        | Default command when container starts (overridable)  |
| `ENTRYPOINT` | Fixed command — CMD becomes its arguments            |
| `COPY`       | Copy files from host into image                      |
| `ADD`        | Like COPY but supports URLs and auto-extracts tar    |
| `ENV`        | Set environment variable (build + runtime)           |
| `ARG`        | Build-time variable (not persisted in image)         |
| `EXPOSE`     | Documents the port (does not publish it)             |
| `VOLUME`     | Creates a mount point                                |
| `WORKDIR`    | Set working directory                                |
| `USER`       | Switch to a user/UID                                 |
| `HEALTHCHECK`| Define a container health check                      |
| `LABEL`      | Add metadata key-value pairs                         |

---

## Docker Compose

### `docker-compose.yml` Example

```yaml
version: "3.9"

services:

  # ── API Service ───────────────────────────────────────────
  api:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        NODE_ENV: production
    image: myapp-api:latest
    container_name: myapp-api
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
    env_file:
      - .env
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - uploads:/app/uploads
    networks:
      - backend
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  # ── Database ──────────────────────────────────────────────
  db:
    image: postgres:16-alpine
    container_name: myapp-db
    restart: unless-stopped
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ── Cache ─────────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    container_name: myapp-redis
    restart: unless-stopped
    networks:
      - backend

  # ── Reverse Proxy ─────────────────────────────────────────
  nginx:
    image: nginx:alpine
    container_name: myapp-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - certdata:/etc/letsencrypt
    depends_on:
      - api
    networks:
      - frontend
      - backend

volumes:
  pgdata:
  uploads:
  certdata:

networks:
  frontend:
  backend:
```

### Compose Commands

```bash
# Start all services (build if needed)
docker compose up

# Start in detached mode
docker compose up -d

# Build images before starting
docker compose up --build

# Scale a specific service
docker compose up -d --scale api=3

# Stop all services
docker compose down

# Stop and remove volumes
docker compose down -v

# Stop and remove images too
docker compose down --rmi all

# View running services
docker compose ps

# View logs for all services
docker compose logs

# Stream logs for a specific service
docker compose logs -f api

# Execute command in a running service
docker compose exec api sh

# Run a one-off command (new container)
docker compose run --rm api node scripts/seed.js

# Restart a single service
docker compose restart api

# Pull latest images
docker compose pull

# Validate compose file
docker compose config
```

---

## Registry & Distribution

```bash
# Login to Docker Hub
docker login

# Login to a private registry
docker login registry.example.com

# Tag image for a registry
docker tag myapp:latest registry.example.com/team/myapp:1.0.0

# Push image to registry
docker push registry.example.com/team/myapp:1.0.0

# Pull from private registry
docker pull registry.example.com/team/myapp:1.0.0

# Logout
docker logout registry.example.com

# Save image to a tar file (air-gapped transfers)
docker save -o myapp.tar myapp:latest

# Load image from tar
docker load -i myapp.tar

# Export container filesystem (not image layers)
docker export my-container -o container.tar

# Import as a new image
docker import container.tar myapp:imported
```

---

## Logging & Monitoring

```bash
# View logs
docker logs my-nginx

# Follow logs
docker logs -f my-nginx

# Tail last N lines
docker logs --tail 50 my-nginx

# Logs since a timestamp
docker logs --since "2024-01-01T00:00:00" my-nginx

# Logs between timestamps
docker logs --since "1h" --until "30m" my-nginx

# Live resource stats (all containers)
docker stats

# Stats snapshot (no stream)
docker stats --no-stream

# Format stats output
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"

# View container events in real time
docker events

# Filter events by container
docker events --filter container=my-nginx
```

### Logging Drivers

```bash
# Set logging driver at run time
docker run -d --log-driver json-file --log-opt max-size=10m --log-opt max-file=3 nginx

# Set default in /etc/docker/daemon.json
# {
#   "log-driver": "json-file",
#   "log-opts": { "max-size": "10m", "max-file": "3" }
# }
```

---

## Resource Management

```bash
# Limit memory
docker run -d --memory="512m" myapp

# Limit memory + swap (--memory-swap = memory + swap total)
docker run -d --memory="512m" --memory-swap="1g" myapp

# Limit CPU shares (relative weight)
docker run -d --cpu-shares=512 myapp

# Limit to specific CPUs
docker run -d --cpuset-cpus="0,1" myapp

# Limit CPU quota (50% of one core)
docker run -d --cpus="0.5" myapp

# Update resource limits on a running container
docker update --memory="1g" --cpus="1.0" my-nginx

# View container resource usage
docker stats my-nginx --no-stream
```

---

## Security

```bash
# Run as a non-root user
docker run -d --user 1001:1001 myapp

# Drop all capabilities, add only what's needed
docker run -d --cap-drop ALL --cap-add NET_BIND_SERVICE myapp

# Read-only root filesystem
docker run -d --read-only myapp

# Disable privilege escalation
docker run -d --security-opt no-new-privileges myapp

# Run with a seccomp profile
docker run -d --security-opt seccomp=/path/to/profile.json myapp

# Scan image for vulnerabilities (Docker Scout)
docker scout cves myapp:latest

# Quick scan summary
docker scout quickview myapp:latest

# Verify image signature (Docker Content Trust)
export DOCKER_CONTENT_TRUST=1
docker pull nginx
```

---

## Docker System & Cleanup

```bash
# Show disk usage
docker system df

# Detailed disk usage
docker system df -v

# Remove all stopped containers
docker container prune

# Remove all unused images
docker image prune -a

# Remove all unused volumes
docker volume prune

# Remove all unused networks
docker network prune

# Remove everything unused at once (DESTRUCTIVE)
docker system prune

# Remove everything including volumes (VERY DESTRUCTIVE)
docker system prune -a --volumes

# Remove all containers (running + stopped)
docker rm -f $(docker ps -aq)

# Remove all images
docker rmi -f $(docker images -q)
```

---

## Advanced Scenarios

### Scenario 1 — Debug a container that keeps crashing

```bash
# Override the entrypoint to drop into a shell
docker run -it --entrypoint sh myapp:latest

# Or run from last known good image state
docker run -it --entrypoint sh myapp:latest ls /app
```

### Scenario 2 — Copy a database dump out of a running container

```bash
docker exec db pg_dump -U user mydb > backup.sql
# or
docker exec db pg_dump -U user mydb | gzip > backup.sql.gz
```

### Scenario 3 — Connect two Compose projects on the same network

```bash
# In project-a's docker-compose.yml, declare an external network:
networks:
  shared:
    name: shared-net

# In project-b's docker-compose.yml:
networks:
  shared:
    external: true
    name: shared-net
```

### Scenario 4 — Hot-reload during development

```bash
# Mount source code as a bind mount so changes reflect immediately
docker run -d \
  -v $(pwd):/app \
  -v /app/node_modules \       # anonymous volume to protect node_modules
  -p 3000:3000 \
  -e NODE_ENV=development \
  myapp:dev
```

### Scenario 5 — Run a one-off task with Compose (e.g., DB migrations)

```bash
docker compose run --rm api npx prisma migrate deploy
docker compose run --rm api python manage.py migrate
```

### Scenario 6 — Inspect a stopped container's filesystem

```bash
docker commit stopped-container debug-image
docker run -it debug-image sh
```

### Scenario 7 — Limit log size to prevent disk exhaustion

```bash
# docker-compose.yml
services:
  api:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "5"
```

### Scenario 8 — Pass secrets without baking them into the image

```bash
# Using --secret (BuildKit)
docker build --secret id=mysecret,src=./secret.txt -t myapp .

# In Dockerfile:
# RUN --mount=type=secret,id=mysecret cat /run/secrets/mysecret

# At runtime via environment file
docker run --env-file .env myapp
```

### Scenario 9 — Multi-stage build to shrink final image size

```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o server .

FROM scratch                        # zero-byte base image
COPY --from=builder /app/server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
```

### Scenario 10 — Health-check dependent service startup in Compose

```yaml
services:
  api:
    depends_on:
      db:
        condition: service_healthy   # waits for DB health check to pass
  db:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      timeout: 3s
      retries: 10
```

---

## Quick Reference Card

| Task                              | Command                                               |
|-----------------------------------|-------------------------------------------------------|
| Pull & run                        | `docker run -d -p 8080:80 nginx`                      |
| Shell into container              | `docker exec -it <name> sh`                           |
| View logs                         | `docker logs -f <name>`                               |
| Stop all containers               | `docker stop $(docker ps -q)`                         |
| Remove all stopped containers     | `docker container prune`                              |
| Build image                       | `docker build -t myapp:latest .`                      |
| Compose up with rebuild           | `docker compose up -d --build`                        |
| Compose one-off task              | `docker compose run --rm api <cmd>`                   |
| Check disk usage                  | `docker system df`                                    |
| Full cleanup (careful!)           | `docker system prune -a --volumes`                    |

---

> Containers are ephemeral — your data should not be. Always use volumes for anything worth keeping.
