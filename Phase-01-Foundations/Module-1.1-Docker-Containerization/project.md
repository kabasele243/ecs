# Project 1.1: Containerize a Production-Ready REST API

## 🎯 Project Goal

Build and containerize a simple REST API with production-ready Docker practices including:
- Multi-stage Dockerfile
- Security best practices (non-root user, minimal base image)
- Health checks
- Proper logging
- Environment configuration
- Docker Compose for local development

---

## 📋 Prerequisites

- Docker installed (version 20.10+)
- Docker Compose installed (version 2.0+)
- Code editor (VS Code recommended)
- Basic knowledge of Node.js or Python

---

## 🏗️ Project Structure

```
docker-api-project/
├── src/
│   ├── server.js           # Main application
│   ├── healthcheck.js      # Health check endpoint
│   └── routes/
│       └── api.js          # API routes
├── Dockerfile              # Production-ready multi-stage build
├── Dockerfile.dev          # Development Dockerfile
├── .dockerignore          # Files to exclude from build
├── docker-compose.yml     # Local development setup
├── package.json           # Node.js dependencies
└── README.md              # Project documentation
```

---

## 📝 Step 1: Create the Application

### 1.1 Initialize Project

```bash
# Create project directory
mkdir docker-api-project
cd docker-api-project

# Initialize npm project
npm init -y

# Install dependencies
npm install express
npm install --save-dev nodemon
```

### 1.2 Create Application Files

**`src/server.js`** - Main application:

```javascript
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

// Middleware
app.use(express.json());

// Routes
app.get('/', (req, res) => {
  res.json({
    message: 'Welcome to Docker API',
    version: '1.0.0',
    environment: process.env.NODE_ENV || 'development'
  });
});

app.get('/api/users', (req, res) => {
  res.json({
    users: [
      { id: 1, name: 'Alice', email: 'alice@example.com' },
      { id: 2, name: 'Bob', email: 'bob@example.com' },
      { id: 3, name: 'Charlie', email: 'charlie@example.com' }
    ]
  });
});

app.post('/api/users', (req, res) => {
  const { name, email } = req.body;
  res.status(201).json({
    id: 4,
    name,
    email,
    created: new Date().toISOString()
  });
});

// Health check endpoint (for Docker)
app.get('/health', (req, res) => {
  res.status(200).json({
    status: 'healthy',
    timestamp: new Date().toISOString(),
    uptime: process.uptime()
  });
});

// Start server
const server = app.listen(PORT, '0.0.0.0', () => {
  console.log(`Server running on port ${PORT}`);
  console.log(`Environment: ${process.env.NODE_ENV || 'development'}`);
});

// Graceful shutdown
process.on('SIGTERM', () => {
  console.log('SIGTERM signal received: closing HTTP server');
  server.close(() => {
    console.log('HTTP server closed');
  });
});

module.exports = app;
```

**`src/healthcheck.js`** - Standalone health check for Docker:

```javascript
const http = require('http');

const options = {
  host: 'localhost',
  port: process.env.PORT || 3000,
  path: '/health',
  timeout: 2000
};

const request = http.request(options, (res) => {
  console.log(`Health check status: ${res.statusCode}`);
  if (res.statusCode === 200) {
    process.exit(0);
  } else {
    process.exit(1);
  }
});

request.on('error', (err) => {
  console.error('Health check failed:', err.message);
  process.exit(1);
});

request.end();
```

### 1.3 Update package.json

Add scripts to `package.json`:

```json
{
  "name": "docker-api-project",
  "version": "1.0.0",
  "description": "Production-ready containerized REST API",
  "main": "src/server.js",
  "scripts": {
    "start": "NODE_ENV=production node src/server.js",
    "dev": "nodemon src/server.js"
  },
  "keywords": ["docker", "api", "express"],
  "author": "Franck Kabasele",
  "license": "MIT",
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

---

## 🐳 Step 2: Create Production Dockerfile

**`Dockerfile`** - Multi-stage production build:

```dockerfile
# ============================================
# Stage 1: Dependencies
# ============================================
FROM node:18-alpine AS deps

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install production dependencies only
RUN npm ci --only=production && \
    npm cache clean --force

# ============================================
# Stage 2: Build (if you had TypeScript/build step)
# ============================================
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install all dependencies (including dev)
RUN npm ci

# Copy source code
COPY src ./src

# In a real app, you'd run: RUN npm run build
# For this simple app, we just copy the source

# ============================================
# Stage 3: Production Runtime
# ============================================
FROM node:18-alpine AS production

# Set environment to production
ENV NODE_ENV=production \
    PORT=3000

WORKDIR /app

# Copy production dependencies from deps stage
COPY --from=deps /app/node_modules ./node_modules
COPY --from=deps /app/package*.json ./

# Copy application code
COPY src ./src

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001 -G nodejs && \
    chown -R nodejs:nodejs /app

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node src/healthcheck.js

# Start application
CMD ["node", "src/server.js"]
```

---

## 🛡️ Step 3: Create .dockerignore

**`.dockerignore`**:

```
# Dependencies
node_modules
npm-debug.log
yarn-error.log

# Git
.git
.gitignore

# IDE
.vscode
.idea
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Documentation
README.md
LICENSE

# Docker
Dockerfile*
docker-compose*.yml

# Environment
.env
.env.local

# Tests
*.test.js
coverage/
```

---

## 🔧 Step 4: Development Dockerfile

**`Dockerfile.dev`** - For development with hot reload:

```dockerfile
FROM node:18-alpine

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm install

# Copy source code
COPY . .

# Expose port
EXPOSE 3000

# Use nodemon for hot reload
CMD ["npm", "run", "dev"]
```

---

## 🐳 Step 5: Docker Compose Setup

**`docker-compose.yml`** - Local development environment:

```yaml
version: '3.9'

services:
  # Development API service
  api:
    build:
      context: .
      dockerfile: Dockerfile.dev
    container_name: docker-api-dev
    ports:
      - "3000:3000"
    volumes:
      # Mount source code for hot reload
      - ./src:/app/src
      # Named volume for node_modules (prevents overwriting)
      - node_modules:/app/node_modules
    environment:
      - NODE_ENV=development
      - PORT=3000
    networks:
      - app-network
    restart: unless-stopped

  # Production-like API service
  api-prod:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
    container_name: docker-api-prod
    ports:
      - "3001:3000"
    environment:
      - NODE_ENV=production
      - PORT=3000
    networks:
      - app-network
    restart: unless-stopped
    profiles:
      - production  # Only run when --profile production is specified

volumes:
  node_modules:

networks:
  app-network:
    driver: bridge
```

---

## 🚀 Step 6: Build and Run

### 6.1 Test Locally (without Docker)

```bash
# Install dependencies
npm install

# Run in development
npm run dev

# Test endpoints
curl http://localhost:3000
curl http://localhost:3000/health
curl http://localhost:3000/api/users
```

### 6.2 Build Docker Image

```bash
# Build production image
docker build -t docker-api:latest .

# Build with specific tag
docker build -t docker-api:v1.0.0 .

# Check image size
docker images | grep docker-api
```

### 6.3 Run Container

```bash
# Run production container
docker run -d \
  --name my-api \
  -p 3000:3000 \
  -e NODE_ENV=production \
  docker-api:latest

# Check container is running
docker ps

# View logs
docker logs my-api
docker logs -f my-api  # Follow logs

# Test the API
curl http://localhost:3000
curl http://localhost:3000/health
curl http://localhost:3000/api/users
```

### 6.4 Use Docker Compose

```bash
# Start development environment
docker-compose up

# Start in detached mode
docker-compose up -d

# View logs
docker-compose logs -f api

# Start production environment
docker-compose --profile production up

# Stop services
docker-compose down

# Rebuild and start
docker-compose up --build
```

---

## ✅ Step 7: Verify Production Best Practices

### 7.1 Check Image Size

```bash
docker images docker-api:latest

# Should be around 150-200MB (Alpine + Node.js + app)
```

### 7.2 Verify Non-Root User

```bash
# Inspect running container
docker exec my-api whoami
# Should output: nodejs (not root!)

# Check process user
docker exec my-api ps aux
```

### 7.3 Test Health Check

```bash
# Check health status
docker inspect my-api | grep -A 10 Health

# Health check should show "healthy" status
docker ps
# Look for "healthy" in STATUS column
```

### 7.4 Inspect Image Layers

```bash
# View image history
docker history docker-api:latest

# Count layers
docker history docker-api:latest | wc -l
```

### 7.5 Security Scan

```bash
# Scan for vulnerabilities (if you have docker scout)
docker scout cves docker-api:latest

# Or use Trivy
trivy image docker-api:latest
```

---

## 🧪 Step 8: Testing Scenarios

### Test 1: Container Restart

```bash
# Stop and restart container
docker stop my-api
docker start my-api

# API should come back up quickly (< 5 seconds)
curl http://localhost:3000/health
```

### Test 2: Graceful Shutdown

```bash
# Send SIGTERM to container
docker stop my-api

# Check logs - should show graceful shutdown message
docker logs my-api
```

### Test 3: Resource Limits

```bash
# Run with resource limits
docker run -d \
  --name my-api-limited \
  -p 3002:3000 \
  --memory="256m" \
  --cpus="0.5" \
  docker-api:latest

# Monitor resource usage
docker stats my-api-limited
```

### Test 4: Network Isolation

```bash
# Create custom network
docker network create api-network

# Run container on custom network
docker run -d \
  --name my-api-isolated \
  --network api-network \
  -p 3003:3000 \
  docker-api:latest

# Verify network
docker network inspect api-network
```

---

## 📊 Step 9: Optimization Challenges

Try these optimizations to improve your Docker skills:

### Challenge 1: Reduce Image Size

**Goal**: Get image size below 100MB

**Hints**:
- Use `node:18-alpine` (already doing this ✓)
- Remove unnecessary files in .dockerignore
- Use multi-stage build to exclude devDependencies (already doing this ✓)
- Try distroless base image

```dockerfile
# Ultra-minimal (advanced)
FROM gcr.io/distroless/nodejs18-debian11
COPY --from=deps /app /app
WORKDIR /app
CMD ["src/server.js"]
```

### Challenge 2: Improve Build Cache

**Goal**: Make rebuilds faster by optimizing layer caching

**Current issue**: Changing source code invalidates all layers after `COPY . .`

**Solution**: Already implemented! Dependencies are copied and installed before source code.

### Challenge 3: Add Development Tools

**Goal**: Create a development container with debugging tools

```dockerfile
# Dockerfile.debug
FROM node:18-alpine

# Install debugging tools
RUN apk add --no-cache curl htop

WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .

# Enable Node.js inspector
CMD ["node", "--inspect=0.0.0.0:9229", "src/server.js"]
```

---

## 🎯 Success Criteria

Your project is complete when you can:

- [ ] Build the Docker image successfully
- [ ] Run the container and access all endpoints
- [ ] Verify the container runs as non-root user (nodejs)
- [ ] See "healthy" status in `docker ps`
- [ ] Image size is reasonable (< 250MB)
- [ ] Container restarts quickly (< 5 seconds)
- [ ] Logs show proper graceful shutdown
- [ ] Can use Docker Compose for local development
- [ ] Understand each line in the Dockerfile

---

## 📚 What You Learned

1. **Multi-stage Builds**: Separate dependency installation, build, and runtime stages
2. **Security**: Non-root user, minimal base image, .dockerignore
3. **Health Checks**: Docker-native health monitoring
4. **Graceful Shutdown**: Handle SIGTERM for clean container stops
5. **Development vs Production**: Different Dockerfiles for different environments
6. **Docker Compose**: Orchestrate multiple services locally
7. **Resource Management**: Memory and CPU limits
8. **Best Practices**: Layer caching, image optimization, security scanning

---

## 🔄 Next Steps

After completing this project:

1. **Add notes** to [notes.md](./notes.md) about what you learned
2. **Move to Module 1.2**: AWS Core Services Review
3. **Later**: We'll push this image to AWS ECR (Elastic Container Registry)
4. **Eventually**: Deploy this container to AWS ECS

---

## 💡 Bonus: Advanced Additions

### Add Database (PostgreSQL)

Update `docker-compose.yml`:

```yaml
services:
  api:
    # ... existing config ...
    depends_on:
      - db
    environment:
      - DATABASE_URL=postgresql://postgres:password@db:5432/mydb

  db:
    image: postgres:15-alpine
    container_name: postgres-db
    environment:
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network

volumes:
  postgres_data:
```

### Add Nginx Reverse Proxy

```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - api
    networks:
      - app-network
```

---

## 📖 Resources

- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Node.js Docker Best Practices](https://github.com/nodejs/docker-node/blob/main/docs/BestPractices.md)
- [Dockerfile Reference](https://docs.docker.com/engine/reference/builder/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)

---

**Congratulations!** You've built a production-ready containerized API. This is the foundation for everything we'll do with ECS.
