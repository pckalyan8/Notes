# Docker Advanced Guide
## From Intermediate to Advanced

---

## Table of Contents
1. [Multi-Stage Builds](#multi-stage-builds)
2. [Docker Compose](#docker-compose)
3. [Advanced Networking](#advanced-networking)
4. [Docker Security](#docker-security)
5. [Image Optimization](#image-optimization)
6. [Health Checks](#health-checks)
7. [Container Resource Management](#container-resource-management)
8. [Docker Registry](#docker-registry)
9. [CI/CD Integration](#cicd-integration)
10. [Logging and Monitoring](#logging-and-monitoring)
11. [Orchestration Introduction](#orchestration-introduction)
12. [Best Practices](#best-practices)
13. [Advanced Examples](#advanced-examples)
14. [Troubleshooting](#troubleshooting)

---

## Multi-Stage Builds

Multi-stage builds allow you to create **smaller, more secure production images** by separating build and runtime environments.

### Problem: Large Images

Traditional Dockerfile:
```dockerfile
FROM node:16
WORKDIR /app
COPY package*.json ./
RUN npm install  # Installs dev dependencies
COPY . .
RUN npm run build  # Build artifacts
CMD ["node", "dist/app.js"]

# Result: Large image with build tools and dev dependencies
```

### Solution: Multi-Stage Build

```dockerfile
# Stage 1: Build
FROM node:16 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install  # Includes dev dependencies
COPY . .
RUN npm run build  # Creates dist/

# Stage 2: Production
FROM node:16-alpine  # Smaller base
WORKDIR /app
COPY package*.json ./
RUN npm install --production  # Only prod dependencies
COPY --from=builder /app/dist ./dist  # Copy only build artifacts
CMD ["node", "dist/app.js"]

# Result: Much smaller image, no build tools
```

### How It Works:

```
Stage 1 (builder)           Stage 2 (final)
┌──────────────────┐       ┌──────────────────┐
│ Build tools      │       │                  │
│ Dev dependencies │──────>│ Copy artifacts   │
│ Source code      │       │ Prod dependencies│
│ Build artifacts  │       │ Runtime only     │
└──────────────────┘       └──────────────────┘
   Discarded                   Final image
```

### Advanced Example: Go Application

```dockerfile
# Stage 1: Build
FROM golang:1.20 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o main .

# Stage 2: Production
FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/main .
EXPOSE 8080
CMD ["./main"]

# Go binary: ~50MB vs ~800MB with full Go image!
```

### Python Example with Testing:

```dockerfile
# Stage 1: Dependencies
FROM python:3.9 AS dependencies
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Stage 2: Testing
FROM dependencies AS testing
COPY . .
RUN pytest tests/

# Stage 3: Production
FROM python:3.9-slim
WORKDIR /app
COPY --from=dependencies /usr/local/lib/python3.9/site-packages /usr/local/lib/python3.9/site-packages
COPY . .
CMD ["python", "app.py"]
```

### Build Specific Stage:

```bash
# Build and stop at testing stage
docker build --target testing -t myapp:test .

# Build production image (default: last stage)
docker build -t myapp:prod .

# Build with custom build args
docker build --build-arg VERSION=1.0 -t myapp:1.0 .
```

### Benefits:
- **Smaller images** (only runtime dependencies)
- **Faster deployments** (less data to transfer)
- **More secure** (no build tools in production)
- **Cleaner separation** (build vs runtime)

---

## Docker Compose

Docker Compose allows you to define and run **multi-container applications** using a YAML file.

### Why Docker Compose?

Without Compose:
```bash
# Create network
docker network create app-network

# Run database
docker run -d --name db --network app-network \
  -e POSTGRES_PASSWORD=secret \
  -v db_data:/var/lib/postgresql/data \
  postgres:13

# Run Redis
docker run -d --name redis --network app-network redis:alpine

# Run web app
docker run -d --name web --network app-network \
  -p 8000:8000 \
  -e DATABASE_URL=postgresql://postgres:secret@db:5432/mydb \
  -e REDIS_URL=redis://redis:6379 \
  myapp:latest

# Hard to manage, easy to make mistakes!
```

With Compose:
```bash
docker compose up
# Done! All services started
```

### Docker Compose File Structure:

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  # Service definitions

networks:
  # Network definitions

volumes:
  # Volume definitions
```

### Basic Example:

```yaml
version: '3.8'

services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html
    networks:
      - app-network

  database:
    image: postgres:13
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: mydb
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  db_data:
```

### Common Compose Commands:

```bash
# Start services (detached)
docker compose up -d

# Start specific services
docker compose up web database

# Stop services
docker compose stop

# Stop and remove containers, networks
docker compose down

# Stop and remove everything (including volumes)
docker compose down -v

# View logs
docker compose logs

# Follow logs
docker compose logs -f

# View logs for specific service
docker compose logs web

# List running services
docker compose ps

# Execute command in service
docker compose exec web bash

# Restart services
docker compose restart

# Build images
docker compose build

# Build without cache
docker compose build --no-cache

# Pull images
docker compose pull
```

### Complete Application Example:

**Directory structure:**
```
myapp/
├── docker-compose.yml
├── web/
│   ├── Dockerfile
│   ├── app.py
│   └── requirements.txt
└── nginx/
    └── nginx.conf
```

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  # Web Application
  web:
    build:
      context: ./web
      dockerfile: Dockerfile
    container_name: flask_app
    environment:
      - DATABASE_URL=postgresql://postgres:secret@db:5432/appdb
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      - db
      - redis
    networks:
      - backend
    volumes:
      - ./web:/app
    restart: unless-stopped

  # Nginx Reverse Proxy
  nginx:
    image: nginx:alpine
    container_name: nginx_proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - web
    networks:
      - frontend
      - backend
    restart: unless-stopped

  # PostgreSQL Database
  db:
    image: postgres:13
    container_name: postgres_db
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - backend
    restart: unless-stopped

  # Redis Cache
  redis:
    image: redis:alpine
    container_name: redis_cache
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    networks:
      - backend
    restart: unless-stopped

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge

volumes:
  postgres_data:
  redis_data:
```

### Service Configuration Options:

```yaml
services:
  myapp:
    # Build from Dockerfile
    build:
      context: ./app
      dockerfile: Dockerfile.prod
      args:
        VERSION: "1.0"
    
    # Or use existing image
    image: myapp:latest
    
    # Container name
    container_name: my_container
    
    # Port mapping
    ports:
      - "8080:80"          # HOST:CONTAINER
      - "443:443"
      - "127.0.0.1:3306:3306"  # Bind to specific IP
    
    # Environment variables
    environment:
      NODE_ENV: production
      API_KEY: secret
    
    # Or from file
    env_file:
      - .env
      - .env.production
    
    # Volumes
    volumes:
      - ./app:/usr/src/app           # Bind mount
      - app_data:/data               # Named volume
      - /var/run/docker.sock:/var/run/docker.sock  # Host path
    
    # Networks
    networks:
      - frontend
      - backend
    
    # Dependencies
    depends_on:
      - db
      - redis
    
    # Restart policy
    restart: unless-stopped  # or: no, always, on-failure
    
    # Health check
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    
    # Resource limits
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    
    # Command override
    command: python app.py --host 0.0.0.0
    
    # Entrypoint override
    entrypoint: /usr/local/bin/docker-entrypoint.sh
    
    # Working directory
    working_dir: /app
    
    # User
    user: "1000:1000"
    
    # Labels
    labels:
      com.example.description: "My app"
      com.example.version: "1.0"
```

### Environment Variables:

**Option 1: Inline**
```yaml
services:
  web:
    environment:
      - NODE_ENV=production
      - API_KEY=secret
```

**Option 2: From file (.env)**
```env
NODE_ENV=production
API_KEY=secret
DATABASE_URL=postgresql://postgres:secret@db:5432/mydb
```

```yaml
services:
  web:
    env_file:
      - .env
```

**Option 3: Substitution**
```yaml
services:
  web:
    image: "myapp:${VERSION:-latest}"
    environment:
      - API_KEY=${API_KEY}
```

```bash
export VERSION=1.0
export API_KEY=secret
docker compose up
```

### Depends On with Health Checks:

```yaml
version: '3.8'

services:
  web:
    image: myapp:latest
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

  db:
    image: postgres:13
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:alpine
```

### Scaling Services:

```bash
# Scale specific service
docker compose up -d --scale web=3

# Scale multiple services
docker compose up -d --scale web=3 --scale worker=5
```

```yaml
services:
  web:
    image: myapp:latest
    deploy:
      replicas: 3  # For Swarm mode
```

### Profiles:

Run different sets of services for different scenarios:

```yaml
version: '3.8'

services:
  web:
    image: nginx:alpine
    # Always runs

  database:
    image: postgres:13
    # Always runs

  test_db:
    image: postgres:13
    profiles:
      - testing
    # Only runs with: docker compose --profile testing up

  monitoring:
    image: prometheus
    profiles:
      - monitoring
    # Only runs with: docker compose --profile monitoring up
```

```bash
# Run default services
docker compose up

# Run with testing profile
docker compose --profile testing up

# Run with multiple profiles
docker compose --profile testing --profile monitoring up
```

### Override Files:

**docker-compose.yml** (base):
```yaml
version: '3.8'

services:
  web:
    image: myapp:latest
    environment:
      - NODE_ENV=production
```

**docker-compose.override.yml** (development overrides):
```yaml
version: '3.8'

services:
  web:
    volumes:
      - ./app:/usr/src/app  # Live code reload
    environment:
      - NODE_ENV=development
      - DEBUG=true
    command: npm run dev
```

```bash
# Automatically uses both files
docker compose up

# Use specific override file
docker compose -f docker-compose.yml -f docker-compose.prod.yml up
```

---

## Advanced Networking

### Network Types Deep Dive:

```
┌────────────────────────────────────────────────────┐
│                 Docker Host                        │
│                                                    │
│  ┌──────────────────────────────────────────────┐ │
│  │          Bridge Network (isolated)           │ │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐      │ │
│  │  │  Web    │  │  API    │  │   DB    │      │ │
│  │  └─────────┘  └─────────┘  └─────────┘      │ │
│  │       ↑            ↑            ↑            │ │
│  └───────┼────────────┼────────────┼────────────┘ │
│          │            │            │              │
│  ┌───────┼────────────┼────────────┼────────────┐ │
│  │       ↓            ↓            ↓            │ │
│  │        Docker Bridge (docker0)              │ │
│  └──────────────────────┬─────────────────────────┘ │
│                         │                          │
│                    Host Network                    │
└─────────────────────────┼──────────────────────────┘
                          │
                     Internet
```

### Custom Bridge Networks:

```bash
# Create network with custom subnet
docker network create \
  --driver bridge \
  --subnet 172.20.0.0/16 \
  --gateway 172.20.0.1 \
  my_network

# Create network with custom options
docker network create \
  --driver bridge \
  --opt "com.docker.network.bridge.name"="br_custom" \
  --opt "com.docker.network.bridge.enable_ip_masquerade"="true" \
  custom_network

# Inspect network
docker network inspect my_network

# Run container with specific IP
docker run -d \
  --network my_network \
  --ip 172.20.0.10 \
  --name web \
  nginx
```

### Container DNS:

Containers on same user-defined network can resolve each other by name:

```bash
# Create network
docker network create app_net

# Run containers
docker run -d --name db --network app_net postgres:13
docker run -d --name redis --network app_net redis:alpine
docker run -d --name web --network app_net myapp:latest

# From web container, can access:
# postgresql://db:5432
# redis://redis:6379
```

### Network Aliases:

```bash
# Single container, multiple aliases
docker run -d \
  --network app_net \
  --network-alias database \
  --network-alias db \
  --network-alias postgres \
  --name postgres_server \
  postgres:13

# Can be accessed as: database, db, postgres
```

In Compose:
```yaml
services:
  database:
    image: postgres:13
    networks:
      app_net:
        aliases:
          - db
          - postgres
          - database
```

### Multiple Networks:

```yaml
version: '3.8'

services:
  web:
    image: nginx:alpine
    networks:
      - frontend  # Public-facing

  api:
    image: myapi:latest
    networks:
      - frontend  # Accessible by web
      - backend   # Can access database

  database:
    image: postgres:13
    networks:
      - backend   # Isolated, only API can access

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # No external access
```

Network isolation:
```
              Internet
                 │
                 ↓
         ┌──────────────┐
         │     Web      │ ← frontend network
         └──────────────┘
                 │
                 ↓
         ┌──────────────┐
         │     API      │ ← frontend + backend networks
         └──────────────┘
                 │
                 ↓
         ┌──────────────┐
         │   Database   │ ← backend network only (isolated)
         └──────────────┘
```

### MacVLAN Networks:

Assign MAC addresses to containers, making them appear as physical devices:

```bash
# Create macvlan network
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  macvlan_net

# Run container with dedicated IP on physical network
docker run -d \
  --network macvlan_net \
  --ip 192.168.1.100 \
  --name web \
  nginx
```

### Container Links (Legacy):

```bash
# Old way (deprecated)
docker run -d --name db postgres:13
docker run -d --link db:database myapp:latest

# Use user-defined networks instead!
```

### Port Publishing Modes:

```bash
# Publish to all interfaces
docker run -d -p 8080:80 nginx

# Publish to specific interface
docker run -d -p 127.0.0.1:8080:80 nginx

# Publish range
docker run -d -p 8080-8090:80 nginx

# Publish UDP
docker run -d -p 53:53/udp dns-server

# Publish all exposed ports to random ports
docker run -d -P nginx
```

### Network Troubleshooting:

```bash
# Inspect container networking
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web

# Test connectivity between containers
docker exec web ping -c 2 db

# Check DNS resolution
docker exec web nslookup db

# View network interfaces inside container
docker exec web ip addr

# Capture network traffic
docker exec web tcpdump -i eth0

# Test port connectivity
docker exec web nc -zv db 5432
```

---

## Docker Security

### Security Principles:

```
┌─────────────────────────────────────────┐
│        Docker Security Layers           │
├─────────────────────────────────────────┤
│  1. Host OS Security                    │
│  2. Docker Daemon Security              │
│  3. Image Security                      │
│  4. Container Runtime Security          │
│  5. Network Security                    │
│  6. Secrets Management                  │
└─────────────────────────────────────────┘
```

### 1. Run as Non-Root User:

**Bad:**
```dockerfile
FROM ubuntu:20.04
COPY app.py /app/
CMD ["python3", "/app/app.py"]
# Runs as root! ❌
```

**Good:**
```dockerfile
FROM ubuntu:20.04

# Create non-root user
RUN useradd -m -u 1000 appuser && \
    mkdir -p /app && \
    chown -R appuser:appuser /app

WORKDIR /app
COPY --chown=appuser:appuser app.py .

# Switch to non-root user
USER appuser

CMD ["python3", "app.py"]
# Runs as appuser ✅
```

### 2. Use Minimal Base Images:

```dockerfile
# Instead of full image
FROM ubuntu:20.04  # 72MB

# Use slim or alpine
FROM python:3.9-slim  # 122MB
FROM python:3.9-alpine  # 46MB
FROM gcr.io/distroless/python3  # 52MB (no shell, even more secure)
```

### 3. Scan Images for Vulnerabilities:

```bash
# Using Docker Scout (built-in)
docker scout cves myapp:latest

# Using Trivy
docker run aquasec/trivy image myapp:latest

# Using Snyk
snyk container test myapp:latest

# Using Clair
docker run -p 5432:5432 -d postgres
docker run -p 6060:6060 -d quay.io/coreos/clair
```

### 4. Don't Include Secrets in Images:

**Bad:**
```dockerfile
FROM node:16
ENV API_KEY=secret123  # ❌ Secret in image!
COPY . .
```

**Good:**
```dockerfile
FROM node:16
COPY . .
# Provide secrets at runtime
```

```bash
# Pass as environment variable
docker run -e API_KEY=secret123 myapp

# Or use Docker secrets (Swarm)
echo "secret123" | docker secret create api_key -
docker service create --secret api_key myapp
```

### 5. Use Docker Secrets:

```bash
# Create secret
echo "mypassword" | docker secret create db_password -

# Use in service
docker service create \
  --name myapp \
  --secret db_password \
  myapp:latest

# Access in container at: /run/secrets/db_password
```

With Compose:
```yaml
version: '3.8'

services:
  web:
    image: myapp:latest
    secrets:
      - db_password
      - api_key

secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    external: true  # Already created with docker secret create
```

### 6. Read-Only Containers:

```bash
# Make filesystem read-only
docker run -d --read-only nginx

# With temporary writable directories
docker run -d \
  --read-only \
  --tmpfs /tmp \
  --tmpfs /var/run \
  nginx
```

### 7. Limit Container Capabilities:

```bash
# Drop all capabilities, add only needed ones
docker run -d \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  nginx

# Default capabilities
docker run -d nginx

# No capabilities (most secure)
docker run -d --cap-drop ALL myapp
```

### 8. Use Security Options:

```bash
# AppArmor
docker run -d \
  --security-opt apparmor=docker-default \
  nginx

# SELinux
docker run -d \
  --security-opt label=level:s0:c100,c200 \
  nginx

# Seccomp profile
docker run -d \
  --security-opt seccomp=path/to/profile.json \
  nginx

# No new privileges
docker run -d \
  --security-opt no-new-privileges \
  nginx
```

### 9. Resource Limits:

```bash
# Prevent DoS attacks
docker run -d \
  --memory="512m" \
  --memory-swap="512m" \
  --cpus="0.5" \
  --pids-limit=100 \
  nginx
```

### 10. Network Isolation:

```bash
# No network access
docker run -d --network none myapp

# Custom isolated network
docker network create --internal isolated_net
docker run -d --network isolated_net myapp
```

### 11. Content Trust:

```bash
# Enable Docker Content Trust
export DOCKER_CONTENT_TRUST=1

# Only pull signed images
docker pull nginx:latest

# Disable for specific command
DOCKER_CONTENT_TRUST=0 docker pull untrusted:latest
```

### 12. Secure Dockerfile Practices:

```dockerfile
# ✅ Good practices
FROM python:3.9-slim AS base

# Don't run as root
RUN useradd -m appuser

# Use specific versions
RUN pip install flask==2.3.0

# Multi-stage to reduce attack surface
FROM base AS builder
COPY requirements.txt .
RUN pip install --user -r requirements.txt

FROM base AS production
COPY --from=builder /home/appuser/.local /home/appuser/.local
COPY --chown=appuser:appuser app.py .

# Remove unnecessary tools
RUN apt-get purge -y --auto-remove && \
    rm -rf /var/lib/apt/lists/*

USER appuser
HEALTHCHECK CMD curl -f http://localhost:8000/health || exit 1
CMD ["python", "app.py"]
```

### Security Checklist:

```
□ Run containers as non-root user
□ Use minimal base images (alpine, distroless)
□ Scan images for vulnerabilities
□ Don't embed secrets in images
□ Use read-only filesystems where possible
□ Drop unnecessary capabilities
□ Set resource limits
□ Use network isolation
□ Enable content trust
□ Keep Docker and images updated
□ Use security scanning in CI/CD
□ Implement least privilege principle
□ Regular security audits
```

---

## Image Optimization

### Goals:
- **Smaller size** → Faster transfers, less storage
- **Fewer layers** → Better performance
- **Less vulnerabilities** → More secure

### 1. Choose Right Base Image:

```dockerfile
# Comparison
FROM ubuntu:20.04         # 72MB
FROM python:3.9           # 885MB
FROM python:3.9-slim      # 122MB
FROM python:3.9-alpine    # 46MB
FROM gcr.io/distroless/python3  # 52MB
```

### 2. Minimize Layers:

**Bad (many layers):**
```dockerfile
FROM python:3.9-slim
RUN apt-get update
RUN apt-get install -y gcc
RUN apt-get install -y g++
RUN apt-get install -y make
RUN rm -rf /var/lib/apt/lists/*
# 5 layers!
```

**Good (combined):**
```dockerfile
FROM python:3.9-slim
RUN apt-get update && \
    apt-get install -y gcc g++ make && \
    rm -rf /var/lib/apt/lists/*
# 1 layer!
```

### 3. Order Matters (Leverage Cache):

**Bad (cache busted often):**
```dockerfile
FROM python:3.9-slim
COPY . /app              # Changes often
RUN pip install -r requirements.txt  # Re-runs every time!
```

**Good (cache efficient):**
```dockerfile
FROM python:3.9-slim
COPY requirements.txt /app/  # Changes rarely
RUN pip install -r requirements.txt  # Cached!
COPY . /app              # Changes often, but deps cached
```

Layer caching:
```
Build 1:                Build 2 (code change):
├─ Base image          ├─ Base image (cached ✓)
├─ requirements.txt    ├─ requirements.txt (cached ✓)
├─ pip install         ├─ pip install (cached ✓)
└─ Copy code           └─ Copy code (rebuilt)
```

### 4. Use .dockerignore:

**.dockerignore:**
```
# Exclude from build context
node_modules
.git
.gitignore
README.md
.env
*.log
coverage/
dist/
build/
.DS_Store
__pycache__/
*.pyc
```

Benefits:
- Faster builds (less data to send)
- Smaller images
- No sensitive files accidentally included

### 5. Multi-Stage Builds:

```dockerfile
# Build stage
FROM node:16 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Production stage
FROM node:16-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY package*.json ./
RUN npm install --production
CMD ["node", "dist/server.js"]

# Result: ~150MB vs ~900MB
```

### 6. Cleanup in Same Layer:

**Bad:**
```dockerfile
RUN apt-get update
RUN apt-get install -y build-tools
RUN make install
RUN apt-get remove -y build-tools  # Doesn't reduce size!
```

**Good:**
```dockerfile
RUN apt-get update && \
    apt-get install -y build-tools && \
    make install && \
    apt-get remove -y build-tools && \
    rm -rf /var/lib/apt/lists/*
```

### 7. Use Specific Package Versions:

```dockerfile
# Bad (unpredictable)
RUN apt-get install python3

# Good (reproducible)
RUN apt-get install python3=3.9.2-1ubuntu0.1
```

### 8. Minimize Installed Packages:

```dockerfile
# Install only what you need
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        python3 \
        python3-pip && \
    rm -rf /var/lib/apt/lists/*
```

### 9. Use COPY Instead of ADD:

```dockerfile
# ADD has extra features (auto-extraction, URLs)
# Use COPY for simple file copying
COPY app.py /app/  # ✅ Preferred
ADD app.py /app/   # ❌ Unless you need ADD features
```

### 10. Optimize Python Images:

```dockerfile
FROM python:3.9-slim

# Don't create .pyc files
ENV PYTHONDONTWRITEBYTECODE=1
# Unbuffered output
ENV PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

### Image Size Comparison:

```bash
# Check image size
docker images myapp

# See layer sizes
docker history myapp:latest

# Analyze with dive
docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  wagoodman/dive:latest myapp:latest
```

### Optimization Checklist:

```
□ Use minimal base images (alpine, slim, distroless)
□ Combine RUN commands
□ Order instructions by change frequency
□ Use .dockerignore
□ Use multi-stage builds
□ Clean up in same layer
□ Specify package versions
□ Install only necessary packages
□ Remove build dependencies
□ Don't include unnecessary files
□ Use COPY instead of ADD
□ Leverage build cache
```

---

## Health Checks

Health checks let Docker monitor container health and automatically restart unhealthy containers.

### Dockerfile Health Check:

```dockerfile
FROM nginx:alpine

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost/ || exit 1

# Options:
# --interval=DURATION  (default: 30s)
# --timeout=DURATION   (default: 30s)
# --start-period=DURATION (default: 0s, grace period)
# --retries=N          (default: 3)
```

### Health Check States:

```
Container Start
      │
      ↓
  [starting] ────────→ [healthy]
      │                    │
      │                    │
      ↓                    ↓
  [unhealthy] ←────────────┘
   (retries exceeded)
```

### Examples:

**HTTP endpoint:**
```dockerfile
HEALTHCHECK CMD curl -f http://localhost:8000/health || exit 1
```

**TCP connection:**
```dockerfile
HEALTHCHECK CMD nc -z localhost 5432 || exit 1
```

**Custom script:**
```dockerfile
COPY healthcheck.sh /usr/local/bin/
HEALTHCHECK CMD /usr/local/bin/healthcheck.sh
```

**Python script:**
```dockerfile
HEALTHCHECK CMD python -c "import requests; requests.get('http://localhost:8000/health')"
```

### Docker Compose Health Check:

```yaml
version: '3.8'

services:
  web:
    image: myapp:latest
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  database:
    image: postgres:13
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
```

### Depends on Health:

```yaml
services:
  app:
    image: myapp:latest
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

  db:
    image: postgres:13
    healthcheck:
      test: ["CMD", "pg_isready"]
      interval: 10s

  redis:
    image: redis:alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
```

### Check Container Health:

```bash
# View health status
docker ps
# STATUS column shows: (healthy) or (unhealthy)

# Inspect health
docker inspect --format='{{.State.Health.Status}}' container_name

# View health check logs
docker inspect --format='{{json .State.Health}}' container_name | jq

# Example output:
# {
#   "Status": "healthy",
#   "FailingStreak": 0,
#   "Log": [
#     {
#       "Start": "2024-01-15T10:00:00Z",
#       "End": "2024-01-15T10:00:01Z",
#       "ExitCode": 0,
#       "Output": "OK"
#     }
#   ]
# }
```

### Disable Health Check:

```dockerfile
# Disable health check from base image
HEALTHCHECK NONE
```

### Real-World Health Check Script:

**healthcheck.sh:**
```bash
#!/bin/bash
set -e

# Check HTTP endpoint
if ! curl -sf http://localhost:8000/health > /dev/null; then
  echo "Health endpoint failed"
  exit 1
fi

# Check database connection
if ! psql -h localhost -U postgres -c "SELECT 1" > /dev/null 2>&1; then
  echo "Database connection failed"
  exit 1
fi

# Check Redis connection
if ! redis-cli ping > /dev/null 2>&1; then
  echo "Redis connection failed"
  exit 1
fi

echo "All checks passed"
exit 0
```

---

## Container Resource Management

Prevent containers from consuming all host resources.

### Memory Limits:

```bash
# Hard memory limit
docker run -d --memory="512m" nginx

# Memory + swap limit
docker run -d --memory="512m" --memory-swap="1g" nginx

# Memory reservation (soft limit)
docker run -d --memory-reservation="256m" nginx

# OOM (Out of Memory) killer disable
docker run -d --memory="512m" --oom-kill-disable nginx
```

### CPU Limits:

```bash
# CPU shares (relative weight)
docker run -d --cpu-shares=512 nginx  # Default: 1024

# CPU cores
docker run -d --cpus="1.5" nginx  # 1.5 cores

# CPU period and quota
docker run -d --cpu-period=100000 --cpu-quota=50000 nginx  # 50% of one core

# Pin to specific CPUs
docker run -d --cpuset-cpus="0,1" nginx  # Use CPU 0 and 1
```

### Disk I/O Limits:

```bash
# Block I/O weight
docker run -d --blkio-weight=500 nginx  # Default: 500, Range: 10-1000

# Read/write limits (bytes per second)
docker run -d \
  --device-read-bps=/dev/sda:10mb \
  --device-write-bps=/dev/sda:5mb \
  nginx
```

### Docker Compose Resource Limits:

```yaml
version: '3.8'

services:
  web:
    image: nginx
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M

  database:
    image: postgres:13
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
```

### PID Limits:

```bash
# Limit number of processes
docker run -d --pids-limit=100 nginx

# Unlimited processes
docker run -d --pids-limit=-1 nginx
```

### Monitor Resource Usage:

```bash
# Real-time stats
docker stats

# Stats for specific container
docker stats web

# Format output
docker stats --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}"

# No streaming (one-time)
docker stats --no-stream
```

### Inspect Resource Usage:

```bash
# View current usage
docker inspect -f '{{.HostConfig.Memory}}' container_name

# View all resource configs
docker inspect container_name | jq '.[0].HostConfig'
```

---

## Docker Registry

### Public Registries:
- Docker Hub (hub.docker.com)
- Google Container Registry (gcr.io)
- Amazon ECR
- GitHub Container Registry (ghcr.io)
- Quay.io

### Push to Docker Hub:

```bash
# Login
docker login

# Tag image
docker tag myapp:latest username/myapp:latest
docker tag myapp:latest username/myapp:1.0

# Push
docker push username/myapp:latest
docker push username/myapp:1.0

# Pull
docker pull username/myapp:latest
```

### Private Registry:

```bash
# Run local registry
docker run -d -p 5000:5000 --name registry registry:2

# Tag for local registry
docker tag myapp:latest localhost:5000/myapp:latest

# Push to local registry
docker push localhost:5000/myapp:latest

# Pull from local registry
docker pull localhost:5000/myapp:latest
```

### Secure Private Registry:

```bash
# Generate certificates
mkdir -p certs
openssl req -newkey rsa:4096 -nodes -sha256 \
  -keyout certs/domain.key -x509 -days 365 \
  -out certs/domain.crt

# Run registry with TLS
docker run -d \
  -p 5000:5000 \
  --name registry \
  -v $(pwd)/certs:/certs \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  registry:2
```

### Registry with Authentication:

```bash
# Create password file
mkdir auth
docker run --rm \
  --entrypoint htpasswd \
  httpd:2 -Bbn user password > auth/htpasswd

# Run registry with auth
docker run -d \
  -p 5000:5000 \
  --name registry \
  -v $(pwd)/auth:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" \
  registry:2

# Login to registry
docker login localhost:5000
```

---

## CI/CD Integration

### GitHub Actions Example:

**.github/workflows/docker.yml:**
```yaml
name: Docker Build and Push

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Log in to Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}

      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### GitLab CI Example:

**.gitlab-ci.yml:**
```yaml
stages:
  - build
  - test
  - deploy

variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG

build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $DOCKER_IMAGE .
    - docker push $DOCKER_IMAGE

test:
  stage: test
  image: $DOCKER_IMAGE
  script:
    - pytest tests/

deploy:
  stage: deploy
  image: docker:latest
  script:
    - docker pull $DOCKER_IMAGE
    - docker stop myapp || true
    - docker rm myapp || true
    - docker run -d --name myapp -p 80:80 $DOCKER_IMAGE
  only:
    - main
```

### Jenkins Pipeline:

**Jenkinsfile:**
```groovy
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "myapp"
        DOCKER_TAG = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    docker.image("${DOCKER_IMAGE}:${DOCKER_TAG}").inside {
                        sh 'pytest tests/'
                    }
                }
            }
        }

        stage('Push') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-credentials') {
                        docker.image("${DOCKER_IMAGE}:${DOCKER_TAG}").push()
                        docker.image("${DOCKER_IMAGE}:${DOCKER_TAG}").push('latest')
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                sh """
                    docker stop myapp || true
                    docker rm myapp || true
                    docker run -d --name myapp -p 80:80 ${DOCKER_IMAGE}:${DOCKER_TAG}
                """
            }
        }
    }

    post {
        always {
            sh 'docker system prune -f'
        }
    }
}
```

---

## Logging and Monitoring

### Logging Drivers:

```bash
# Check default logging driver
docker info --format '{{.LoggingDriver}}'

# Run with specific logging driver
docker run -d --log-driver json-file nginx
docker run -d --log-driver syslog nginx
docker run -d --log-driver journald nginx
docker run -d --log-driver gelf nginx

# Set log options
docker run -d \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  nginx
```

### View Logs:

```bash
# View logs
docker logs container_name

# Follow logs
docker logs -f container_name

# Last N lines
docker logs --tail 100 container_name

# Since timestamp
docker logs --since 2024-01-01T00:00:00 container_name

# With timestamps
docker logs -t container_name

# Between timestamps
docker logs --since 2024-01-01 --until 2024-01-02 container_name
```

### Centralized Logging with ELK Stack:

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  elasticsearch:
    image: elasticsearch:8.5.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports:
      - "9200:9200"

  logstash:
    image: logstash:8.5.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    depends_on:
      - elasticsearch

  kibana:
    image: kibana:8.5.0
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

  app:
    image: myapp:latest
    logging:
      driver: gelf
      options:
        gelf-address: "udp://localhost:12201"
        tag: "myapp"
```

### Monitoring with Prometheus & Grafana:

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      - prometheus

  node-exporter:
    image: prom/node-exporter
    ports:
      - "9100:9100"

  cadvisor:
    image: gcr.io/cadvisor/cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro

volumes:
  prometheus_data:
  grafana_data:
```

**prometheus.yml:**
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
```

---

## Orchestration Introduction

For production, use orchestration platforms:

### Docker Swarm (Built-in):

```bash
# Initialize swarm
docker swarm init

# Deploy stack
docker stack deploy -c docker-compose.yml myapp

# List services
docker service ls

# Scale service
docker service scale myapp_web=5

# View service logs
docker service logs myapp_web
```

### Kubernetes (Industry Standard):

**deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        ports:
        - containerPort: 8080
        resources:
          limits:
            memory: "512Mi"
            cpu: "500m"
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

```bash
# Deploy to Kubernetes
kubectl apply -f deployment.yaml

# Scale
kubectl scale deployment myapp --replicas=5

# View pods
kubectl get pods

# View logs
kubectl logs -f pod-name
```

---

## Best Practices

### Development:
1. Use Docker Compose for local development
2. Mount source code as volumes for live reload
3. Use `:latest` tag locally
4. Keep Dockerfiles in version control

### Production:
1. Use specific image tags (not `latest`)
2. Multi-stage builds for smaller images
3. Run as non-root user
4. Scan images for vulnerabilities
5. Use health checks
6. Set resource limits
7. Implement proper logging
8. Use secrets management
9. Regular security updates
10. Monitor and alert

### General:
- One process per container
- Containers should be stateless
- Use `.dockerignore`
- Document with labels
- Keep images small
- Leverage build cache
- Use official base images
- Pin versions

---

## Advanced Examples

### Microservices Application:

```yaml
version: '3.8'

services:
  api-gateway:
    build: ./api-gateway
    ports:
      - "80:80"
    networks:
      - frontend
    depends_on:
      - user-service
      - product-service

  user-service:
    build: ./user-service
    environment:
      DATABASE_URL: postgresql://postgres:secret@user-db:5432/users
    networks:
      - frontend
      - backend
    depends_on:
      - user-db

  product-service:
    build: ./product-service
    environment:
      DATABASE_URL: postgresql://postgres:secret@product-db:5432/products
      REDIS_URL: redis://redis:6379
    networks:
      - frontend
      - backend
    depends_on:
      - product-db
      - redis

  user-db:
    image: postgres:13
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: users
    volumes:
      - user_data:/var/lib/postgresql/data
    networks:
      - backend

  product-db:
    image: postgres:13
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: products
    volumes:
      - product_data:/var/lib/postgresql/data
    networks:
      - backend

  redis:
    image: redis:alpine
    networks:
      - backend

networks:
  frontend:
  backend:

volumes:
  user_data:
  product_data:
```

---

## Troubleshooting

### Common Issues:

**1. Container exits immediately:**
```bash
# Check logs
docker logs container_name

# Check exit code
docker inspect container_name --format='{{.State.ExitCode}}'

# Run interactively to debug
docker run -it --entrypoint /bin/sh image_name
```

**2. Port already in use:**
```bash
# Find process using port
sudo lsof -i :8080
sudo netstat -tulpn | grep 8080

# Kill process
kill -9 PID

# Use different host port
docker run -p 8081:80 nginx
```

**3. Permission denied:**
```bash
# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Or use sudo
sudo docker run nginx
```

**4. Cannot connect to Docker daemon:**
```bash
# Start Docker service
sudo systemctl start docker

# Enable on boot
sudo systemctl enable docker

# Check status
sudo systemctl status docker
```

**5. Image not found:**
```bash
# Pull image
docker pull image_name

# Check image name
docker images | grep image_name

# Use full image path
docker pull registry.hub.docker.com/library/nginx:latest
```

**6. Out of disk space:**
```bash
# Check disk usage
docker system df

# Remove unused data
docker system prune -a

# Remove specific items
docker image prune
docker container prune
docker volume prune
```

---

## Summary

You've mastered:

✅ Multi-stage builds for optimized images  
✅ Docker Compose for multi-container apps  
✅ Advanced networking strategies  
✅ Security best practices  
✅ Image optimization techniques  
✅ Health checks and monitoring  
✅ Resource management  
✅ CI/CD integration  
✅ Production-ready deployments  

### Congratulations! 🎉

You now have comprehensive knowledge of Docker from basics to advanced production usage!

---

**End of Docker Advanced Guide**

Keep practicing and building real-world applications!
