# Docker Containerization

## Overview

Docker revolutionizes application deployment by packaging applications and their dependencies into lightweight, portable containers. Containers ensure consistency across development, testing, and production environments, eliminate "works on my machine" problems, and enable efficient resource utilization. Docker has become essential for modern web application deployment and microservices architectures.

## Core Concepts

### Docker Architecture

```
┌─────────────────────────────────────┐
│        Docker Client (CLI)          │
└──────────────┬──────────────────────┘
               │ Docker API
               ▼
┌──────────────────────────────────────┐
│         Docker Daemon                │
│  ┌────────────────────────────────┐  │
│  │   Images (Templates)           │  │
│  │  - Node:18                     │  │
│  │  - Nginx:alpine                │  │
│  └────────────────────────────────┘  │
│  ┌────────────────────────────────┐  │
│  │   Containers (Running)         │  │
│  │  - my-app-1                    │  │
│  │  - my-app-2                    │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│         Docker Registry              │
│          (Docker Hub, ECR)           │
└──────────────────────────────────────┘
```

## Basic Dockerfile

### React Application

```dockerfile
# Dockerfile
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY . .

# Build application
RUN npm run build

# Production stage
FROM nginx:alpine

# Copy built assets from builder
COPY --from=builder /app/dist /usr/share/nginx/html

# Copy nginx configuration
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### Multi-stage Build Benefits

```dockerfile
# Multi-stage Dockerfile with optimization
FROM node:18-alpine AS dependencies
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM node:18-alpine AS builder
WORKDIR /app
COPY --from=dependencies /app/node_modules ./node_modules
COPY . .
RUN npm run build && npm run test

FROM nginx:alpine AS production
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --quiet --tries=1 --spider http://localhost/ || exit 1
CMD ["nginx", "-g", "daemon off;"]

# Image size comparison:
# Single-stage: ~1.5GB
# Multi-stage: ~25MB (60x smaller!)
```

## Common Dockerfile Patterns

### Node.js Backend

```dockerfile
FROM node:18-alpine

# Create app directory
WORKDIR /usr/src/app

# Install app dependencies
COPY package*.json ./
RUN npm ci --only=production

# Bundle app source
COPY . .

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

USER nodejs

EXPOSE 3000

CMD ["node", "server.js"]
```

### Next.js Application

```dockerfile
FROM node:18-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM node:18-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED 1
RUN npm run build

FROM node:18-alpine AS runner
WORKDIR /app
ENV NODE_ENV production
ENV NEXT_TELEMETRY_DISABLED 1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
ENV PORT 3000
CMD ["node", "server.js"]
```

## Docker Compose

### Complete Stack

```yaml
# docker-compose.yml
version: '3.8'

services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
      args:
        - NODE_ENV=production
    ports:
      - "3000:80"
    environment:
      - REACT_APP_API_URL=http://backend:5000
    depends_on:
      - backend
    networks:
      - app-network
    restart: unless-stopped

  backend:
    build: ./backend
    ports:
      - "5000:5000"
    environment:
      - DATABASE_URL=postgresql://postgres:password@db:5432/myapp
      - REDIS_URL=redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - app-network
    volumes:
      - ./backend:/app
      - /app/node_modules
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=myapp
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - app-network
    command: redis-server --appendonly yes

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/ssl:/etc/nginx/ssl
    depends_on:
      - frontend
      - backend
    networks:
      - app-network

volumes:
  postgres-data:
  redis-data:

networks:
  app-network:
    driver: bridge
```

## Optimization Techniques

### Layer Caching

```dockerfile
# Bad - Invalidates cache on any file change
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build

# Good - Optimized layer caching
FROM node:18-alpine
WORKDIR /app

# Copy only package files first
COPY package*.json ./
RUN npm ci

# Copy source code last
COPY . .
RUN npm run build
```

### .dockerignore

```bash
# .dockerignore
node_modules
npm-debug.log
.git
.gitignore
.env
.env.*
dist
build
coverage
.DS_Store
*.md
.vscode
.idea
```

## Environment Configuration

```dockerfile
# Dockerfile with build args
FROM node:18-alpine AS builder

ARG NODE_ENV=production
ARG API_URL
ARG API_KEY

ENV NODE_ENV=${NODE_ENV}
ENV VITE_API_URL=${API_URL}
ENV VITE_API_KEY=${API_KEY}

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build
```

```bash
# Build with arguments
docker build \
  --build-arg API_URL=https://api.production.com \
  --build-arg API_KEY=prod-key-123 \
  -t my-app:latest .

# Run with environment variables
docker run -d \
  -e NODE_ENV=production \
  -e DATABASE_URL=postgresql://localhost/mydb \
  -p 3000:3000 \
  my-app:latest
```

## Common Mistakes

1. **Using latest tag in production** - unpredictable versions
2. **Running as root** - security risk
3. **Large image sizes** - slow deployments
4. **Not using .dockerignore** - context bloat
5. **Hardcoded secrets** - exposed credentials
6. **No health checks** - unhealthy containers running
7. **Not cleaning up** - disk space issues
8. **Copying node_modules** - unnecessary files

## Best Practices

1. Use specific version tags (node:18-alpine, not node:latest)
2. Implement multi-stage builds for smaller images
3. Use .dockerignore to exclude unnecessary files
4. Run containers as non-root user
5. Implement health checks
6. Use environment variables for configuration
7. Minimize layers and combine RUN commands
8. Cache dependencies before copying source code
9. Use alpine images when possible
10. Scan images for vulnerabilities

## Key Takeaways

1. Docker ensures consistency across environments
2. Multi-stage builds dramatically reduce image size
3. Layer caching speeds up builds
4. Docker Compose manages multi-container applications
5. Security best practices are critical
6. Optimization reduces deployment time
7. Environment variables enable configuration
8. Health checks ensure reliability
9. .dockerignore prevents bloat
10. Proper image tagging is essential

## Resources

- [Docker Documentation](https://docs.docker.com/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Dockerfile Reference](https://docs.docker.com/engine/reference/builder/)
- [Docker Compose](https://docs.docker.com/compose/)
- [Docker Hub](https://hub.docker.com/)
