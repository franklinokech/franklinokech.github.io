---
title: 'Docker Best Practices'
date: 2024-09-10
permalink: /posts/2024/09/docker-best-practices/
tags:
  - docker
  - docker-compose
  - linux
  - best-practices
---

Docker is simple. Production is not.Most Docker problems don’t come from Docker itself—they come from small shortcuts that compound over time. In this post, we’ll cover practical Docker best practices that keep images small, builds fast, and deployments predictable.

## What is Docker?
For the uninitiated, docker is one of the containerization platform that helps packaging appliction and their corresponding dependencies with an aim of ensuring the application works in any environment in a more consistent and reliable eliminating the dreaded **"it works on my computer"** problem.

## What is Docker Compose?
Docker compose is a container orchestration platform that helps in configuraing multi app dockerised servics, all configured from a single Yaml file and controlled with single commands. Docker compose is useful in managing complex application that needs to speak to each.

## Prerequistes
To follow along with this tutorial, you will need the following installed in your platform
- Docker version 28.3.3, build 980b856
  - Follow the official guide: [docker installation](https://docs.docker.com/engine/install/)
- Docker Compose version v2.39.1
  - Follow the official guide: [docker compose installation](https://docs.docker.com/engine/install/)

## Best Practices
### Always use the Base Image

``` bash
FROM python:3.9-slim
```

This gives the benefit of having faster and deployment, reduce your overall storage cost, reduce your security attack vector

### Insist on official Base Images
Use official images like alphine, debian-slim, ubunt

```bash
FROM python:3.9-alpine
```
This gives better community support, these images gets regular updates and are trusted

### Leverage Multi-build paradigm
Large images can be broken down into a build and a runtime stages. This can be implemented as shown below.

```bash
# Stage 1: Build stage
FROM python:3.9.20-slim AS builder
ARG ENVIRONMENT=dev
WORKDIR /app/
COPY requirements/ /app/requirements/
RUN if [ "$ENVIRONMENT" = "prod" ]; then \
        pip install --no-cache-dir -r /app/requirements/prod.txt --target /app/deps; \
    else \
        pip install --no-cache-dir -r /app/requirements/dev.txt --target /app/deps; \
    fi

# Stage 2: Final runtime stage
FROM python:3.9.20-slim
COPY --from=builder /app/deps /usr/local/lib/python3.9/site-packages
COPY ./src /app

```

This ensures separation of concerns, smaller final images, deploys faster and secure since no build tool is used in production

### Minimize image layers
Each docker instruction line added to a dockerfile creates a layer. You can mitigate this by chaining multiple instructions into a single **RUN** instruction.

```bash
RUN apt-get update && apt-get install -y \ 
    curl \    
    vim \   
    && rm -rf /var/lib/apt/lists/*

```
this is beneficial in reduce final image size, better caching,better security by reducing attack vector.

### Use .dockerignore File
This works like the .gitignore File. It is used to exclude unnecessary files in the image.

```bash
.git
.gitignore
README.md
.env
*.log
node_modules

```
This gives smaller images which means faster builds, prevent sensitive files from being included, better perfomance.

### SET WORKDIR
This is implemented as

```bash
WORKDIR /app
```

This gives a consistent working directory, elimates the need for the **RUN cd** command and provides better readability.

### Use specific image tag

```bash
FROM node:16.13.1-alpine

```
This ensure a more consistent, reproducible which works in a more predicable way. It also ensure full control over security updates which may break things unexpectadly.

### Clean Up After Installations

```bash
RUN apt-get update && apt-get install -y \     
    build-essential \     
    && rm -rf /var/lib/apt/lists/*

```
This ensure smaller image size,, faster deployment, reduce attack vector, better security

### Limit Container Priviledges

```bash
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

```

This is benefical from a security point of view. It ensure the principle of least priviledge is adhered to, reduced security risk, better container isolation and ensure compliance with security standards.

### Specify a Health Check

```bash
HEALTHCHECK CMD curl --fail http://localhost:8080/health || exit 1

```
This enables automatic container health check monitoring, better orchestration support, faster failure detection and improves readability.

### Use CMD and ENTRYPOINT Appropriately.

```bash
ENTRYPOINT ["python", "app.py"]
CMD ["--help"]

```

This is beneficial in ensuring default behaviour is defined, flexible container usage, easier paramters override, better container reusability.

### Label Your Image

```bash
LABEL maintainer="franklinokecha@gmail.com"
LABEL version="1.0.0"
LABEL description="Production-ready FastAPI application"

```
This ensures better image organization, collaboration, metadata for automation and compliance tracking.

### Minimize Image Size

```bash
RUN pip install --no-cache-dir -r requirements.txt

```

this ensures smaller images, reduce storage cost, better perfomance and reduce security attack vector.

#### Avoid Hard Coding Ports.

```bash
EXPOSE ${PORT:-8080}

```
This is critical in ensuring environment flexibility, make the image more portable, easier configuration and management, reduced port conflicts.

### Use Signals Correctly in ENTRYPOINT.

```bash
ENTRYPOINT ["exec", "myapp"]

```

This ensures proper signal handling, graceful shutdowns, better orchestration support, improved reliability.

### Log Verbosely for Easier Debugging

```bash
RUN echo "Building app..." && \     
    echo "Step 1: Installing dependencies"

```

This is beneficial in ensuring easier troubleshooting, better build visibility, faster issue resolution, improves debugging experience.

### Set Permissions Correctly

```bash
COPY --chown=appuser:appgroup myapp /usr/local/bin/

```

This is critical security compliance which reduces priviledges escalations, better container isolation, proper file access control.

### Always use Immutable Image tag

```bash
# Use specific version tags instead of :latest
FROM python:3.9.20-slim

```

This is key in ensuring reproducible builds, no unexpected changes, better rollback capability, improves reliability.

### Optimize Docker Cache Layer

```bash
COPY requirements.txt /app/
RUN pip install -r /app/requirements.txt
COPY . /app/

```

This is beneficial in ensuring faster incremental builds, better cache utilization, reduce build time, improves development experience and workflow.

### Use Signed Images

```bash
export DOCKER_CONTENT_TRUST=1

```
This is a good security practice which ensures image authenticity verification, integrity checking, trusted source validation, better security posture.

### Encrypt Secrets and Sensitive Data

**Never** store sensitive information in Dockerfiles. Use environment variables

```bash
ENV DATABASE_URL=${DATABASE_URL}
ENV API_KEY=${API_KEY}

```

This is beneficial by ensuring no secrets in Images, ensure environment specific configurations, better security, ensure compliance with security standards.

### Use Envrionment Variables

```bash
ENV NODE_ENV=production
ENV PORT=3000

```

This ensures flexible configurations, environment specific settings, easy maintainance , better portability.

### Document Dockerfile and Container Usage
This can be achieved by use of comments

```bash
# Install system dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Set working directory for the application
WORKDIR /app

```

This is beneficial in collaboration, better onboarding,easier maintainance , reduce errors

### Log outputs clearly
Ensure your application logs to stdout and stderr

```bash
# Your application should log to stdout/stderr
CMD ["python", "-u", "app.py"]

```

This ensures better logging driver integration, better monitoring, better debugging, centralised log management.

### Handle Signals for Graceful Shutdown
Ensure your app handles OS signals properly.

```bash
# Use exec form for proper signal handling
ENTRYPOINT ["python", "app.py"]

```

this is beneficial in ensuring graceful shutdowns, better orchestration, improves reliability, reduce data loss.

### Use hadolint for Linting

```bash
# Install hadolint
curl -Lo hadolint "https://github.com/hadolint/hadolint/releases/latest/download/hadolint-$(uname -s)-$(uname -m)"
chmod +x hadolint

# Lint your Dockerfile
./hadolint Dockerfile

```

Linting helps catch common mistakes, enforce best practices, improve code quality, automated code review.

## Docker Compose Best Practices Implementation

### Use .env Files for Configuration

```bash
services:
  db:
    image: postgres:12.1-alpine
    environment:
      - POSTGRES_USER
      - POSTGRES_PASSWORD
      - POSTGRES_DB

```

This ensures environment specific configurations, no hard coded values, easy configuration management, better security.

### Monitor Container Health

```bash
services:
  api:
    image: fastapiapp:latest
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
      retries: 3

```

This is beneficial in ensuring automatic heallth monitoring, faster failure detection, better orchestration, improve reliability.

### Limit Resources for Better Performance

```bash
services:
  api:
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M

```
This is resource optimization stratergy which ensure resource usage are predictable, better stability, cost control, improves perfomance.

### Installing Environment-Specific Dependencies

```bash
RUN if [ "$ENVIRONMENT" = "prod" ]; then \
        pip install --no-cache-dir -r /app/requirements/prod.txt; \
    else \
        pip install --no-cache-dir -r /app/requirements/dev.txt;

```

This ensures environment specific build, smaller production images, better security, optimized deployment

### Use Docker Compose's Watch Feature for Development

```bash
develop:
  watch:
    - action: sync
      path: ./src/
      target: /app/src/
    - action: rebuild
      path: requirements/dev.txt

```

This ensures better developer experience by ensuring faster development time, automatic code synchronization, reduce manaual builds

### Use Volumes for Persistence

```bash
services:   
   db:  
     image: postgres   
     volumes:     
        - db-data:/var/lib/postgresql/data
volumes:  
    db-data:

```

This is a data management practice aimed at ensuring data persistance, better data management, easier backuo, improve reliability.

### Enable Networking Isolation in Docker Compose

```bash
networks:
  app_network:
    driver: bridge

services:
  app:
    networks:
      - frontend
  db:
    networks:
      - backend

networks:
  frontend:
  backend:

```

This security practice ensures better security, network isolation, controlled communication, improved architecture.

### restart: always

```bash
restart: always

```

This ensures automatic recovery, better availability, reduce manual intervention, improve reliability.

### Deploy with Replicas (Scaling)

```bash
services: 
  app:   
    image: myapp  
    deploy:     
      replicas: 3

```

This is beneficaling for scalling the application by load distribution, ensuring high availability, better perfomance and improve scalability.

### Limit Container Privileges

```bash
services:
  app:
    image: myapp
    cap_drop:
      - ALL
    read_only: true

```

This security best practice reduces attack vectors, ensure compliance with security standards, improves container isolation

### Enforce Image Pull Policies

```bash
services:
  app:
    image: myapp:1.0.0
    pull_policy: always

```

this security best practice ensures latest security patches are integrated, consistent deployment, better security, reduce vulnerabilities.

### Run Containers in Non-Privileged Mode

```bash
services:
  app:
    image: myapp
    privileged: false

```

This is a security best practice which is beneficial in reducing host access, improve container isolation, ensure compliance with security standards, promoting better security.


### Use Named Volumes and Networks

```bash
services:
  app:
    volumes:
      - app-data:/var/www/html

volumes:
  app-data:

```
This ensures better organization, reusable resources, easier management, improve clarity

### Use Service Dependencies

```bash
services:
  app:
    depends_on:
      - db

```

This is key in ensuring controlled startup behavior, better reliability, reduce startup failures, improve orchestration.

### Log Configuration

```bash
services:
  app:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

```

This ensures controlled log growth, better disk management, centralised logging, improve monitoring.

### Use External Configuration Files

```bash
services:
  web:
    image: nginx
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf

```

This is beneficial in ensuring easier configuration management, version controls for configs, better maintainability, reduced builds.

### Use Labels for Metadata

```bash
services:
  app:
    labels:
      - "maintainer=franklinokecha@gmail.com"
      - "version=1.0"

```
This gives better organization, team collaboration, automation support, improve management.

### Testing Before Deploying

```bash
# Validate configuration
docker compose config

# Test the setup
docker compose up --dry-run

```

This helps in catching configuration errors, validate before deployment, reduce deployment failures, better reliability.


## Production Grade Example for a FASTAPI Application

### Dockerfile

```bash
# Multi-stage build for production
FROM python:3.9.20-slim AS builder

# Set build arguments
ARG ENVIRONMENT=prod
ARG BUILD_DATE
ARG VCS_REF

# Set labels
LABEL maintainer="maher.naija@gmail.com"
LABEL org.opencontainers.image.created=$BUILD_DATE
LABEL org.opencontainers.image.revision=$VCS_REF
LABEL org.opencontainers.image.version="1.0.0"

# Install build dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /app

# Copy requirements first for better caching
COPY requirements/ /app/requirements/

# Install dependencies based on environment
RUN if [ "$ENVIRONMENT" = "prod" ]; then \
        pip install --no-cache-dir -r /app/requirements/prod.txt --target /app/deps; \
    else \
        pip install --no-cache-dir -r /app/requirements/dev.txt --target /app/deps; \
    fi

# Production stage
FROM python:3.9.20-slim

# Set environment variables
ENV PYTHONPATH=/app/deps
ENV PYTHONUNBUFFERED=1
ENV PORT=8000

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Set working directory
WORKDIR /app

# Copy dependencies from builder stage
COPY --from=builder /app/deps /app/deps

# Copy application code
COPY --chown=appuser:appgroup ./src /app/src

# Switch to non-root user
USER appuser

# Expose port
EXPOSE ${PORT}

# Health check
HEALTHCHECK CMD curl --fail http://localhost:${PORT}/health || exit 1

# Set entrypoint and command
ENTRYPOINT ["python", "-u", "src/main.py"]
CMD ["--host", "0.0.0.0", "--port", "8000"]

```

### docker-compose.yml

```bash
version: '3.8'

services:
  app:
    build:
      context: .
      args:
        ENVIRONMENT: ${ENVIRONMENT:-prod}
        BUILD_DATE: ${BUILD_DATE:-$(date -u +'%Y-%m-%dT%H:%M:%SZ')}
        VCS_REF: ${VCS_REF:-$(git rev-parse --short HEAD)}
    image: fastapi-app:${TAG:-latest}
    container_name: fastapi-app
    restart: unless-stopped
    ports:
      - "${PORT:-8000}:8000"
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
      - API_KEY=${API_KEY}
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
    networks:
      - app-network
    labels:
      - "maintainer=maher.naija@gmail.com"
      - "version=${TAG:-latest}"
      - "environment=${ENVIRONMENT:-prod}"

  db:
    image: postgres:15-alpine
    container_name: postgres-db
    restart: unless-stopped
    environment:
      - POSTGRES_DB=${POSTGRES_DB}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M

  redis:
    image: redis:7-alpine
    container_name: redis-cache
    restart: unless-stopped
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network
    deploy:
      resources:
        limits:
          cpus: '0.25'
          memory: 128M

networks:
  app-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16

volumes:
  postgres-data:
    driver: local
  redis-data:
    driver: local

```

### .env File

```bash
# Application Configuration
ENVIRONMENT=prod
TAG=v1.0.0
PORT=8000

# Database Configuration
POSTGRES_DB=fastapi_app
POSTGRES_USER=app_user
POSTGRES_PASSWORD=secure_password_here
DATABASE_URL=postgresql://app_user:secure_password_here@db:5432/fastapi_app

# Redis Configuration
REDIS_URL=redis://redis:6379/0

# API Configuration
API_KEY=your_secure_api_key_here

# Build Information
BUILD_DATE=2025-01-15T10:00:00Z
VCS_REF=abc1234

```

## Conclusion
Mastering Docker is essential for DevOps, Data Engineers and virtually anyone who want to share some code to production. Doing this in a fast, secure and scalable way is essential for improving the app perfomance. These workflows help you build faster, reduce risk and ensure reliability of your application in any envrionment.

### Key Takeaways:
1. **Security First** - Always run containers with minimal priviledges.
1. **Perfomance Matter** - Use multi stage build stratergy and optimize image layers.
1. **Monitoring is Key** - Implement proper health check and logging
1. **Configuration Management** - Use environment variable and external configs
1. **Testing and Validation** - Test configs before deployment
1. **Resource Management** - Set appropritae limits and reservations.
1. **Documentation** - Document everything for team collaboration.


