# Module 1.1: Docker & Containerization Basics

## 🎯 Learning Objectives

By the end of this module, you will:
- Understand the fundamental differences between containers and VMs
- Write production-ready Dockerfiles
- Implement multi-stage builds for optimization
- Configure Docker networking
- Apply container security best practices

---

## 📖 Section 1: Understanding Containers vs Virtual Machines

### What is a Virtual Machine?

A **Virtual Machine (VM)** is a complete virtualization of a physical computer:

```
┌─────────────────────────────────────┐
│         Application A & B           │
├─────────────────────────────────────┤
│        Guest OS (Ubuntu)            │
├─────────────────────────────────────┤
│          Hypervisor (VMware)        │
├─────────────────────────────────────┤
│        Host OS (Windows/Linux)      │
├─────────────────────────────────────┤
│       Physical Hardware (CPU/RAM)   │
└─────────────────────────────────────┘
```

**Characteristics:**
- Full operating system (kernel, libraries, binaries)
- Hardware virtualization via hypervisor
- Strong isolation (separate OS per VM)
- Heavy resource consumption (GBs of RAM, disk)
- Slow startup time (minutes)

### What is a Container?

A **Container** is an isolated process that shares the host OS kernel:

```
┌─────────────────────────────────────┐
│    App A    │    App B    │  App C  │
│   + Libs    │   + Libs    │ + Libs  │
├─────────────┼─────────────┼─────────┤
│         Container Runtime            │
│            (Docker Engine)           │
├─────────────────────────────────────┤
│        Host OS (Linux Kernel)       │
├─────────────────────────────────────┤
│       Physical Hardware (CPU/RAM)   │
└─────────────────────────────────────┘
```

**Characteristics:**
- Shares host OS kernel
- Process-level isolation (namespaces, cgroups)
- Lightweight (MBs of disk, minimal RAM overhead)
- Fast startup (seconds)
- Portable across environments

### Key Differences

| Feature | Virtual Machines | Containers |
|---------|-----------------|------------|
| **Isolation** | Hardware-level (hypervisor) | Process-level (kernel features) |
| **OS** | Full OS per VM | Shared host OS kernel |
| **Size** | GBs (includes full OS) | MBs (app + dependencies only) |
| **Startup** | Minutes | Seconds |
| **Resource Usage** | Heavy | Lightweight |
| **Portability** | Less portable | Highly portable |
| **Use Case** | Different OS requirements | Microservices, cloud-native apps |

### When to Use What?

**Use VMs when:**
- You need to run different operating systems (Windows on Linux host)
- Require hardware-level isolation for security
- Running legacy applications with OS dependencies
- Need full OS control and customization

**Use Containers when:**
- Building cloud-native, microservices applications
- Need rapid deployment and scaling
- Want consistent environments (dev, staging, prod)
- Optimizing resource utilization
- **This is what we'll use for ECS!**

---

## 📖 Section 2: Docker Architecture

### Docker Components

```
┌──────────────────────────────────────────────┐
│              Docker Client (CLI)             │
│              $ docker run nginx              │
└────────────────┬─────────────────────────────┘
                 │ REST API
┌────────────────▼─────────────────────────────┐
│           Docker Daemon (dockerd)            │
│  ┌────────────────────────────────────────┐  │
│  │     Container Management               │  │
│  │  - Create  - Start  - Stop  - Delete   │  │
│  └────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────┐  │
│  │     Image Management                   │  │
│  │  - Build  - Pull  - Push  - Tag        │  │
│  └────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────┐  │
│  │     Network & Volume Management        │  │
│  └────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
                 │
┌────────────────▼─────────────────────────────┐
│         containerd (Runtime)                 │
└────────────────┬─────────────────────────────┘
                 │
┌────────────────▼─────────────────────────────┐
│              runc (OCI Runtime)              │
│          Creates actual containers           │
└──────────────────────────────────────────────┘
```

### Key Concepts

1. **Docker Image**: Read-only template with instructions for creating a container
   - Consists of layers (union file system)
   - Immutable once built
   - Stored in registries (Docker Hub, ECR)

2. **Docker Container**: Running instance of an image
   - Isolated process with its own filesystem, network, and process tree
   - Ephemeral by default (data lost when deleted)
   - Can be started, stopped, moved, and deleted

3. **Dockerfile**: Text file with instructions to build an image
4. **Docker Registry**: Storage for Docker images (Docker Hub, AWS ECR, etc.)

---

## 📖 Section 3: Writing Production-Ready Dockerfiles

### Basic Dockerfile Structure

```dockerfile
# Dockerfile for Node.js API

# 1. Base Image - Start from an official image
FROM node:18-alpine

# 2. Metadata
LABEL maintainer="franck@example.com"
LABEL version="1.0"
LABEL description="Production-ready Node.js API"

# 3. Working Directory
WORKDIR /app

# 4. Copy dependency files first (for layer caching)
COPY package*.json ./

# 5. Install dependencies
RUN npm ci --only=production

# 6. Copy application code
COPY . .

# 7. Expose port (documentation only)
EXPOSE 3000

# 8. Non-root user for security
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001
USER nodejs

# 9. Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node healthcheck.js

# 10. Start command
CMD ["node", "server.js"]
```

### Dockerfile Best Practices

#### 1. **Use Specific Base Image Tags**

```dockerfile
# ❌ Bad - Uses latest, unpredictable
FROM node:latest

# ✅ Good - Specific version
FROM node:18.17.1-alpine3.18

# ✅ Better - Use digest for immutability
FROM node:18.17.1-alpine3.18@sha256:abc123...
```

#### 2. **Minimize Layers & Image Size**

```dockerfile
# ❌ Bad - Multiple RUN commands create multiple layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
RUN apt-get clean

# ✅ Good - Chain commands in single RUN
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        git && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

#### 3. **Leverage Build Cache**

```dockerfile
# ✅ Copy dependency files first, then install
# This allows Docker to cache the dependency layer
COPY package*.json ./
RUN npm ci --only=production

# Copy source code last (changes frequently)
COPY . .
```

#### 4. **Use .dockerignore**

Create `.dockerignore` to exclude unnecessary files:

```
# .dockerignore
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
.DS_Store
coverage/
.vscode/
*.test.js
```

#### 5. **Run as Non-Root User**

```dockerfile
# Create user and group
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

# Change ownership of app files
RUN chown -R appuser:appgroup /app

# Switch to non-root user
USER appuser
```

---

## 📖 Section 4: Multi-Stage Builds for Optimization

Multi-stage builds allow you to use multiple `FROM` statements in your Dockerfile. Each `FROM` begins a new build stage. You can selectively copy artifacts from one stage to another.

### Why Multi-Stage Builds?

- **Smaller final images**: Only production dependencies in final image
- **Security**: Build tools and source code not in production image
- **Separation of concerns**: Build stage vs runtime stage

### Example: Node.js Application

```dockerfile
# ============================================
# Stage 1: Build Stage
# ============================================
FROM node:18-alpine AS builder

WORKDIR /app

# Copy dependency files
COPY package*.json ./

# Install ALL dependencies (including devDependencies)
RUN npm ci

# Copy source code
COPY . .

# Run tests
RUN npm run test

# Build application (TypeScript, bundling, etc.)
RUN npm run build

# ============================================
# Stage 2: Production Stage
# ============================================
FROM node:18-alpine AS production

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install ONLY production dependencies
RUN npm ci --only=production && \
    npm cache clean --force

# Copy built artifacts from builder stage
COPY --from=builder /app/dist ./dist

# Security: Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001 && \
    chown -R nodejs:nodejs /app

USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node dist/healthcheck.js

# Start application
CMD ["node", "dist/server.js"]
```

### Size Comparison

```bash
# Single-stage build (with dev dependencies)
node-app:single    450MB

# Multi-stage build (production only)
node-app:multi     150MB

# Reduction: 67% smaller! 🎉
```

### Advanced: Multiple Target Stages

```dockerfile
# Development stage
FROM node:18-alpine AS development
WORKDIR /app
COPY package*.json ./
RUN npm install  # All dependencies including dev
COPY . .
CMD ["npm", "run", "dev"]

# Test stage
FROM development AS test
RUN npm run test
RUN npm run lint

# Build stage
FROM development AS builder
RUN npm run build

# Production stage
FROM node:18-alpine AS production
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY --from=builder /app/dist ./dist
USER node
CMD ["node", "dist/server.js"]
```

**Build specific stage:**
```bash
# Build for development
docker build --target development -t myapp:dev .

# Build for production
docker build --target production -t myapp:prod .

# Run tests
docker build --target test -t myapp:test .
```

---

## 📖 Section 5: Docker Networking Basics

### Network Types

Docker provides several network drivers:

#### 1. **Bridge Network (Default)**

```
┌──────────────────────────────────────┐
│            Host Machine              │
│  ┌────────────────────────────────┐  │
│  │    Docker Bridge Network       │  │
│  │      (172.17.0.0/16)           │  │
│  │  ┌──────┐  ┌──────┐  ┌──────┐ │  │
│  │  │ App1 │  │ App2 │  │ App3 │ │  │
│  │  │.0.2  │  │.0.3  │  │.0.4  │ │  │
│  │  └──────┘  └──────┘  └──────┘ │  │
│  └────────────────────────────────┘  │
│              │ NAT                    │
│  ┌───────────▼────────────────────┐  │
│  │      Host Network Interface    │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

**Usage:**
```bash
# Containers on same bridge network can communicate by container name
docker network create my-app-network
docker run --network my-app-network --name api myapi:latest
docker run --network my-app-network --name db postgres:15
# API can connect to: postgresql://db:5432
```

#### 2. **Host Network**

Container shares host's network stack (no isolation).

```bash
docker run --network host myapp:latest
# App binds directly to host's ports
```

⚠️ **Not recommended for production** - no network isolation

#### 3. **None Network**

No network access (completely isolated).

```bash
docker run --network none myapp:latest
```

### Container Communication

```bash
# Create custom bridge network
docker network create --driver bridge app-network

# Run database
docker run -d \
  --name postgres-db \
  --network app-network \
  -e POSTGRES_PASSWORD=secret \
  postgres:15

# Run API (can connect to postgres-db by name)
docker run -d \
  --name api \
  --network app-network \
  -e DATABASE_URL=postgresql://postgres:secret@postgres-db:5432/mydb \
  myapi:latest

# API connects to database using hostname "postgres-db"
```

### Port Mapping

```bash
# Map container port to host port
docker run -p 8080:3000 myapi:latest
#         └─┬─┘ └─┬─┘
#      Host Port │
#           Container Port

# Access: http://localhost:8080
```

---

## 📖 Section 6: Container Security Best Practices

### 1. **Use Official & Minimal Base Images**

```dockerfile
# ❌ Avoid - Large, many vulnerabilities
FROM ubuntu:latest

# ✅ Better - Minimal attack surface
FROM node:18-alpine

# ✅ Best - Distroless (no shell, no package manager)
FROM gcr.io/distroless/nodejs18-debian11
```

### 2. **Scan Images for Vulnerabilities**

```bash
# Using Docker Scout
docker scout cves myapp:latest

# Using Trivy
trivy image myapp:latest

# Using Snyk
snyk container test myapp:latest
```

### 3. **Run as Non-Root User**

```dockerfile
# ❌ Bad - Runs as root (default)
FROM node:18-alpine
COPY . /app
CMD ["node", "server.js"]

# ✅ Good - Runs as non-root user
FROM node:18-alpine
RUN addgroup -g 1001 nodejs && \
    adduser -S -u 1001 -G nodejs nodejs
WORKDIR /app
COPY --chown=nodejs:nodejs . .
USER nodejs
CMD ["node", "server.js"]
```

### 4. **Use Read-Only Root Filesystem**

```bash
docker run --read-only --tmpfs /tmp myapp:latest
```

```dockerfile
# In Dockerfile
FROM node:18-alpine
# ... app setup ...
USER nodejs

# Mark rootfs as read-only
# Use volumes for writable directories
VOLUME ["/app/logs", "/app/uploads"]
```

### 5. **Limit Container Capabilities**

```bash
# Drop all capabilities, add only what's needed
docker run \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  myapp:latest
```

### 6. **Resource Limits**

```bash
docker run \
  --memory="512m" \
  --cpus="1.0" \
  --pids-limit=100 \
  myapp:latest
```

### 7. **Never Store Secrets in Images**

```dockerfile
# ❌ NEVER DO THIS
ENV DATABASE_PASSWORD=mysecretpassword
COPY .env /app/.env

# ✅ Use environment variables at runtime
docker run -e DATABASE_PASSWORD=${DB_PASS} myapp:latest

# ✅ Or use Docker secrets (Swarm/ECS)
# We'll cover AWS Secrets Manager for ECS later
```

### 8. **Keep Images Updated**

```bash
# Regularly rebuild images to get security patches
docker build --no-cache -t myapp:latest .

# Use automated tools to monitor base images
# Dependabot, Renovate, etc.
```

---

## 📖 Section 7: Essential Docker Commands

### Image Management

```bash
# Build an image
docker build -t myapp:v1.0 .
docker build -t myapp:latest --target production .

# List images
docker images
docker image ls

# Remove image
docker rmi myapp:v1.0
docker image rm myapp:v1.0

# Tag image
docker tag myapp:v1.0 myapp:latest
docker tag myapp:v1.0 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.0

# Push to registry
docker push myapp:v1.0

# Pull from registry
docker pull nginx:latest

# Inspect image layers
docker history myapp:latest
docker image inspect myapp:latest
```

### Container Management

```bash
# Run container
docker run -d --name myapp -p 8080:3000 myapp:latest

# Flags:
# -d                  Detached mode (background)
# --name              Container name
# -p                  Port mapping
# -e                  Environment variables
# -v                  Volume mount
# --network           Network
# --rm                Remove after stop

# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# Stop container
docker stop myapp

# Start stopped container
docker start myapp

# Restart container
docker restart myapp

# Remove container
docker rm myapp
docker rm -f myapp  # Force remove running container

# View logs
docker logs myapp
docker logs -f myapp  # Follow logs
docker logs --tail 100 myapp  # Last 100 lines

# Execute command in running container
docker exec -it myapp /bin/sh
docker exec myapp node -v

# Inspect container
docker inspect myapp

# View resource usage
docker stats myapp
```

### Network Commands

```bash
# List networks
docker network ls

# Create network
docker network create my-network

# Inspect network
docker network inspect my-network

# Remove network
docker network rm my-network

# Connect container to network
docker network connect my-network myapp

# Disconnect
docker network disconnect my-network myapp
```

### Volume Commands

```bash
# List volumes
docker volume ls

# Create volume
docker volume create mydata

# Inspect volume
docker volume inspect mydata

# Remove volume
docker volume rm mydata

# Use volume
docker run -v mydata:/app/data myapp:latest
```

### Cleanup Commands

```bash
# Remove stopped containers
docker container prune

# Remove unused images
docker image prune
docker image prune -a  # Remove all unused images

# Remove unused volumes
docker volume prune

# Remove unused networks
docker network prune

# Remove everything unused
docker system prune
docker system prune -a --volumes  # Nuclear option
```

---

## 🎯 Key Takeaways

1. **Containers vs VMs**: Containers are lightweight, fast, portable process isolation; VMs are full OS virtualization
2. **Production Dockerfiles**: Use specific tags, minimize layers, run as non-root, leverage .dockerignore
3. **Multi-Stage Builds**: Separate build and runtime stages for smaller, more secure images
4. **Networking**: Use custom bridge networks for container communication; map ports for external access
5. **Security**: Use minimal base images, scan for vulnerabilities, run as non-root, never embed secrets

---

## 📝 Next Steps

Now that you understand Docker fundamentals, you'll:
1. Complete the hands-on project (containerizing a REST API)
2. Learn AWS core services needed for ECS
3. Prepare to push images to AWS ECR (Elastic Container Registry)

Ready to build your first production-ready container? Check out [project.md](./project.md)!
