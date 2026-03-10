# Phase 15.1 — Docker for Java

## What is Docker and Why Does It Matter?

Docker is a containerization platform that packages your Java application along with all its dependencies — the JRE, libraries, config files, environment variables — into a single, portable unit called a **container**. Unlike a Virtual Machine (VM), a container does not include a full OS kernel. Instead, it shares the host OS kernel and only bundles the application layer, making it lightweight and fast to start.

The problem Docker solves is the classic "it works on my machine" syndrome. By shipping not just your `.jar` file but the entire runtime environment, you guarantee that the app behaves identically on a developer's laptop, in a CI pipeline, and in production.

---

## Core Concepts

**Image** — A read-only blueprint for a container. Think of it like a Java class.

**Container** — A running instance of an image. Think of it like a Java object (instantiated from the class).

**Dockerfile** — A script of instructions that Docker uses to build an image.

**Registry** — A storage location for images. Docker Hub is the public default; you can also use AWS ECR, GCR, or a private registry.

**Layer** — Each instruction in a Dockerfile creates a read-only layer. Layers are cached, which is why the order of instructions matters enormously for build performance.

---

## Writing a Dockerfile for Java

### The Naive Approach (Don't Do This)

```dockerfile
# ❌ Naive — ships the entire JDK + source code
FROM openjdk:21
COPY . /app
WORKDIR /app
RUN ./mvnw package
CMD ["java", "-jar", "target/myapp.jar"]
```

This is terrible in production because:
- The final image includes Maven, source code, test dependencies — easily 600MB+
- Every code change invalidates all layers, so rebuilds are slow
- You are shipping build tools to production (a security risk)

---

### The Right Approach — Multi-Stage Builds

A **multi-stage build** uses multiple `FROM` instructions in one Dockerfile. The first stage compiles and packages the app (using a fat JDK image). The second stage creates a lean runtime image that only copies the compiled artifact from stage 1. The build-stage artifacts are discarded automatically.

```dockerfile
# ============================================================
# STAGE 1: Build — uses a full JDK image to compile
# ============================================================
FROM eclipse-temurin:21-jdk-alpine AS builder

WORKDIR /app

# Copy Maven wrapper and pom.xml FIRST (dependency layer is cached separately)
# This is the single most important caching trick: if pom.xml hasn't changed,
# Docker reuses the cached dependency layer and skips re-downloading everything.
COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .

# Download dependencies (cached as a separate layer)
RUN ./mvnw dependency:go-offline -q

# Now copy the actual source code
COPY src ./src

# Build the fat JAR, skipping tests (tests should run in CI before this step)
RUN ./mvnw package -DskipTests -q

# ============================================================
# STAGE 2: Runtime — uses a minimal JRE image
# ============================================================
FROM eclipse-temurin:21-jre-alpine

# Create a non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy ONLY the final JAR from the build stage — nothing else
COPY --from=builder /app/target/myapp-*.jar app.jar

# Switch to non-root user before running
USER appuser

# Expose the port your Spring Boot app listens on
EXPOSE 8080

# JVM flags explained below
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-jar", "app.jar"]
```

The resulting image size is typically **150–200MB** instead of 600MB+.

---

## JVM in Containers — The Critical Problem

Before Java 10, the JVM was **container-unaware**. When you set a container's memory limit to 512MB, the JVM would query the host machine's total RAM (say 16GB) and size its heap accordingly — leading to OutOfMemoryKiller termination from the OS, which looks like a random crash.

**`-XX:+UseContainerSupport`** (enabled by default from Java 11 onwards) tells the JVM to respect cgroup (container) memory and CPU limits instead of reading the full host system resources.

**`-XX:MaxRAMPercentage=75.0`** sets the max heap as a percentage of the container's memory limit. Leaving 25% for the JVM's off-heap memory (Metaspace, thread stacks, code cache, GC overhead) is a safe default.

```bash
# Example: container has 1GB limit
# With MaxRAMPercentage=75.0, the JVM heap will be ~768MB
docker run -m 1g -e JAVA_OPTS="-XX:MaxRAMPercentage=75.0" myapp
```

**CPU Awareness** — `-XX:ActiveProcessorCount=N` can be used to override the CPU count the JVM sees, though `UseContainerSupport` usually handles this automatically.

---

## Docker Compose for Local Development

Docker Compose lets you define a multi-container application (your app + database + cache) in a single YAML file and spin everything up with one command.

```yaml
# docker-compose.yml
version: "3.9"

services:
  # Your Spring Boot application
  app:
    build: .                          # Build from Dockerfile in current directory
    ports:
      - "8080:8080"                   # host:container
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/mydb
      - SPRING_DATASOURCE_USERNAME=postgres
      - SPRING_DATASOURCE_PASSWORD=secret
      - SPRING_REDIS_HOST=redis
    depends_on:
      db:
        condition: service_healthy    # Wait for DB to be ready before starting app
      redis:
        condition: service_started
    networks:
      - backend

  # PostgreSQL database
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres_data:/var/lib/postgresql/data   # Persist data between restarts
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

  # Redis cache
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - backend

volumes:
  postgres_data:       # Named volume persists across container restarts

networks:
  backend:             # Isolated bridge network — containers talk by service name
```

```bash
# Start everything in the background
docker-compose up -d

# View logs
docker-compose logs -f app

# Stop and remove containers (keeps volumes)
docker-compose down

# Stop and ALSO remove volumes (wipes database)
docker-compose down -v
```

---

## JIB — Build Docker Images Without a Dockerfile

**JIB** is a Maven/Gradle plugin by Google that builds optimized Docker images directly from your Java project without needing a Dockerfile or Docker daemon installed. It splits your application into multiple layers (dependencies, resources, classes) so that incremental builds are extremely fast — only changed layers are rebuilt and pushed.

```xml
<!-- pom.xml — Maven JIB plugin -->
<plugin>
  <groupId>com.google.cloud.tools</groupId>
  <artifactId>jib-maven-plugin</artifactId>
  <version>3.4.0</version>
  <configuration>
    <!-- Base image to build from (JRE only, not JDK) -->
    <from>
      <image>eclipse-temurin:21-jre-alpine</image>
    </from>
    <!-- Target image to push to -->
    <to>
      <image>registry.example.com/myorg/myapp:${project.version}</image>
    </to>
    <container>
      <mainClass>com.example.MyApplication</mainClass>
      <jvmFlags>
        <jvmFlag>-XX:+UseContainerSupport</jvmFlag>
        <jvmFlag>-XX:MaxRAMPercentage=75.0</jvmFlag>
      </jvmFlags>
      <ports>
        <port>8080</port>
      </ports>
      <!-- Use current timestamp for reproducibility -->
      <creationTime>USE_CURRENT_TIMESTAMP</creationTime>
    </container>
  </configuration>
</plugin>
```

```bash
# Build and push to remote registry
mvn jib:build

# Build to local Docker daemon (for testing)
mvn jib:dockerBuild
```

JIB is particularly powerful in CI environments where you may not want to run a Docker daemon.

---

## Distroless Images

**Distroless** images (maintained by Google) contain only your application and its runtime dependencies — no shell (`/bin/sh`), no package manager, no OS utilities. This dramatically reduces the attack surface.

```dockerfile
# Using Google's distroless Java image
FROM gcr.io/distroless/java21-debian12

COPY --from=builder /app/target/myapp.jar /app/myapp.jar

EXPOSE 8080

CMD ["/app/myapp.jar"]
```

The downside is that debugging is harder because you cannot `docker exec` into a shell. A common pattern is to use distroless in production and a regular alpine image in development.

---

## Best Practices Summary

**Layer ordering matters.** Always copy `pom.xml` and download dependencies before copying source code. This way, rebuilds caused by source changes don't re-download all dependencies.

**Never run as root.** Create a dedicated non-root user in your Dockerfile. Many container security scanners will flag root-running containers.

**Use specific image tags, never `latest`.** `FROM eclipse-temurin:21` is much safer than `FROM eclipse-temurin:latest` because `latest` can change unexpectedly and break your build.

**Use `.dockerignore`.** Just like `.gitignore`, this file tells Docker what NOT to copy into the build context. A missing `.dockerignore` can cause huge build contexts (e.g., copying `target/` or `node_modules/` unintentionally).

```
# .dockerignore
target/
.git/
*.md
.mvn/wrapper/maven-wrapper.jar
```

**Set explicit memory limits.** Always run Java containers with a memory limit (`docker run -m 1g`) and set `-XX:MaxRAMPercentage` accordingly. Never let the JVM run in a container without a memory constraint.

**Use health checks.** For Spring Boot, the Actuator `/actuator/health` endpoint is perfect.

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1
```
