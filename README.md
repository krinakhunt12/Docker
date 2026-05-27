<img width="445" height="105" alt="image" src="https://github.com/user-attachments/assets/9433ee7e-3a0c-4bbe-8b73-598db21dd281" /># 🐳 Docker — Complete Knowledge Reference
---

## 📋 Table of Contents
1. [What is Docker?](#1-what-is-docker)
2. [Core Concepts](#2-core-concepts)
3. [Dockerfile Deep Dive](#3-dockerfile-deep-dive)
4. [Building Docker Images](#4-building-docker-images)
5. [Running Containers](#5-running-containers)
6. [Port Mapping Explained](#6-port-mapping-explained)
7. [Docker Layer Caching](#7-docker-layer-caching)
8. [Inspecting Containers](#8-inspecting-containers)
9. [Docker Image Management](#9-docker-image-management)
10. [Google Artifact Registry](#10-google-artifact-registry)
11. [Complete Command Reference](#11-complete-command-reference)
12. [Production Best Practices](#12-production-best-practices)

---

## 1. What is Docker?

Docker is an open-source platform for **building, shipping, and running applications inside containers**. A container packages your application code along with all its dependencies, libraries, and configuration into a single, portable unit.

<img width="445" height="105" alt="image" src="https://github.com/user-attachments/assets/10c8a4c6-5668-491d-b5e7-a54658d51fd7" />


### Container vs Virtual Machine

| Feature | Container | Virtual Machine |
|---|---|---|
| OS | Shares host kernel | Full OS per VM |
| Size | MBs (e.g., node-app: 1.62 GB with runtime) | GBs (typically 10–40 GB) |
| Startup time | Milliseconds | Minutes |
| Isolation | Process-level | Hardware-level |
| Portability | Run anywhere Docker runs | Hypervisor-specific |

---

## 2. Core Concepts

### Image
A **read-only template** used to create containers. Built from a Dockerfile. Images are composed of **layers** — each instruction in the Dockerfile adds a new layer.

```
docker.io/library/node:lts    ← Base image (Layer 1)
         WORKDIR /app          ← Layer 2
         ADD . /app            ← Layer 3
```

### Container
A **running instance** of an image. Containers are isolated from each other and the host, but share the host OS kernel.

### Dockerfile
A text file containing instructions to build a Docker image. Think of it as a recipe for your container.

### Registry
A storage system for Docker images. Examples:
- **Docker Hub** — public registry (`docker.io/library/node`)
- **Google Artifact Registry** — GCP's managed private registry
- **Amazon ECR** — AWS equivalent

### Tag
A label applied to an image to distinguish versions. Format: `name:tag`
- `node-app:0.1` — version 0.1
- `node-app:0.2` — version 0.2
- `node-app:latest` — default if no tag specified (avoid in production!)

---

## 3. Dockerfile Deep Dive

The Dockerfile used in this lab (Node.js app):

```dockerfile
# Step 1: Base image — pulls node:lts from Docker Hub
FROM node:lts

# Step 2: Set working directory inside the container
WORKDIR /app

# Step 3: Copy all files from host . into container /app
ADD . /app

# Step 4: Expose the port the app listens on (documentation only)
EXPOSE 8080

# Step 5: Command to run when container starts
CMD ["node", "app.js"]
```

### Dockerfile Instruction Reference

| Instruction | Purpose | Example |
|---|---|---|
| `FROM` | Base image | `FROM node:lts` |
| `WORKDIR` | Set working directory | `WORKDIR /app` |
| `COPY` | Copy files (preferred over ADD) | `COPY . /app` |
| `ADD` | Copy + extract archives | `ADD app.tar.gz /app` |
| `RUN` | Execute commands during build | `RUN npm install` |
| `ENV` | Set environment variables | `ENV NODE_ENV=production` |
| `EXPOSE` | Document port (does NOT publish) | `EXPOSE 8080` |
| `CMD` | Default command on run | `CMD ["node", "app.js"]` |
| `ENTRYPOINT` | Fixed command on run | `ENTRYPOINT ["node"]` |
| `ARG` | Build-time variable | `ARG VERSION=1.0` |
| `LABEL` | Metadata | `LABEL version="1.0"` |

### CMD vs ENTRYPOINT

```dockerfile
# CMD — can be overridden at docker run
CMD ["node", "app.js"]
# Override: docker run myapp python script.py

# ENTRYPOINT — cannot be overridden (only appended)
ENTRYPOINT ["node"]
CMD ["app.js"]
# Container always runs `node`, but the file can be changed
```

---

## 4. Building Docker Images

### Command Seen in Lab

```bash
# Build image tagged as node-app:0.1 from current directory
docker build -t node-app:0.1 .

# Build image tagged as node-app:0.2
docker build -t node-app:0.2 .
```

### Build Output Explained

```
[+] Building 19.8s (4/7)                          docker:default
=> [internal] load build definition from Dockerfile     0.0s
=> [internal] load metadata for docker.io/library/node:lts  0.6s
=> [1/3] FROM docker.io/library/node:lts@sha256:8530f76...  19.1s
=> [2/3] WORKDIR /app                                   0.0s  ← CACHED
=> [3/3] ADD . /app                                     0.0s  ← CACHED
=> exporting to image                                   0.2s
```

- **Each line = one Dockerfile instruction = one layer**
- `sha256:` — cryptographic hash guaranteeing image integrity
- `CACHED` — layer unchanged since last build (instant!)
- Build time: 19.8s (first time) vs 0.5s (second time, all cached)

### Build Flags

```bash
docker build -t name:tag .          # Tag the image
docker build -f MyDockerfile .      # Custom Dockerfile name
docker build --no-cache .           # Force rebuild all layers
docker build --build-arg KEY=val .  # Pass build arguments
```

---

## 5. Running Containers

### Basic Run Commands

```bash
# Run interactively (foreground)
docker run -it node-app:0.1

# Run detached (background)
docker run -d node-app:0.1

# Run with port mapping
docker run -p 4000:8080 node-app:0.1

# Run with name
docker run --name my-node-app -d -p 4000:8080 node-app:0.1

# Run and remove on exit
docker run --rm node-app:0.1
```

### Container Lifecycle

```
docker run     → Creates + starts container
docker stop    → Gracefully stops (SIGTERM → wait → SIGKILL)
docker start   → Restarts a stopped container
docker restart → Stop + start
docker rm      → Delete stopped container
docker rm -f   → Force delete (kills if running)
```

### Listing Containers

```bash
docker ps           # Running containers only
docker ps -a        # All containers (including stopped)
docker ps -q        # Only container IDs
```

---

## 6. Port Mapping Explained

### What the Lab Showed

```bash
# This FAILED — port 8080 is inside the container, not the host
curl http://localhost:8080
# curl: (7) Failed to connect to localhost port 8080

# This WORKED — host port 4000 maps to container port 8080
curl http://localhost:4000
# Hello World
```

### Port Mapping Diagram

```
HOST MACHINE                    CONTAINER
┌─────────────────────────┐    ┌─────────────────────────┐
│                         │    │                         │
│  Browser/curl           │    │  Node.js app            │
│  → localhost:4000  ─────┼────┼──→ 8080                 │
│                         │    │                         │
└─────────────────────────┘    └─────────────────────────┘
         -p 4000:8080
         ↑host  ↑container
```

### Syntax

```bash
docker run -p <HOST_PORT>:<CONTAINER_PORT> image:tag

# Examples
docker run -p 80:8080 myapp        # Host:80 → Container:8080
docker run -p 443:443 myapp        # Same port both sides
docker run -p 127.0.0.1:8080:8080  # Bind to specific interface
```

> **Rule:** `EXPOSE` in Dockerfile is documentation only. The `-p` flag at `docker run` actually publishes the port.

---

## 7. Docker Layer Caching

One of Docker's most powerful features — dramatically speeds up builds.

### How Caching Works

```dockerfile
FROM node:lts           # Layer 1 — cached unless node:lts changes
WORKDIR /app            # Layer 2 — cached unless WORKDIR changes
COPY package.json .     # Layer 3 — only re-runs if package.json changes
RUN npm install         # Layer 4 — cached if package.json unchanged ✅
COPY . .                # Layer 5 — re-runs if any source file changes
```

### Optimization Pattern

```dockerfile
# BAD — npm install re-runs every time ANY file changes
COPY . /app
RUN npm install

# GOOD — npm install only re-runs if package.json changes
COPY package*.json ./
RUN npm install
COPY . .
```

### Cache Invalidation Rules

- Any change to a layer **invalidates all layers below it**
- `ADD` and `COPY` instructions check file checksums
- `RUN` instructions use the command string as cache key
- Use `--no-cache` to force full rebuild: `docker build --no-cache .`

---

## 8. Inspecting Containers

### docker inspect (as seen in lab)

```bash
docker inspect daf044b5d100
```

Returns full JSON metadata. Key fields from the lab output:

```json
{
  "Id": "daf044b5d100026968c662a30ef20dddfcdc9ba77ff2a55cd7f13ce76fd7ead",
  "Created": "2026-05-27T12:13:49.6686250792",
  "Path": "docker-entrypoint.sh",
  "Args": ["node", "app.js"],
  "State": {
    "Status": "running",
    "Running": true,
    "Paused": false,
    "Restarting": false,
    "OOMKilled": false,
    "Dead": false,
    "Pid": 1626,
    "ExitCode": 0,
    "StartedAt": "2026-05-27T12:13:49.7754326152"
  },
  "Image": "sha256:9a045f5a869586ad32364ad761631a954b1f4be7292556830644500ca484f80b"
}
```

### What Each Field Means

| Field | Meaning |
|---|---|
| `Id` | Full container ID (unique) |
| `Created` | Timestamp when container was created |
| `Path` | Entrypoint executable |
| `Args` | Arguments passed to entrypoint |
| `State.Status` | `running`, `exited`, `paused`, `dead` |
| `State.Running` | true/false |
| `State.OOMKilled` | Was container killed for exceeding memory? |
| `State.Pid` | Process ID on host |
| `State.ExitCode` | 0 = success, non-zero = error |
| `Image` | SHA256 of the image used |

### Other Useful Inspection Commands

```bash
# View container logs
docker logs <container_id>
docker logs -f <container_id>          # Follow (stream) logs

# Execute command inside running container
docker exec -it <container_id> bash    # Interactive shell
docker exec <container_id> ls /app     # Single command

# View container resource usage
docker stats <container_id>

# View running processes
docker top <container_id>

# Copy files to/from container
docker cp myfile.txt <container_id>:/app/
docker cp <container_id>:/app/logs.txt ./
```

---

## 9. Docker Image Management

### Commands from the Lab

```bash
# List all images
docker images

# Output:
# IMAGE              ID              DISK USAGE    CONTENT SIZE
# hello-world:latest f7931603f70e    20.3kB        3.96kB
# node-app:0.1       6ab532e26f8d    1.62GB        408MB
# node-app:0.2       <new-id>        1.62GB        408MB
```

### Image Sizes Explained

- `hello-world` = 20.3 kB — minimal test image, no OS
- `node-app:0.1` = 1.62 GB — includes full Node.js LTS runtime
- `node:slim` = ~250 MB — reduced dependencies
- `node:alpine` = ~120 MB — Alpine Linux base, minimal

### Tagging and Pushing

```bash
# Retag an image for a registry
docker tag node-app:0.1 us-east4-docker.pkg.dev/PROJECT_ID/my-repo/node-app:0.1

# Push to registry
docker push us-east4-docker.pkg.dev/PROJECT_ID/my-repo/node-app:0.1

# Pull from registry
docker pull us-east4-docker.pkg.dev/PROJECT_ID/my-repo/node-app:0.1
```

### Cleanup Commands

```bash
docker rmi node-app:0.1                 # Remove image by name
docker rmi <image_id>                   # Remove by ID
docker image prune                      # Remove dangling images
docker image prune -a                   # Remove all unused images
docker system prune                     # Remove everything unused
```

> **Dependency rule:** You cannot remove `node` without first removing `node-app` — child images depend on parent images.

---

## 10. Google Artifact Registry

### What It Is

Google Cloud's managed container registry — the modern replacement for Container Registry (`gcr.io`). Stores Docker images, Maven packages, npm packages, and more.

### Setup Seen in Lab

1. Navigate to **Artifact Registry → Create Repository**
2. Repository type: **Docker**
3. Region: `us-east4`
4. **Vulnerability scanning: Enabled** ← automatic CVE scanning on push
5. Cleanup policies configured
6. Platform logs: inherited from project

### Vulnerability Scanning

When enabled, every pushed image is automatically scanned by **Artifact Analysis** for known CVEs (Common Vulnerabilities and Exposures). Results appear in the GCP Console under Security.

### Pushing to Artifact Registry

```bash
# 1. Configure Docker to use GCP credentials
gcloud auth configure-docker us-east4-docker.pkg.dev

# 2. Tag your image with the registry path
docker tag node-app:0.1 \
  us-east4-docker.pkg.dev/PROJECT_ID/REPO_NAME/node-app:0.1

# 3. Push
docker push \
  us-east4-docker.pkg.dev/PROJECT_ID/REPO_NAME/node-app:0.1
```

### Why Artifact Registry Over Docker Hub for Production?

| Feature | Docker Hub (free) | Google Artifact Registry |
|---|---|---|
| Access control | Public by default | IAM-controlled (private) |
| Integration | Manual | Native GKE, Cloud Run, Cloud Build |
| Vulnerability scanning | Paid | Built-in |
| Storage location | Global | Regional (lower latency) |
| Cleanup policies | Manual | Automated |

---

## 11. Complete Command Reference

### Build & Tag
```bash
docker build -t name:tag .           # Build from Dockerfile in current dir
docker build -t name:tag -f path/Dockerfile .  # Custom Dockerfile
docker build --no-cache -t name:tag .           # No layer caching
docker tag source:tag target:tag     # Retag image
```

### Images
```bash
docker images                        # List images
docker images -a                     # Including intermediate layers
docker rmi name:tag                  # Remove image
docker image prune                   # Remove dangling images
docker pull name:tag                 # Pull from registry
docker push name:tag                 # Push to registry
docker inspect name:tag              # Full image metadata
docker history name:tag              # Layer history
```

### Containers
```bash
docker run name:tag                  # Run (foreground)
docker run -d name:tag               # Run detached
docker run -it name:tag bash         # Interactive shell
docker run -p host:container name:tag  # Port mapping
docker run --name myname name:tag    # Named container
docker run -e KEY=VALUE name:tag     # Environment variable
docker run -v /host:/container name:tag  # Volume mount
docker run --rm name:tag             # Auto-remove on exit
docker ps                            # List running
docker ps -a                         # List all
docker stop <id>                     # Graceful stop
docker kill <id>                     # Force kill
docker rm <id>                       # Remove container
docker rm -f <id>                    # Force remove
docker start <id>                    # Start stopped container
docker restart <id>                  # Restart
```

### Debug & Inspect
```bash
docker inspect <id>                  # Full JSON metadata
docker logs <id>                     # Container logs
docker logs -f <id>                  # Follow logs
docker exec -it <id> bash            # Shell into container
docker exec <id> <cmd>               # Run command
docker stats                         # Resource usage (live)
docker top <id>                      # Running processes
docker cp <id>:/path ./local         # Copy from container
```

### System
```bash
docker system df                     # Disk usage
docker system prune                  # Clean up unused resources
docker system prune -a --volumes     # Nuclear clean
docker info                          # Docker daemon info
docker version                       # Client + server versions
```

---

## 12. Production Best Practices

### Image Optimization

```dockerfile
# 1. Use specific versions, not :latest
FROM node:20.11-alpine          # ✅ pinned version + slim base
# FROM node:latest              # ❌ unpredictable

# 2. Use .dockerignore
# .dockerignore file:
# node_modules
# .git
# *.log
# .env

# 3. Multi-stage builds (drastically reduce image size)
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
CMD ["node", "app.js"]

# 4. Non-root user (security)
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

### Security Checklist

- [ ] Never store secrets in Dockerfile (`ENV SECRET=...`) — use Secrets Manager
- [ ] Run as non-root user
- [ ] Use minimal base images (Alpine, Distroless)
- [ ] Enable vulnerability scanning (Artifact Registry)
- [ ] Pin base image versions (`node:20.11-alpine3.18`)
- [ ] Use `.dockerignore` to exclude sensitive files

### Health Checks

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

### Resource Limits

```bash
docker run \
  --memory="512m" \          # Max 512 MB RAM
  --cpus="1.0" \             # Max 1 CPU core
  node-app:0.1
```

---

## Full Pipeline Summary

```
Developer writes code
        ↓
  Creates Dockerfile
        ↓
  docker build -t app:v1 .        ← Builds layers
        ↓
  docker run -p 4000:8080 app:v1  ← Test locally
        ↓
  curl localhost:4000              ← Verify "Hello World"
        ↓
  docker inspect <id>              ← Debug/validate
        ↓
  docker tag + docker push         ← Push to Artifact Registry
        ↓
  Vulnerability scan (auto)        ← GCP Artifact Analysis
        ↓
  Deploy to Cloud Run / GKE        ← Production
```

---
