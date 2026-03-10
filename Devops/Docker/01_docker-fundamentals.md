# Docker Fundamentals Guide
## From Beginner to Intermediate

---

## Table of Contents
1. [What is Docker?](#what-is-docker)
2. [Why Use Docker?](#why-use-docker)
3. [Docker Architecture](#docker-architecture)
4. [Installation](#installation)
5. [Core Concepts](#core-concepts)
6. [Docker Images](#docker-images)
7. [Docker Containers](#docker-containers)
8. [Dockerfile Basics](#dockerfile-basics)
9. [Essential Docker Commands](#essential-docker-commands)
10. [Docker Volumes](#docker-volumes)
11. [Docker Networking Basics](#docker-networking-basics)
12. [Practical Examples](#practical-examples)
13. [Exercises](#exercises)

---

## What is Docker?

Docker is an **open-source platform** that enables developers to automate the deployment, scaling, and management of applications using **containerization**.

### Key Concepts:

**Containerization** = Packaging an application with all its dependencies, libraries, and configuration files into a single, isolated unit called a "container."

Think of it like this:
```
Traditional Shipping:
- Different goods need different vehicles
- Loading/unloading is complex
- Incompatible systems

Standardized Containers:
- All goods go in standard containers
- Any ship/truck can carry them
- Predictable and efficient

Docker does this for software!
```

### Visual Representation:

```
┌─────────────────────────────────────────────────────────┐
│                    Physical Server                       │
│  ┌───────────────────────────────────────────────────┐  │
│  │              Operating System                      │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │          Docker Engine                      │  │  │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  │  │  │
│  │  │  │Container1│  │Container2│  │Container3│  │  │  │
│  │  │  │  App A   │  │  App B   │  │  App C   │  │  │  │
│  │  │  │  + Deps  │  │  + Deps  │  │  + Deps  │  │  │  │
│  │  │  └──────────┘  └──────────┘  └──────────┘  │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

## Why Use Docker?

### 1. **Consistency Across Environments**
Problem: "It works on my machine!"
```
Developer's Laptop  →  Testing Server  →  Production
   (Python 3.8)         (Python 3.7)       (Python 3.9)
        ❌                   ❌                 ❌
```

Solution with Docker:
```
Developer's Laptop  →  Testing Server  →  Production
   (Container)          (Container)        (Container)
        ✅                   ✅                 ✅
```

### 2. **Isolation**
Each container runs independently:
- No dependency conflicts
- No port conflicts
- Secure separation

### 3. **Portability**
Run anywhere Docker is installed:
- Local machine
- Cloud (AWS, Azure, GCP)
- On-premise servers

### 4. **Efficiency**
- Lightweight (shares OS kernel)
- Fast startup (seconds)
- Better resource utilization

### 5. **Version Control**
- Track changes to application environment
- Easy rollback
- Reproducible builds

### Comparison: Containers vs Virtual Machines

```
┌─────────────────────────┐     ┌─────────────────────────┐
│   Virtual Machines      │     │      Containers         │
├─────────────────────────┤     ├─────────────────────────┤
│  App A  │  App B  │App C│     │  App A │ App B │ App C │
│ ──────  │ ──────  │──── │     │ ────── │────── │────── │
│ Bins/   │ Bins/   │Bins/│     │ Bins/  │ Bins/ │ Bins/ │
│ Libs    │ Libs    │Libs │     │ Libs   │ Libs  │ Libs  │
│ ──────  │ ──────  │──── │     ├────────┴───────┴───────┤
│Guest OS │Guest OS │GstOS│     │    Docker Engine       │
├─────────┴─────────┴─────┤     ├────────────────────────┤
│     Hypervisor          │     │      Host OS           │
├─────────────────────────┤     ├────────────────────────┤
│     Host OS             │     │    Infrastructure      │
├─────────────────────────┤     └────────────────────────┘
│    Infrastructure       │
└─────────────────────────┘

Size: GBs                  Size: MBs
Boot: Minutes              Boot: Seconds
```

---

## Docker Architecture

Docker uses a **client-server architecture**:

```
┌──────────────────────────────────────────────────────────┐
│                      Docker Client                       │
│              (Command Line Interface)                    │
│              docker run, docker build, etc.              │
└────────────────────────┬─────────────────────────────────┘
                         │ REST API
                         ↓
┌──────────────────────────────────────────────────────────┐
│                     Docker Daemon                        │
│                   (dockerd process)                      │
│  ┌────────────────────────────────────────────────────┐ │
│  │  Manages: Images, Containers, Networks, Volumes    │ │
│  └────────────────────────────────────────────────────┘ │
└────────────────────────┬─────────────────────────────────┘
                         │
                         ↓
┌──────────────────────────────────────────────────────────┐
│                    Docker Registry                       │
│                    (Docker Hub, etc.)                    │
│              Stores and distributes images               │
└──────────────────────────────────────────────────────────┘
```

### Components:

1. **Docker Client** (`docker`)
   - User interface for Docker
   - Sends commands to Docker daemon
   - Can communicate with multiple daemons

2. **Docker Daemon** (`dockerd`)
   - Listens for Docker API requests
   - Manages Docker objects (images, containers, networks, volumes)
   - Can communicate with other daemons

3. **Docker Registry**
   - Stores Docker images
   - Docker Hub is the default public registry
   - Can run private registries

4. **Docker Objects**:
   - **Images**: Read-only templates with instructions
   - **Containers**: Runnable instances of images
   - **Networks**: Communication channels between containers
   - **Volumes**: Persistent data storage

---

## Installation

### Linux (Ubuntu/Debian):
```bash
# Update package index
sudo apt-get update

# Install required packages
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Add Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Set up repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io

# Add your user to docker group (avoid using sudo)
sudo usermod -aG docker $USER
newgrp docker
```

### macOS:
1. Download Docker Desktop from https://www.docker.com/products/docker-desktop
2. Install the .dmg file
3. Launch Docker Desktop from Applications

### Windows:
1. Download Docker Desktop from https://www.docker.com/products/docker-desktop
2. Run the installer
3. Enable WSL 2 if prompted
4. Launch Docker Desktop

### Verify Installation:
```bash
docker --version
# Output: Docker version 24.0.x, build xxxxx

docker run hello-world
# Should download and run a test container
```

---

## Core Concepts

### 1. Images
- **Blueprint** for containers
- Read-only template
- Contains application code, runtime, libraries, environment variables, and configuration files
- Layered file system

```
Image Layers (Bottom to Top):
┌────────────────────────┐
│  Application Code      │  ← Layer 4
├────────────────────────┤
│  Application Dependencies │  ← Layer 3
├────────────────────────┤
│  Python Runtime        │  ← Layer 2
├────────────────────────┤
│  Base OS (Ubuntu)      │  ← Layer 1
└────────────────────────┘
```

### 2. Containers
- **Running instance** of an image
- Isolated process
- Can be started, stopped, moved, and deleted
- Ephemeral by default (data lost when deleted)

```
Image → Container Relationship:

      Image (Blueprint)
           │
           ├─→ Container 1 (Running)
           ├─→ Container 2 (Running)
           └─→ Container 3 (Stopped)
```

### 3. Dockerfile
- Text file with instructions to build an image
- Automates image creation
- Version controllable

### 4. Registry
- Storage and distribution system for images
- Docker Hub is the default
- Can host private registries

---

## Docker Images

### Understanding Images

Think of images as **class definitions** and containers as **instances** (in OOP terms).

```
class Image:           |  Container 1 = new Image()
  - OS                 |  Container 2 = new Image()
  - Dependencies       |  Container 3 = new Image()
  - Application        |
```

### Image Naming Convention:
```
[registry/][username/]repository[:tag]

Examples:
nginx                    → Official nginx image, latest tag
nginx:1.21               → nginx version 1.21
ubuntu:20.04             → Ubuntu 20.04
myregistry.com/app:v1.0  → Custom registry
python:3.9-slim          → Python 3.9 slim variant
```

### Working with Images:

#### 1. Pull an Image from Registry:
```bash
docker pull nginx
# Downloads nginx:latest from Docker Hub

docker pull nginx:1.21
# Downloads specific version

docker pull ubuntu:20.04
# Downloads Ubuntu 20.04
```

#### 2. List Images:
```bash
docker images
# or
docker image ls

# Output:
# REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
# nginx        latest    abc123def456   2 weeks ago    142MB
# ubuntu       20.04     xyz789abc123   3 weeks ago    72.8MB
```

#### 3. Inspect an Image:
```bash
docker image inspect nginx

# Shows detailed information:
# - Layers
# - Environment variables
# - Exposed ports
# - Entry point
# - Commands
```

#### 4. Search for Images:
```bash
docker search python

# Shows available Python images on Docker Hub
```

#### 5. Remove an Image:
```bash
docker rmi nginx
# or
docker image rm nginx

# Remove specific version
docker rmi nginx:1.21

# Remove by ID
docker rmi abc123def456

# Force remove (even if containers exist)
docker rmi -f nginx
```

#### 6. Remove Unused Images:
```bash
# Remove dangling images (untagged)
docker image prune

# Remove all unused images
docker image prune -a

# Remove without confirmation
docker image prune -af
```

### Image Layers

Docker images are built in layers. Each instruction in a Dockerfile creates a new layer.

```
Dockerfile:                  Layers Created:
FROM ubuntu:20.04       →    Layer 1: Ubuntu base
RUN apt-get update      →    Layer 2: Updated packages
RUN apt-get install -y  →    Layer 3: Python installed
    python3
COPY app.py /app/       →    Layer 4: Application code
CMD ["python3", "app"]  →    Metadata (not a layer)

Benefits:
- Reusable layers (caching)
- Efficient storage
- Faster builds
```

---

## Docker Containers

### Container Lifecycle:

```
      CREATE
         │
         ↓
    ┌─────────┐
    │ CREATED │
    └─────────┘
         │ START
         ↓
    ┌─────────┐     PAUSE      ┌────────┐
    │ RUNNING │ ←──────────────│ PAUSED │
    └─────────┘                └────────┘
         │ STOP               UNPAUSE  ↑
         ↓                             │
    ┌─────────┐ ───────────────────────┘
    │ STOPPED │
    └─────────┘
         │ REMOVE
         ↓
    ┌─────────┐
    │ DELETED │
    └─────────┘
```

### Running Containers:

#### 1. Basic Container Run:
```bash
# Run container (foreground)
docker run nginx

# Run container (background/detached)
docker run -d nginx

# Run with custom name
docker run -d --name my-nginx nginx

# Run and remove after exit
docker run --rm nginx
```

#### 2. Interactive Containers:
```bash
# Interactive mode with terminal
docker run -it ubuntu bash
# -i = interactive (keep STDIN open)
# -t = TTY (pseudo-terminal)

# Now you're inside the container:
root@abc123:/# ls
root@abc123:/# echo "Hello from container"
root@abc123:/# exit
```

#### 3. Port Mapping:
```bash
# Map container port to host port
docker run -d -p 8080:80 nginx
# -p HOST_PORT:CONTAINER_PORT
# Access nginx at http://localhost:8080

# Map to random host port
docker run -d -P nginx
# Publishes all exposed ports to random ports

# Multiple port mappings
docker run -d -p 8080:80 -p 443:443 nginx
```

#### 4. Environment Variables:
```bash
# Set single environment variable
docker run -d -e MYSQL_ROOT_PASSWORD=secret mysql

# Set multiple variables
docker run -d \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=myapp \
  -e MYSQL_USER=appuser \
  mysql

# Load from file
docker run -d --env-file ./config.env mysql
```

#### 5. Execute Commands in Running Container:
```bash
# Execute command in running container
docker exec my-nginx ls /usr/share/nginx/html

# Interactive shell in running container
docker exec -it my-nginx bash

# Run as specific user
docker exec -u root my-nginx whoami
```

### Container Management Commands:

#### 1. List Containers:
```bash
# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# List with specific format
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Show only container IDs
docker ps -q
```

#### 2. Stop/Start Containers:
```bash
# Stop container (graceful, sends SIGTERM)
docker stop my-nginx

# Stop with timeout
docker stop -t 30 my-nginx

# Force stop (sends SIGKILL)
docker kill my-nginx

# Start stopped container
docker start my-nginx

# Restart container
docker restart my-nginx
```

#### 3. Pause/Unpause:
```bash
# Pause container (freeze processes)
docker pause my-nginx

# Unpause container
docker unpause my-nginx
```

#### 4. View Logs:
```bash
# View container logs
docker logs my-nginx

# Follow logs (real-time)
docker logs -f my-nginx

# Show timestamps
docker logs -t my-nginx

# Show last N lines
docker logs --tail 100 my-nginx

# Show logs since timestamp
docker logs --since 2024-01-01T00:00:00 my-nginx
```

#### 5. Inspect Container:
```bash
# Detailed container information
docker inspect my-nginx

# Get specific field
docker inspect -f '{{.NetworkSettings.IPAddress}}' my-nginx

# Get multiple fields
docker inspect -f '{{.State.Status}} {{.RestartCount}}' my-nginx
```

#### 6. Container Stats:
```bash
# Real-time resource usage
docker stats

# Stats for specific container
docker stats my-nginx

# One-time snapshot (no streaming)
docker stats --no-stream
```

#### 7. Remove Containers:
```bash
# Remove stopped container
docker rm my-nginx

# Force remove (even if running)
docker rm -f my-nginx

# Remove multiple containers
docker rm container1 container2 container3

# Remove all stopped containers
docker container prune

# Remove all containers (stopped and running)
docker rm -f $(docker ps -aq)
```

---

## Dockerfile Basics

A **Dockerfile** is a text document containing instructions to build a Docker image.

### Dockerfile Structure:

```dockerfile
# Comment
INSTRUCTION arguments
```

### Common Instructions:

#### 1. FROM
Specifies the base image. **Must be the first instruction.**

```dockerfile
FROM ubuntu:20.04
FROM python:3.9
FROM node:16-alpine
FROM scratch  # Empty base (for minimal images)
```

#### 2. RUN
Executes commands in a new layer.

```dockerfile
# Shell form (runs in /bin/sh -c)
RUN apt-get update
RUN apt-get install -y python3

# Exec form (doesn't invoke shell)
RUN ["apt-get", "install", "-y", "python3"]

# Multiple commands (better - creates fewer layers)
RUN apt-get update && \
    apt-get install -y python3 python3-pip && \
    apt-get clean
```

#### 3. CMD
Provides default command to run when container starts. **Only last CMD is used.**

```dockerfile
# Shell form
CMD python3 app.py

# Exec form (preferred)
CMD ["python3", "app.py"]

# As parameters to ENTRYPOINT
CMD ["--port", "8080"]
```

#### 4. ENTRYPOINT
Configures container to run as executable.

```dockerfile
ENTRYPOINT ["python3", "app.py"]

# Can be combined with CMD for default parameters
ENTRYPOINT ["python3", "app.py"]
CMD ["--port", "8080"]
# Run with: docker run myapp
# Override CMD: docker run myapp --port 9000
```

#### 5. COPY
Copies files from host to container.

```dockerfile
COPY app.py /app/
COPY requirements.txt /app/
COPY . /app/  # Copy everything in current directory

# Copy with ownership
COPY --chown=user:group app.py /app/
```

#### 6. ADD
Similar to COPY, but with extra features (auto-extraction of tar files, URL support).

```dockerfile
ADD app.tar.gz /app/  # Auto-extracts
ADD http://example.com/file.txt /app/  # Downloads

# Use COPY instead unless you need ADD's features
```

#### 7. WORKDIR
Sets working directory for subsequent instructions.

```dockerfile
WORKDIR /app
# All following commands execute in /app

COPY . .  # Copies to /app
RUN ls    # Lists /app contents
```

#### 8. ENV
Sets environment variables.

```dockerfile
ENV APP_HOME=/app
ENV PORT=8080
ENV DEBUG=true

# Multiple variables
ENV APP_HOME=/app \
    PORT=8080 \
    DEBUG=true
```

#### 9. EXPOSE
Documents which ports the container listens on (doesn't actually publish).

```dockerfile
EXPOSE 80
EXPOSE 443
EXPOSE 8080/tcp
EXPOSE 8081/udp
```

#### 10. VOLUME
Creates mount point for persistent data.

```dockerfile
VOLUME /data
VOLUME ["/data", "/logs"]
```

#### 11. USER
Sets user/group for subsequent instructions.

```dockerfile
USER appuser
USER 1000:1000

# Best practice: Don't run as root
RUN useradd -m appuser
USER appuser
```

#### 12. ARG
Defines build-time variables.

```dockerfile
ARG VERSION=1.0
ARG PYTHON_VERSION=3.9

FROM python:${PYTHON_VERSION}

# Use during build:
# docker build --build-arg VERSION=2.0 .
```

#### 13. LABEL
Adds metadata to image.

```dockerfile
LABEL version="1.0"
LABEL description="My application"
LABEL maintainer="dev@example.com"
```

### Complete Dockerfile Example:

```dockerfile
# Use official Python runtime as base image
FROM python:3.9-slim

# Set metadata
LABEL maintainer="developer@example.com"
LABEL version="1.0"

# Set build arguments
ARG APP_DIR=/app
ARG USER=appuser

# Set environment variables
ENV PYTHONUNBUFFERED=1 \
    APP_HOME=${APP_DIR}

# Install system dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        gcc \
        && rm -rf /var/lib/apt/lists/*

# Create application directory
WORKDIR ${APP_HOME}

# Copy requirements first (for caching)
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create non-root user
RUN useradd -m ${USER} && \
    chown -R ${USER}:${USER} ${APP_HOME}

# Switch to non-root user
USER ${USER}

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD python -c "import requests; requests.get('http://localhost:8000/health')"

# Set entrypoint and default command
ENTRYPOINT ["python"]
CMD ["app.py"]
```

### Building Images:

```bash
# Build from Dockerfile in current directory
docker build -t myapp:1.0 .

# Build with custom Dockerfile name
docker build -t myapp -f Dockerfile.prod .

# Build with build arguments
docker build --build-arg VERSION=2.0 -t myapp .

# Build without cache
docker build --no-cache -t myapp .

# Build and output to file
docker build -t myapp -o type=tar,dest=myapp.tar .
```

### Example: Python Flask Application

**Directory structure:**
```
myapp/
├── Dockerfile
├── requirements.txt
└── app.py
```

**app.py:**
```python
from flask import Flask
import os

app = Flask(__name__)
port = int(os.environ.get('PORT', 8000))

@app.route('/')
def hello():
    return 'Hello from Docker!'

@app.route('/health')
def health():
    return 'OK', 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=port)
```

**requirements.txt:**
```
Flask==2.3.0
```

**Dockerfile:**
```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 8000

CMD ["python", "app.py"]
```

**Build and run:**
```bash
# Build image
docker build -t flask-app:1.0 .

# Run container
docker run -d -p 8080:8000 --name my-flask-app flask-app:1.0

# Test
curl http://localhost:8080
# Output: Hello from Docker!

# View logs
docker logs my-flask-app

# Stop and remove
docker stop my-flask-app
docker rm my-flask-app
```

---

## Essential Docker Commands

### Quick Reference:

```bash
# IMAGES
docker pull <image>              # Download image
docker images                    # List images
docker rmi <image>               # Remove image
docker build -t <name> .         # Build image
docker image prune               # Remove unused images

# CONTAINERS
docker run <image>               # Create and start container
docker ps                        # List running containers
docker ps -a                     # List all containers
docker stop <container>          # Stop container
docker start <container>         # Start stopped container
docker restart <container>       # Restart container
docker rm <container>            # Remove container
docker exec -it <container> bash # Execute command in container
docker logs <container>          # View container logs

# SYSTEM
docker system df                 # Show disk usage
docker system prune              # Remove unused data
docker system prune -a           # Remove all unused data
docker info                      # Display system information
docker version                   # Show Docker version

# NETWORKS
docker network ls                # List networks
docker network create <name>     # Create network
docker network rm <name>         # Remove network
docker network inspect <name>    # Inspect network

# VOLUMES
docker volume ls                 # List volumes
docker volume create <name>      # Create volume
docker volume rm <name>          # Remove volume
docker volume inspect <name>     # Inspect volume
```

### Command Cheat Sheet with Examples:

```bash
# Run nginx in background, name it web, map port 8080→80
docker run -d --name web -p 8080:80 nginx

# Run Ubuntu interactively, remove after exit
docker run --rm -it ubuntu bash

# Run with environment variables and volume
docker run -d \
  -e MYSQL_ROOT_PASSWORD=secret \
  -v mysql_data:/var/lib/mysql \
  --name db \
  mysql:8.0

# View last 100 log lines, follow new logs
docker logs --tail 100 -f web

# Execute bash in running container
docker exec -it web bash

# Copy file from container to host
docker cp web:/etc/nginx/nginx.conf ./nginx.conf

# Copy file from host to container
docker cp ./index.html web:/usr/share/nginx/html/

# Inspect container JSON output
docker inspect web

# Get container IP address
docker inspect -f '{{.NetworkSettings.IPAddress}}' web

# Stop all running containers
docker stop $(docker ps -q)

# Remove all stopped containers
docker rm $(docker ps -aq)

# Remove all unused images, containers, networks
docker system prune -a

# Export container to tar file
docker export web > web.tar

# Import tar file as image
docker import web.tar myapp:latest

# Save image to tar file
docker save -o nginx.tar nginx:latest

# Load image from tar file
docker load -i nginx.tar

# Tag image
docker tag nginx:latest myregistry.com/nginx:1.0

# Push image to registry
docker push myregistry.com/nginx:1.0

# Pull specific platform image
docker pull --platform linux/amd64 nginx

# Show real-time resource usage
docker stats

# Show resource usage for specific container
docker stats web
```

---

## Docker Volumes

Containers are **ephemeral** by default - data is lost when container is removed. Volumes provide **persistent storage**.

### Why Use Volumes?

```
Without Volumes:                With Volumes:
Container starts                Container starts
    ↓                               ↓
Data created                    Data created
    ↓                               ↓
Container removed               Volume persists
    ↓                               ↓
Data LOST ❌                   Data SAVED ✅
```

### Types of Data Persistence:

```
┌─────────────────────────────────────────────────┐
│              Host Machine                       │
│                                                 │
│  ┌──────────────────┐    ┌─────────────────┐  │
│  │    Volumes       │    │  Bind Mounts    │  │
│  │  (Docker-managed)│    │  (Host paths)   │  │
│  │                  │    │                 │  │
│  │  /var/lib/docker/│    │  /home/user/    │  │
│  │    volumes/      │    │     data/       │  │
│  └────────┬─────────┘    └────────┬────────┘  │
│           │                       │            │
│           └───────────┬───────────┘            │
│                       │                        │
│                  Container                     │
│              ┌──────────────┐                  │
│              │  /app/data   │                  │
│              └──────────────┘                  │
└─────────────────────────────────────────────────┘
```

### 1. Named Volumes (Recommended):

Managed by Docker, stored in Docker's directory.

```bash
# Create volume
docker volume create my_data

# List volumes
docker volume ls

# Inspect volume
docker volume inspect my_data
# Shows mount point: /var/lib/docker/volumes/my_data/_data

# Use volume in container
docker run -d \
  -v my_data:/app/data \
  --name app1 \
  ubuntu

# Multiple containers can share volume
docker run -d \
  -v my_data:/app/data \
  --name app2 \
  ubuntu

# Remove volume
docker volume rm my_data

# Remove all unused volumes
docker volume prune
```

### 2. Bind Mounts:

Maps host directory to container directory.

```bash
# Mount current directory
docker run -d \
  -v $(pwd):/app \
  --name app \
  nginx

# Windows PowerShell
docker run -d -v ${PWD}:/app --name app nginx

# Absolute path
docker run -d \
  -v /home/user/myapp:/app \
  --name app \
  nginx

# Read-only mount
docker run -d \
  -v $(pwd):/app:ro \
  --name app \
  nginx
```

### 3. tmpfs Mounts:

Stored in host memory, not persisted to disk.

```bash
docker run -d \
  --tmpfs /app/temp \
  --name app \
  nginx

# With size and mode options
docker run -d \
  --tmpfs /app/temp:rw,size=100m,mode=1777 \
  --name app \
  nginx
```

### Volume Use Cases:

**Named Volumes:**
- Database data
- Application state
- Shared data between containers
- Production deployments

**Bind Mounts:**
- Development (live code reloading)
- Configuration files
- Access to host files
- Sharing code with container

**tmpfs:**
- Sensitive data (not persisted)
- Temporary cache
- Build artifacts

### Practical Example: MySQL with Persistent Data

```bash
# Create volume for MySQL data
docker volume create mysql_data

# Run MySQL with volume
docker run -d \
  --name mysql_db \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=myapp \
  -v mysql_data:/var/lib/mysql \
  -p 3306:3306 \
  mysql:8.0

# Connect and create data
docker exec -it mysql_db mysql -uroot -psecret
# mysql> CREATE TABLE users (id INT, name VARCHAR(50));
# mysql> INSERT INTO users VALUES (1, 'Alice');
# mysql> exit

# Stop and remove container
docker stop mysql_db
docker rm mysql_db

# Data is still in volume!
# Start new container with same volume
docker run -d \
  --name mysql_db_new \
  -e MYSQL_ROOT_PASSWORD=secret \
  -v mysql_data:/var/lib/mysql \
  -p 3306:3306 \
  mysql:8.0

# Data persists!
docker exec -it mysql_db_new mysql -uroot -psecret -e "SELECT * FROM myapp.users;"
# Output: 1 | Alice
```

### Volume Backup and Restore:

```bash
# Backup volume to tar file
docker run --rm \
  -v my_data:/data \
  -v $(pwd):/backup \
  ubuntu \
  tar czf /backup/backup.tar.gz /data

# Restore volume from tar file
docker run --rm \
  -v my_data:/data \
  -v $(pwd):/backup \
  ubuntu \
  bash -c "cd /data && tar xzf /backup/backup.tar.gz --strip 1"

# Copy data between volumes
docker run --rm \
  -v old_volume:/from \
  -v new_volume:/to \
  ubuntu \
  bash -c "cd /from && cp -av . /to"
```

---

## Docker Networking Basics

Docker networking allows containers to communicate with each other and the outside world.

### Network Drivers:

```
┌────────────────────────────────────────────────┐
│              Docker Networks                   │
├────────────────────────────────────────────────┤
│  1. bridge    (Default, single host)           │
│  2. host      (Use host network)               │
│  3. none      (No networking)                  │
│  4. overlay   (Multi-host, Swarm)              │
│  5. macvlan   (Assign MAC address)             │
└────────────────────────────────────────────────┘
```

### 1. Bridge Network (Default):

Containers on same bridge can communicate using container names.

```bash
# List networks
docker network ls
# Default networks: bridge, host, none

# Create custom bridge network
docker network create my_network

# Run containers on same network
docker run -d --name web --network my_network nginx
docker run -d --name api --network my_network python:3.9

# Containers can ping each other by name
docker exec web ping api
docker exec api ping web

# Inspect network
docker network inspect my_network

# Connect running container to network
docker network connect my_network existing_container

# Disconnect container from network
docker network disconnect my_network existing_container

# Remove network
docker network rm my_network
```

### Network Communication Example:

```bash
# Create network
docker network create app_network

# Run database
docker run -d \
  --name postgres_db \
  --network app_network \
  -e POSTGRES_PASSWORD=secret \
  postgres:13

# Run application (can connect to database using hostname "postgres_db")
docker run -d \
  --name flask_app \
  --network app_network \
  -e DATABASE_URL=postgresql://postgres:secret@postgres_db:5432/mydb \
  -p 8000:8000 \
  myflask:1.0

# Application connects to database using name "postgres_db"
```

### 2. Host Network:

Container uses host's network directly (no isolation).

```bash
docker run -d --network host nginx

# Container uses host's ports directly
# No need for -p flag
# Access on host's IP:80
```

### 3. None Network:

No networking.

```bash
docker run -d --network none nginx

# Container has no network access
# Useful for isolated processing
```

### Port Mapping Recap:

```bash
# Map single port
docker run -d -p 8080:80 nginx
# Host:Container

# Map all exposed ports to random host ports
docker run -d -P nginx

# Map specific IP
docker run -d -p 127.0.0.1:8080:80 nginx

# Map UDP port
docker run -d -p 8080:80/udp myapp

# Multiple ports
docker run -d \
  -p 80:80 \
  -p 443:443 \
  nginx
```

### Exposing Ports:

```dockerfile
# In Dockerfile (documentation only)
EXPOSE 80
EXPOSE 443

# Still need -p flag when running
docker run -d -p 8080:80 myimage
```

---

## Practical Examples

### Example 1: Simple Web Server

**Dockerfile:**
```dockerfile
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/

EXPOSE 80
```

**index.html:**
```html
<!DOCTYPE html>
<html>
<head><title>Docker Demo</title></head>
<body>
  <h1>Hello from Docker!</h1>
  <p>This is running in a container.</p>
</body>
</html>
```

**Commands:**
```bash
# Build
docker build -t my-web .

# Run
docker run -d -p 8080:80 --name web my-web

# Test
curl http://localhost:8080

# Clean up
docker stop web && docker rm web
```

### Example 2: Node.js Application

**app.js:**
```javascript
const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => {
  res.json({ message: 'Hello from Node.js in Docker!' });
});

app.listen(port, () => {
  console.log(`Server running on port ${port}`);
});
```

**package.json:**
```json
{
  "name": "docker-node-app",
  "version": "1.0.0",
  "main": "app.js",
  "dependencies": {
    "express": "^4.18.0"
  }
}
```

**Dockerfile:**
```dockerfile
FROM node:16-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY app.js .

EXPOSE 3000

CMD ["node", "app.js"]
```

**.dockerignore:**
```
node_modules
npm-debug.log
.git
.gitignore
README.md
```

**Commands:**
```bash
docker build -t node-app .
docker run -d -p 3000:3000 --name node-app node-app
curl http://localhost:3000
```

### Example 3: Multi-Container Application

**Setup: Web + Database**

```bash
# Create network
docker network create app-network

# Run PostgreSQL
docker run -d \
  --name postgres \
  --network app-network \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=appdb \
  -v postgres_data:/var/lib/postgresql/data \
  postgres:13

# Run web application
docker run -d \
  --name web \
  --network app-network \
  -e DB_HOST=postgres \
  -e DB_PASSWORD=secret \
  -p 8080:8080 \
  my-web-app:1.0

# Web app connects to database using hostname "postgres"
```

---

## Exercises

### Exercise 1: Basic Container Operations
1. Pull the `ubuntu:20.04` image
2. Run it interactively with bash
3. Inside container, create a file: `echo "Hello" > /tmp/test.txt`
4. Exit container
5. Start the same container again
6. Verify the file still exists
7. Remove the container

<details>
<summary>Solution</summary>

```bash
docker pull ubuntu:20.04
docker run -it --name my-ubuntu ubuntu:20.04 bash
# Inside container:
echo "Hello" > /tmp/test.txt
cat /tmp/test.txt
exit
# Outside container:
docker start my-ubuntu
docker exec my-ubuntu cat /tmp/test.txt
docker stop my-ubuntu
docker rm my-ubuntu
```
</details>

### Exercise 2: Build a Simple Web Application
Create a Python Flask app that displays "Hello, Docker!" and build a Docker image for it.

Requirements:
- Use Python 3.9
- Application should run on port 5000
- Access at http://localhost:5000

<details>
<summary>Solution</summary>

**app.py:**
```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello():
    return 'Hello, Docker!'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

**requirements.txt:**
```
Flask==2.3.0
```

**Dockerfile:**
```dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]
```

**Commands:**
```bash
docker build -t flask-hello .
docker run -d -p 5000:5000 --name flask-app flask-hello
curl http://localhost:5000
docker stop flask-app && docker rm flask-app
```
</details>

### Exercise 3: Persistent Data with Volumes
1. Create a named volume `data_vol`
2. Run an Ubuntu container with this volume mounted at `/data`
3. Create a file in `/data` directory
4. Remove the container
5. Start a new container with the same volume
6. Verify the file still exists

<details>
<summary>Solution</summary>

```bash
docker volume create data_vol
docker run -it --name test1 -v data_vol:/data ubuntu bash
# Inside container:
echo "Persistent data!" > /data/myfile.txt
exit
docker rm test1
docker run -it --name test2 -v data_vol:/data ubuntu bash
# Inside container:
cat /data/myfile.txt  # File still exists!
exit
docker rm test2
docker volume rm data_vol
```
</details>

### Exercise 4: Networking
1. Create a custom bridge network
2. Run two containers (nginx and alpine) on this network
3. From alpine container, ping nginx by name
4. From alpine, curl nginx

<details>
<summary>Solution</summary>

```bash
docker network create test-network
docker run -d --name nginx --network test-network nginx
docker run -it --name alpine --network test-network alpine sh
# Inside alpine:
ping nginx
apk add curl
curl http://nginx
exit
docker stop nginx alpine
docker rm nginx alpine
docker network rm test-network
```
</details>

---

## Summary

You've learned:

✅ What Docker is and why it's useful  
✅ Docker architecture and core concepts  
✅ How to work with images and containers  
✅ Writing Dockerfiles  
✅ Essential Docker commands  
✅ Data persistence with volumes  
✅ Basic networking between containers  

### Next Steps:

Continue to **Docker Advanced Guide (Part 2)** to learn:
- Multi-stage builds
- Docker Compose
- Advanced networking
- Security best practices
- Image optimization
- CI/CD integration
- Container orchestration

---

## Quick Command Reference Card

```bash
# Images
docker pull <image>              # Download image
docker build -t <name> .         # Build image
docker images                    # List images
docker rmi <image>               # Remove image

# Containers
docker run <image>               # Run container
docker run -d <image>            # Run in background
docker run -it <image> bash      # Interactive mode
docker run -p 8080:80 <image>    # Port mapping
docker ps                        # List running
docker ps -a                     # List all
docker stop <container>          # Stop container
docker start <container>         # Start container
docker restart <container>       # Restart
docker rm <container>            # Remove container
docker logs <container>          # View logs
docker exec -it <container> bash # Execute command

# Volumes
docker volume create <name>      # Create volume
docker volume ls                 # List volumes
docker volume rm <name>          # Remove volume
docker run -v <name>:/path       # Use volume

# Networks
docker network create <name>     # Create network
docker network ls                # List networks
docker run --network <name>      # Use network

# Cleanup
docker system prune              # Remove unused data
docker system prune -a           # Remove all unused
```

---

**End of Docker Fundamentals Guide**

Continue your learning with **Docker Advanced Guide** for production-ready skills!
