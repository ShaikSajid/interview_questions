# Q49: Docker - Containerization for Node.js

## 📋 Summary
This question covers **Docker containerization** for Node.js applications - creating efficient Docker images, multi-stage builds, security best practices, and production-ready Dockerfiles for banking applications.

**Key Topics**:
- Docker fundamentals and architecture
- Dockerfile best practices for Node.js
- Multi-stage builds for optimization
- Layer caching and image size reduction
- Security hardening (non-root user, scanning)
- Environment variable management
- Health checks and graceful shutdown
- Production deployment strategies

**Banking Use Cases**:
- Containerized banking API
- Microservices deployment
- CI/CD pipeline integration
- Development environment consistency
- Scalable container orchestration

---

## 🎯 Understanding Docker

### What is Docker?

**Docker** packages applications with their dependencies into lightweight, portable containers.

```
┌─────────────────────────────────────────────────────────────┐
│                  Docker Architecture                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────────────────────────┐            │
│  │         Application Container               │            │
│  │  ┌────────────────────────────────┐        │            │
│  │  │   Banking API (Node.js)        │        │            │
│  │  │   - app.js                     │        │            │
│  │  │   - package.json               │        │            │
│  │  │   - node_modules/              │        │            │
│  │  └────────────────────────────────┘        │            │
│  │  ┌────────────────────────────────┐        │            │
│  │  │   Node.js Runtime (18.x)       │        │            │
│  │  └────────────────────────────────┘        │            │
│  │  ┌────────────────────────────────┐        │            │
│  │  │   Base OS (Alpine Linux)       │        │            │
│  │  └────────────────────────────────┘        │            │
│  └────────────────────────────────────────────┘            │
│                      │                                      │
│                      ▼                                      │
│  ┌────────────────────────────────────────────┐            │
│  │           Docker Engine                     │            │
│  └────────────────────────────────────────────┘            │
│                      │                                      │
│                      ▼                                      │
│  ┌────────────────────────────────────────────┐            │
│  │           Host OS (Linux)                   │            │
│  └────────────────────────────────────────────┘            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Docker vs Virtual Machines

| Aspect | Docker Container | Virtual Machine |
|--------|-----------------|-----------------|
| **Size** | MBs | GBs |
| **Startup** | Seconds | Minutes |
| **Performance** | Native | Overhead |
| **Isolation** | Process-level | OS-level |
| **Resource Usage** | Low | High |
| **Portability** | High | Medium |

### Key Docker Concepts

**Image**: Blueprint for container (read-only template)
**Container**: Running instance of an image
**Dockerfile**: Instructions to build an image
**Registry**: Repository for images (Docker Hub, ECR)
**Volume**: Persistent data storage
**Network**: Communication between containers

---

## 💡 Example 1: Banking API Dockerization

Complete production-ready Dockerfile with multi-stage build and security best practices.

### Project Structure

```
banking-api/
├── src/
│   ├── server.js
│   ├── routes/
│   ├── controllers/
│   └── models/
├── package.json
├── package-lock.json
├── .dockerignore
├── Dockerfile
├── Dockerfile.dev
└── docker-compose.yml
```

### Basic Dockerfile (Not Optimized)

```dockerfile
# ❌ Basic but not production-ready
FROM node:18

# Set working directory
WORKDIR /app

# Copy everything
COPY . .

# Install dependencies
RUN npm install

# Expose port
EXPOSE 3000

# Start application
CMD ["node", "src/server.js"]

# Issues:
# - Large image size (includes dev dependencies)
# - Runs as root user (security risk)
# - No layer caching optimization
# - Rebuilds everything on code change
```

### Optimized Dockerfile (Multi-Stage Build)

```dockerfile
# ============================================
# STAGE 1: Dependencies
# ============================================
FROM node:18-alpine AS dependencies

# Add metadata
LABEL maintainer="devops@bank.com"
LABEL version="1.0"
LABEL description="Banking API - Dependencies Stage"

# Install build dependencies
RUN apk add --no-cache \
    python3 \
    make \
    g++ \
    && rm -rf /var/cache/apk/*

WORKDIR /app

# Copy package files (leverage layer caching)
COPY package*.json ./

# Install production dependencies only
RUN npm ci --only=production && npm cache clean --force

# ============================================
# STAGE 2: Build (if using TypeScript)
# ============================================
FROM dependencies AS build

# Install dev dependencies for build
RUN npm ci

# Copy source code
COPY . .

# Build application (if TypeScript)
# RUN npm run build

# ============================================
# STAGE 3: Production
# ============================================
FROM node:18-alpine AS production

# Install security updates
RUN apk upgrade --no-cache && \
    apk add --no-cache dumb-init && \
    rm -rf /var/cache/apk/*

# Create non-root user
RUN addgroup -g 1001 nodejs && \
    adduser -D -u 1001 -G nodejs nodejs

WORKDIR /app

# Copy dependencies from dependencies stage
COPY --from=dependencies --chown=nodejs:nodejs /app/node_modules ./node_modules

# Copy application code
COPY --chown=nodejs:nodejs src ./src
COPY --chown=nodejs:nodejs package*.json ./

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

# Use dumb-init to handle signals properly
ENTRYPOINT ["dumb-init", "--"]

# Start application
CMD ["node", "src/server.js"]

# Image size comparison:
# - Basic: 950 MB
# - Multi-stage: 180 MB
# - Savings: 770 MB (81% reduction)
```

### .dockerignore

```
# .dockerignore - Exclude files from Docker build context
node_modules
npm-debug.log
.git
.gitignore
.env
.env.local
.DS_Store
*.md
test/
coverage/
.vscode/
.idea/
dist/
build/
logs/
*.log
tmp/
```

### Development Dockerfile

```dockerfile
# Dockerfile.dev - For local development with hot-reload
FROM node:18-alpine

# Install nodemon for hot-reload
RUN npm install -g nodemon

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install all dependencies (including dev)
RUN npm install

# Copy source (will be overridden by volume mount)
COPY . .

EXPOSE 3000

# Start with nodemon for hot-reload
CMD ["nodemon", "src/server.js"]
```

### Application Code (src/server.js)

```javascript
/**
 * Banking API Server - Docker-optimized
 */

const express = require('express');
const app = express();

const PORT = process.env.PORT || 3000;
const NODE_ENV = process.env.NODE_ENV || 'development';

// Middleware
app.use(express.json());

// Health check endpoint (for Docker HEALTHCHECK)
app.get('/health', (req, res) => {
  res.status(200).json({
    status: 'healthy',
    uptime: process.uptime(),
    timestamp: Date.now(),
    environment: NODE_ENV
  });
});

// Readiness probe (checks DB connections)
app.get('/ready', async (req, res) => {
  try {
    // Check database connection
    // await db.ping();
    
    res.status(200).json({
      status: 'ready',
      services: {
        database: 'connected',
        cache: 'connected'
      }
    });
  } catch (error) {
    res.status(503).json({
      status: 'not ready',
      error: error.message
    });
  }
});

// Sample routes
app.get('/api/accounts', (req, res) => {
  res.json({
    accounts: [
      { id: 1, name: 'Checking', balance: 5000 },
      { id: 2, name: 'Savings', balance: 10000 }
    ]
  });
});

app.post('/api/transactions', (req, res) => {
  const { accountId, amount, type } = req.body;
  
  res.status(201).json({
    transactionId: `TXN-${Date.now()}`,
    accountId,
    amount,
    type,
    status: 'completed'
  });
});

// Start server
const server = app.listen(PORT, '0.0.0.0', () => {
  console.log(`🚀 Banking API running on port ${PORT}`);
  console.log(`📊 Environment: ${NODE_ENV}`);
  console.log(`🏥 Health check: http://localhost:${PORT}/health`);
});

// ============================================
// GRACEFUL SHUTDOWN (Important for Docker)
// ============================================

function gracefulShutdown(signal) {
  console.log(`\n${signal} received, shutting down gracefully...`);
  
  server.close(() => {
    console.log('✅ HTTP server closed');
    
    // Close database connections
    // db.close();
    
    console.log('✅ All connections closed');
    process.exit(0);
  });
  
  // Force shutdown after 10 seconds
  setTimeout(() => {
    console.error('❌ Forced shutdown after timeout');
    process.exit(1);
  }, 10000);
}

// Handle termination signals
process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));

// Handle uncaught exceptions
process.on('uncaughtException', (error) => {
  console.error('❌ Uncaught Exception:', error);
  process.exit(1);
});

process.on('unhandledRejection', (reason, promise) => {
  console.error('❌ Unhandled Rejection at:', promise, 'reason:', reason);
  process.exit(1);
});

module.exports = app;
```

### package.json

```json
{
  "name": "banking-api",
  "version": "1.0.0",
  "description": "Dockerized Banking API",
  "main": "src/server.js",
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js",
    "docker:build": "docker build -t banking-api:latest .",
    "docker:run": "docker run -p 3000:3000 banking-api:latest",
    "docker:dev": "docker build -f Dockerfile.dev -t banking-api:dev . && docker run -p 3000:3000 -v $(pwd):/app banking-api:dev"
  },
  "dependencies": {
    "express": "^4.18.0"
  },
  "devDependencies": {
    "nodemon": "^2.0.20"
  },
  "engines": {
    "node": ">=18.0.0",
    "npm": ">=9.0.0"
  }
}
```

---

## 💡 Example 2: Advanced Docker Patterns

### Multi-Stage Build with TypeScript

```dockerfile
# ============================================
# STAGE 1: Dependencies
# ============================================
FROM node:18-alpine AS dependencies

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# ============================================
# STAGE 2: Build TypeScript
# ============================================
FROM node:18-alpine AS build

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY tsconfig.json ./
COPY src ./src

# Compile TypeScript to JavaScript
RUN npm run build

# ============================================
# STAGE 3: Production
# ============================================
FROM node:18-alpine AS production

RUN apk add --no-cache dumb-init && \
    addgroup -g 1001 nodejs && \
    adduser -D -u 1001 -G nodejs nodejs

WORKDIR /app

# Copy dependencies
COPY --from=dependencies --chown=nodejs:nodejs /app/node_modules ./node_modules

# Copy compiled JavaScript
COPY --from=build --chown=nodejs:nodejs /app/dist ./dist
COPY --chown=nodejs:nodejs package*.json ./

USER nodejs

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s \
    CMD node -e "require('http').get('http://localhost:3000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/server.js"]
```

### Build Arguments and Environment Variables

```dockerfile
FROM node:18-alpine

# Build arguments (set at build time)
ARG NODE_ENV=production
ARG BUILD_DATE
ARG VERSION

# Environment variables (available at runtime)
ENV NODE_ENV=${NODE_ENV} \
    PORT=3000 \
    LOG_LEVEL=info

# Labels for metadata
LABEL org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.version="${VERSION}" \
      org.opencontainers.image.vendor="Bank Corp" \
      org.opencontainers.image.title="Banking API"

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

EXPOSE ${PORT}

CMD ["node", "server.js"]

# Build with arguments:
# docker build --build-arg NODE_ENV=production --build-arg VERSION=1.2.3 -t banking-api .

# Run with environment variables:
# docker run -e PORT=8080 -e LOG_LEVEL=debug banking-api
```

### Security Scanning

```dockerfile
# Use specific version (not 'latest' for security)
FROM node:18.17.0-alpine3.18

# Security: Install security updates
RUN apk upgrade --no-cache

# Security: Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001 -G nodejs

WORKDIR /app

# Security: Set file permissions
COPY --chown=nodejs:nodejs package*.json ./

USER nodejs

RUN npm ci --only=production

COPY --chown=nodejs:nodejs . .

# Security: Read-only root filesystem
# docker run --read-only --tmpfs /tmp banking-api

EXPOSE 3000

CMD ["node", "server.js"]
```

---

## 🎯 Docker Best Practices

### 1. Layer Caching Optimization

```dockerfile
# ❌ Bad: Copies everything first
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN npm install
# Result: Cache invalidated on ANY file change

# ✅ Good: Copies package.json first
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
# Result: Cache reused if package.json unchanged
```

### 2. Image Size Reduction

```dockerfile
# Use Alpine base (5x smaller)
FROM node:18-alpine  # 180 MB
# vs
FROM node:18         # 950 MB

# Multi-stage build (remove build dependencies)
FROM node:18-alpine AS build
RUN npm install && npm run build

FROM node:18-alpine AS production
COPY --from=build /app/dist ./dist
# Result: Only production artifacts in final image

# Clean up in same layer
RUN npm install && npm cache clean --force && rm -rf /tmp/*
```

### 3. Security Hardening

```dockerfile
# 1. Use specific versions
FROM node:18.17.0-alpine3.18

# 2. Security updates
RUN apk upgrade --no-cache

# 3. Non-root user
RUN adduser -D nodejs
USER nodejs

# 4. No secrets in image
# Use: docker run -e DB_PASSWORD=secret
# NOT: ENV DB_PASSWORD=secret

# 5. Minimal attack surface
RUN apk add --no-cache dumb-init
# Don't install unnecessary packages

# 6. Read-only filesystem
# docker run --read-only --tmpfs /tmp
```

### 4. Development vs Production

```bash
# Development: Fast rebuilds, debugging
docker build -f Dockerfile.dev -t app:dev .
docker run -v $(pwd):/app -p 3000:3000 app:dev

# Production: Optimized, secure
docker build -t app:prod .
docker run -d -p 3000:3000 --restart=unless-stopped app:prod
```

### 5. Docker Build Commands

```bash
# Build image
docker build -t banking-api:latest .

# Build with custom Dockerfile
docker build -f Dockerfile.prod -t banking-api:prod .

# Build with arguments
docker build --build-arg NODE_ENV=production -t banking-api .

# Build with no cache
docker build --no-cache -t banking-api .

# Multi-platform build
docker buildx build --platform linux/amd64,linux/arm64 -t banking-api .

# Tag image
docker tag banking-api:latest banking-api:v1.2.3

# Push to registry
docker push banking-api:v1.2.3
```

### 6. Docker Run Commands

```bash
# Basic run
docker run -p 3000:3000 banking-api

# Detached mode
docker run -d -p 3000:3000 --name banking-api banking-api

# With environment variables
docker run -e NODE_ENV=production -e PORT=8080 banking-api

# With volume mount (persistence)
docker run -v /data:/app/data banking-api

# With restart policy
docker run --restart=unless-stopped banking-api

# With resource limits
docker run --memory=512m --cpus=2 banking-api

# Interactive shell
docker run -it banking-api sh

# Override entrypoint
docker run --entrypoint sh banking-api
```

### 7. Docker Management Commands

```bash
# List images
docker images

# List containers
docker ps           # Running
docker ps -a        # All

# View logs
docker logs banking-api
docker logs -f banking-api  # Follow

# Execute command in container
docker exec -it banking-api sh
docker exec banking-api node -v

# Inspect container
docker inspect banking-api

# View resource usage
docker stats banking-api

# Stop/Start/Restart
docker stop banking-api
docker start banking-api
docker restart banking-api

# Remove container
docker rm banking-api

# Remove image
docker rmi banking-api:latest

# Clean up
docker system prune -a  # Remove all unused images
docker volume prune     # Remove unused volumes
```

---

## 📚 Common Interview Questions

### Q1: Explain multi-stage Docker builds

**Answer**:
Multi-stage builds use multiple `FROM` statements in one Dockerfile. Each stage can copy artifacts from previous stages, reducing final image size by excluding build dependencies.

```dockerfile
FROM node:18 AS build
RUN npm install && npm run build

FROM node:18-alpine AS production
COPY --from=build /app/dist ./dist
# Result: Final image doesn't include build tools
```

### Q2: How do you optimize Docker image size?

**Answer**:
1. Use Alpine base images (smaller)
2. Multi-stage builds (remove build dependencies)
3. `.dockerignore` (exclude unnecessary files)
4. Combine RUN commands (reduce layers)
5. `npm ci --only=production` (no dev deps)
6. Clean cache: `npm cache clean --force`

### Q3: What's the difference between CMD and ENTRYPOINT?

**Answer**:
- **CMD**: Default command, can be overridden
- **ENTRYPOINT**: Main command, args appended

```dockerfile
ENTRYPOINT ["node"]
CMD ["server.js"]

# docker run myapp           → node server.js
# docker run myapp app.js    → node app.js
```

### Q4: How do you handle secrets in Docker?

**Answer**:
**Never** put secrets in Dockerfile or image!

Methods:
1. **Environment variables**: `docker run -e DB_PASSWORD=secret`
2. **Docker secrets** (Swarm): `docker secret create`
3. **Volume mounts**: Mount secrets from file
4. **AWS Secrets Manager**: Fetch at runtime
5. **Build secrets** (Buildkit): `RUN --mount=type=secret`

### Q5: Explain Docker layer caching

**Answer**:
Docker caches each layer (instruction). If a layer hasn't changed, Docker reuses the cached layer.

Optimize by:
- Put frequently changing code last
- Copy `package.json` before `COPY . .`
- Combine commands to reduce layers

---

## ✅ Summary & Key Takeaways

### Core Concepts

1. **Dockerfile**: Instructions to build image
2. **Multi-stage**: Optimize size by copying only needed artifacts
3. **Layer Caching**: Order matters for fast rebuilds
4. **Alpine**: Use Alpine Linux for smaller images

### Production Checklist

```dockerfile
✅ Production-Ready Dockerfile:
  - [ ] Multi-stage build
  - [ ] Alpine base image
  - [ ] Non-root user
  - [ ] Security updates (apk upgrade)
  - [ ] .dockerignore file
  - [ ] Health check
  - [ ] Graceful shutdown (SIGTERM)
  - [ ] Specific versions (not 'latest')
  - [ ] Minimal dependencies
  - [ ] Environment variables (no secrets)
  - [ ] Labels for metadata
  - [ ] dumb-init for signal handling
```

### Banking Applications

1. **Microservices**: Each service in container
2. **CI/CD**: Automated builds and deployments
3. **Consistency**: Same environment dev → prod
4. **Scalability**: Spin up/down containers easily

### Size Optimization Results

```
Basic Dockerfile:        950 MB
+ Alpine:                180 MB (81% smaller)
+ Multi-stage:           120 MB (87% smaller)
+ Optimizations:          80 MB (92% smaller)
```

---

**Status**: ✅ Complete with production-ready Dockerfiles, multi-stage builds, security hardening, and optimization techniques!
