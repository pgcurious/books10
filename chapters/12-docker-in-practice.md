# Chapter 12: Docker in Practice

> **First Principles Question**: Now that you understand what containers really are (Chapter 11), how do you use Docker effectively? What are the patterns, pitfalls, and best practices for production container usage?

---

## Chapter Overview

Docker made containers accessible. Before Docker, using containers meant deep Linux knowledge and manual scripting. Docker provided a simple interface, standardized image format, and ecosystem that transformed how we deploy software.

This chapter focuses on practical Docker usage—not just commands, but the patterns and practices that lead to reliable, secure, and efficient containerized applications.

**What readers will understand after this chapter:**
- Docker architecture and components
- Writing effective Dockerfiles
- Image optimization techniques
- Container networking and storage
- Docker Compose for multi-container applications
- Production-ready container practices
- Debugging containerized applications

---

## Section 1: Docker Architecture

### 1.1 The Components

```
┌─────────────────────────────────────────────────────────────────┐
│                        Docker CLI                               │
│                     (docker build, run, push)                   │
└───────────────────────────┬─────────────────────────────────────┘
                            │ REST API
┌───────────────────────────▼─────────────────────────────────────┐
│                      Docker Daemon                              │
│                        (dockerd)                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │   Images    │  │ Containers  │  │   Networks / Volumes    │ │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘ │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│                       containerd                                │
│                  (container runtime)                            │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│                          runc                                   │
│              (OCI runtime, creates containers)                  │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│                    Linux Kernel                                 │
│            (namespaces, cgroups, overlay fs)                   │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Images vs. Containers

**Image:**
- Read-only template
- Layered filesystem
- Built from Dockerfile
- Stored in registry

**Container:**
- Running instance of an image
- Has writable layer
- Has process(es)
- Temporary by default

**Analogy:**
- Image = Recipe
- Container = Cake baked from recipe

```bash
# List images
docker images

# List containers (running)
docker ps

# List all containers (including stopped)
docker ps -a
```

### 1.3 Registries

**Where Images Live:**
```
┌─────────────────────────────────────────────────────────────────┐
│                        Registries                               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────┐  │
│  │  Docker Hub     │  │  ECR (AWS)      │  │  GCR (Google)  │  │
│  │  (public)       │  │  (private)      │  │  (private)     │  │
│  └─────────────────┘  └─────────────────┘  └────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**Image Naming:**
```
registry/repository:tag

Examples:
nginx                           → docker.io/library/nginx:latest
nginx:1.21                      → docker.io/library/nginx:1.21
mycompany/myapp:v1.2.3          → docker.io/mycompany/myapp:v1.2.3
123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
gcr.io/my-project/myapp:latest
```

---

## Section 2: Writing Effective Dockerfiles

### 2.1 Dockerfile Basics

```dockerfile
# Base image
FROM openjdk:17-slim

# Metadata
LABEL maintainer="team@example.com"
LABEL version="1.0"

# Set working directory
WORKDIR /app

# Copy files
COPY target/myapp.jar app.jar

# Environment variables
ENV JAVA_OPTS="-Xmx512m"

# Expose port (documentation)
EXPOSE 8080

# Run command
CMD ["java", "-jar", "app.jar"]
```

### 2.2 Layer Optimization

**Bad: Every RUN creates a layer**
```dockerfile
FROM ubuntu:22.04
RUN apt-get update
RUN apt-get install -y python3
RUN apt-get install -y pip
RUN apt-get clean
RUN rm -rf /var/lib/apt/lists/*
# 5 layers, cleanup doesn't reduce image size!
```

**Good: Combine commands**
```dockerfile
FROM ubuntu:22.04
RUN apt-get update && \
    apt-get install -y python3 pip && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
# 1 layer, cleanup actually reduces size
```

### 2.3 Build Cache Optimization

**Bad: Copy everything first**
```dockerfile
FROM node:18
WORKDIR /app
COPY . .                    # Any change invalidates cache
RUN npm install             # Reinstalls everything
RUN npm run build
```

**Good: Copy dependencies first**
```dockerfile
FROM node:18
WORKDIR /app
COPY package*.json ./       # Only dependency files
RUN npm install             # Cached unless package.json changes
COPY . .                    # Source code changes don't affect npm install
RUN npm run build
```

### 2.4 Multi-Stage Builds

**The Problem:**
Build tools in production images add size and security risk.

**Single-Stage (Bad):**
```dockerfile
FROM maven:3.8-openjdk-17
WORKDIR /app
COPY . .
RUN mvn clean package
CMD ["java", "-jar", "target/app.jar"]
# Image includes Maven, all build tools (~500MB+)
```

**Multi-Stage (Good):**
```dockerfile
# Build stage
FROM maven:3.8-openjdk-17 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

# Runtime stage
FROM openjdk:17-slim
WORKDIR /app
COPY --from=builder /app/target/app.jar ./app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
# Final image has only JRE and app (~200MB)
```

### 2.5 Java-Specific Dockerfile

```dockerfile
# Build stage
FROM eclipse-temurin:17-jdk AS builder
WORKDIR /app

# Install build tools
RUN apt-get update && apt-get install -y maven && rm -rf /var/lib/apt/lists/*

# Cache dependencies
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Build application
COPY src ./src
RUN mvn clean package -DskipTests -B

# Extract layers (Spring Boot 2.3+)
RUN java -Djarmode=layertools -jar target/*.jar extract

# Runtime stage
FROM eclipse-temurin:17-jre
WORKDIR /app

# Create non-root user
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

# Copy layers in order of change frequency
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./

# Set ownership
RUN chown -R appuser:appgroup /app
USER appuser

EXPOSE 8080
ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
```

### 2.6 Security Best Practices

**Don't Run as Root:**
```dockerfile
# Create user
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

# Switch to user
USER appuser
```

**Don't Include Secrets:**
```dockerfile
# BAD: Secret in image
ENV DATABASE_PASSWORD=secret123

# GOOD: Pass at runtime
# docker run -e DATABASE_PASSWORD=secret123 myapp
```

**Use Specific Tags:**
```dockerfile
# BAD: Unpredictable
FROM node:latest

# GOOD: Reproducible
FROM node:18.17.0-slim
```

**Minimize Attack Surface:**
```dockerfile
# Use slim/alpine images
FROM python:3.11-slim

# Remove unnecessary packages
RUN apt-get purge -y --auto-remove && \
    rm -rf /var/lib/apt/lists/*
```

---

## Section 3: Container Networking

### 3.1 Network Types

**Bridge (Default):**
```
┌─────────────────────────────────────────────┐
│  Host                                       │
│  ┌───────────────────────────────────────┐  │
│  │  docker0 bridge (172.17.0.1)          │  │
│  │                                        │  │
│  │    ┌─────────┐      ┌─────────┐       │  │
│  │    │Container│      │Container│       │  │
│  │    │172.17.0.2│     │172.17.0.3│      │  │
│  │    └─────────┘      └─────────┘       │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

**Host:**
```bash
# Container uses host's network directly
docker run --network host nginx
# nginx listens on host's port 80
```

**None:**
```bash
# No networking
docker run --network none myapp
```

### 3.2 User-Defined Networks

```bash
# Create network
docker network create mynetwork

# Run containers on network
docker run -d --name db --network mynetwork postgres
docker run -d --name app --network mynetwork myapp

# Containers can reach each other by name
# app can connect to "db:5432"
```

**DNS Resolution:**
```
┌─────────────────────────────────────────────────┐
│  mynetwork                                      │
│                                                 │
│  ┌─────────┐ "db"        ┌─────────┐ "app"     │
│  │   db    │◄────────────│   app   │           │
│  │ postgres │             │         │           │
│  └─────────┘             └─────────┘           │
│                                                 │
│  Embedded DNS: container names → IPs            │
└─────────────────────────────────────────────────┘
```

### 3.3 Port Mapping

```bash
# Map container port to host port
docker run -p 8080:80 nginx
# Host:8080 → Container:80

# Map to specific interface
docker run -p 127.0.0.1:8080:80 nginx
# Only localhost can access

# Random host port
docker run -p 80 nginx
docker port <container_id>
# Shows assigned host port
```

---

## Section 4: Data Persistence

### 4.1 The Problem

```bash
docker run -d postgres
# Data stored in container's writable layer

docker rm postgres
# Data gone!
```

### 4.2 Volumes

**Named Volumes (Recommended):**
```bash
# Create volume
docker volume create mydata

# Use volume
docker run -v mydata:/var/lib/postgresql/data postgres

# Data persists across container recreations
docker rm postgres
docker run -v mydata:/var/lib/postgresql/data postgres
# Data still there!
```

**Volume Commands:**
```bash
docker volume ls              # List volumes
docker volume inspect mydata  # Show details
docker volume rm mydata       # Remove volume
docker volume prune           # Remove unused volumes
```

### 4.3 Bind Mounts

**Mount Host Directory:**
```bash
# Development: mount source code
docker run -v $(pwd):/app node:18

# Share configuration
docker run -v /etc/myapp:/etc/myapp:ro myapp
```

**Read-Only Mount:**
```bash
docker run -v /config:/config:ro myapp
# Container can't modify /config
```

### 4.4 tmpfs Mounts

**In-Memory Storage:**
```bash
docker run --tmpfs /tmp myapp
# /tmp is in RAM, not written to disk
# Useful for secrets, temporary data
```

### 4.5 Comparison

| Type | Use Case | Persistence | Performance |
|------|----------|-------------|-------------|
| Volume | Production data | Yes | Good |
| Bind Mount | Development, config | Host-dependent | Good |
| tmpfs | Secrets, temp | No (RAM) | Excellent |

---

## Section 5: Docker Compose

### 5.1 Why Compose?

**Without Compose:**
```bash
docker network create myapp
docker run -d --name db --network myapp \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:15
docker run -d --name redis --network myapp redis:7
docker run -d --name app --network myapp \
  -p 8080:8080 \
  -e DATABASE_URL=postgres://db:5432 \
  -e REDIS_URL=redis://redis:6379 \
  myapp:latest
```

**With Compose:**
```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    image: myapp:latest
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://db:5432/myapp
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis

  db:
    image: postgres:15
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB=myapp

  redis:
    image: redis:7

volumes:
  pgdata:
```

```bash
docker compose up -d    # Start all services
docker compose down     # Stop and remove
docker compose logs     # View logs
docker compose ps       # Status
```

### 5.2 Development Compose File

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
      - "5005:5005"  # Debug port
    volumes:
      - ./src:/app/src:ro           # Hot reload source
      - ./target:/app/target         # Build output
      - maven-cache:/root/.m2        # Cache dependencies
    environment:
      - SPRING_PROFILES_ACTIVE=dev
      - JAVA_TOOL_OPTIONS=-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:15
    ports:
      - "5432:5432"  # Expose for local tools
    environment:
      - POSTGRES_PASSWORD=devpassword
      - POSTGRES_DB=myapp
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7
    ports:
      - "6379:6379"

volumes:
  maven-cache:
```

### 5.3 Production Compose File

```yaml
version: '3.8'

services:
  app:
    image: myregistry/myapp:${VERSION:-latest}
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - DATABASE_URL=postgres://db:5432/myapp
    secrets:
      - db_password
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  db:
    image: postgres:15
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
    secrets:
      - db_password

secrets:
  db_password:
    external: true

volumes:
  pgdata:
```

### 5.4 Override Files

```yaml
# docker-compose.yml (base)
services:
  app:
    image: myapp
    environment:
      - LOG_LEVEL=INFO

# docker-compose.override.yml (dev, auto-loaded)
services:
  app:
    build: .
    volumes:
      - ./src:/app/src
    environment:
      - LOG_LEVEL=DEBUG

# docker-compose.prod.yml
services:
  app:
    deploy:
      replicas: 3
```

```bash
# Development (uses override automatically)
docker compose up

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up
```

---

## Section 6: Production Practices

### 6.1 Health Checks

**In Dockerfile:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

**In Compose:**
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

### 6.2 Logging

**Docker Logging Drivers:**
```bash
# JSON file (default)
docker run --log-driver json-file --log-opt max-size=10m myapp

# Fluentd
docker run --log-driver fluentd --log-opt fluentd-address=localhost:24224 myapp

# AWS CloudWatch
docker run --log-driver awslogs --log-opt awslogs-group=myapp myapp
```

**Application Logging Best Practices:**
```java
// Log to stdout (Docker captures it)
// Use JSON format for structured logging
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "INFO",
  "message": "Request processed",
  "requestId": "abc-123",
  "duration": 45
}
```

### 6.3 Resource Limits

```bash
# Memory limit
docker run --memory=512m myapp

# CPU limit (number of CPUs)
docker run --cpus=0.5 myapp

# Combined
docker run --memory=512m --cpus=0.5 myapp
```

**In Compose:**
```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
```

### 6.4 Image Scanning

```bash
# Docker Scout (built-in)
docker scout cves myapp:latest

# Trivy
trivy image myapp:latest

# Snyk
snyk container test myapp:latest
```

**CI/CD Integration:**
```yaml
# GitHub Actions example
- name: Scan image
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    exit-code: '1'
    severity: 'CRITICAL,HIGH'
```

### 6.5 Secrets Management

**Bad: Environment Variables**
```bash
docker run -e DB_PASSWORD=secret myapp
# Visible in docker inspect, process list
```

**Better: Docker Secrets**
```bash
# Create secret
echo "mysecret" | docker secret create db_password -

# Use in service
docker service create --secret db_password myapp

# In container: /run/secrets/db_password
```

**Best: External Secret Manager**
```yaml
# Use AWS Secrets Manager, HashiCorp Vault, etc.
services:
  app:
    environment:
      - DB_PASSWORD_ARN=arn:aws:secretsmanager:...
    # App fetches secret at runtime
```

---

## Section 7: Debugging Containers

### 7.1 Container Inspection

```bash
# View container details
docker inspect <container_id>

# Specific fields
docker inspect --format='{{.State.Status}}' <container_id>
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <container_id>
```

### 7.2 Logs

```bash
# View logs
docker logs <container_id>

# Follow logs
docker logs -f <container_id>

# Last N lines
docker logs --tail 100 <container_id>

# With timestamps
docker logs -t <container_id>
```

### 7.3 Exec Into Container

```bash
# Interactive shell
docker exec -it <container_id> /bin/bash

# If no bash, try sh
docker exec -it <container_id> /bin/sh

# Run command
docker exec <container_id> ps aux
```

### 7.4 Debug Without Shell

**Problem:** Minimal images (distroless, scratch) have no shell.

**Solution: Debug container**
```bash
# Share process namespace
docker run -it --pid=container:<target_container_id> busybox

# Now you can see target's processes, use debugging tools
```

**Solution: Ephemeral container (Docker 23+)**
```bash
docker debug <container_id>
# Adds debugging tools temporarily
```

### 7.5 Network Debugging

```bash
# See container networks
docker network inspect bridge

# Test connectivity from container
docker exec <container_id> ping other-container
docker exec <container_id> curl http://other-container:8080

# View port mappings
docker port <container_id>
```

### 7.6 Resource Usage

```bash
# Live stats
docker stats

# Specific container
docker stats <container_id>

# One-shot (no streaming)
docker stats --no-stream
```

---

## Section 8: CI/CD Integration

### 8.1 Building in CI

**GitHub Actions:**
```yaml
name: Build and Push

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Login to Registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### 8.2 Testing in Containers

**Testcontainers:**
```java
@SpringBootTest
@Testcontainers
class IntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb");

    @DynamicPropertySource
    static void setProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Test
    void testWithRealDatabase() {
        // Test against real PostgreSQL
    }
}
```

### 8.3 Image Tagging Strategy

```bash
# Git SHA (immutable)
myapp:abc123def

# Semantic version
myapp:1.2.3

# Branch name (latest on branch)
myapp:main

# Latest (dangerous in production)
myapp:latest

# Recommended: Multiple tags
docker tag myapp:abc123 myapp:1.2.3
docker tag myapp:abc123 myapp:latest
```

---

## Section 9: Now You Understand Why...

Armed with this foundation, you can now understand:

- **Why multi-stage builds matter**: Smaller images, no build tools in production

- **Why layer ordering in Dockerfiles matters**: Build cache optimization

- **Why you shouldn't run as root**: Security, principle of least privilege

- **Why named volumes exist**: Data persistence across container lifecycle

- **Why Compose uses YAML**: Declarative, version-controlled infrastructure

- **Why health checks are important**: Orchestrators need to know container state

- **Why image scanning is necessary**: Containers inherit vulnerabilities from base images

---

## Practical Exercises

### Exercise 1: Optimize a Dockerfile
Take this Dockerfile and optimize it:
```dockerfile
FROM ubuntu:22.04
RUN apt-get update
RUN apt-get install -y openjdk-17-jdk maven
COPY . /app
WORKDIR /app
RUN mvn clean package
CMD ["java", "-jar", "target/app.jar"]
```

### Exercise 2: Multi-Container Application
Create a Compose file for:
- Spring Boot application
- PostgreSQL database
- Redis cache
- Ensure proper startup order and health checks

### Exercise 3: Debug a Container
1. Run a container that's crashing
2. Use logs, inspect, and exec to find the problem
3. Fix and rebuild

### Exercise 4: CI/CD Pipeline
Create a GitHub Actions workflow that:
- Builds a Docker image
- Runs tests in containers
- Scans for vulnerabilities
- Pushes to a registry

---

## Key Takeaways

1. **Layer order matters**. Put rarely-changing content early, frequently-changing content late.

2. **Multi-stage builds are essential**. Separate build and runtime environments.

3. **Don't run as root**. Create a non-root user and use it.

4. **Use health checks**. Let orchestrators know if your container is healthy.

5. **Manage data explicitly**. Use volumes for persistence, bind mounts for development.

6. **Scan your images**. Vulnerabilities in base images become your vulnerabilities.

7. **Compose for local development**. Define your entire stack declaratively.

---

## Looking Ahead

Docker works great for running containers on a single machine. But what happens when you need dozens or hundreds of containers across multiple machines? Chapter 13 explores why Kubernetes exists—the problem of container orchestration at scale.

---

## Chapter 12 Summary Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         DOCKER IN PRACTICE                                   │
│                                                                             │
│  DOCKERFILE BEST PRACTICES                                                  │
│  ─────────────────────────                                                  │
│  • Layer order: dependencies first, source code last                       │
│  • Multi-stage: build in one stage, run in another                         │
│  • Specific tags: FROM node:18.17.0-slim, not :latest                     │
│  • Non-root user: USER appuser                                             │
│  • Minimal base: -slim, -alpine variants                                   │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  NETWORKING                                                                 │
│  ──────────                                                                 │
│                                                                             │
│  Bridge:     Default, containers on virtual network                        │
│  Host:       Container uses host's network                                 │
│  None:       No networking                                                 │
│  Custom:     User-defined with DNS resolution by name                      │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  STORAGE                                                                    │
│  ───────                                                                    │
│                                                                             │
│  Volumes:      docker volume create mydata                                 │
│                Managed by Docker, persist across containers                │
│                                                                             │
│  Bind Mounts:  -v $(pwd)/src:/app/src                                      │
│                Host directory mounted in container                         │
│                                                                             │
│  tmpfs:        --tmpfs /tmp                                                │
│                In-memory, for secrets/temp data                            │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  DOCKER COMPOSE                                                             │
│  ──────────────                                                             │
│                                                                             │
│  services:                                                                  │
│    app:                                                                     │
│      image: myapp                                                           │
│      depends_on:                                                            │
│        db:                                                                  │
│          condition: service_healthy                                        │
│    db:                                                                      │
│      image: postgres:15                                                     │
│      healthcheck:                                                           │
│        test: ["CMD", "pg_isready"]                                         │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  PRODUCTION CHECKLIST                                                       │
│  ────────────────────                                                       │
│                                                                             │
│  □ Health checks configured                                                │
│  □ Resource limits set (memory, CPU)                                       │
│  □ Non-root user                                                           │
│  □ Image scanned for vulnerabilities                                       │
│  □ Secrets managed properly (not in env vars)                              │
│  □ Logging to stdout/stderr                                                │
│  □ Specific image tags (not :latest)                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

*Next Chapter: Why Kubernetes Exists — The problem of container orchestration at scale*
