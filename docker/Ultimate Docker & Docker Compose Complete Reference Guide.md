# Ultimate Docker & Docker Compose Complete Reference Guide

## Table of Contents
1. [Docker Fundamentals](#docker-fundamentals)
2. [Dockerfile Complete Reference](#dockerfile-complete-reference)
3. [Docker Compose Complete Reference](#docker-compose-complete-reference)
4. [Docker CLI Commands](#docker-cli-commands)
5. [Docker Networks](#docker-networks)
6. [Docker Volumes](#docker-volumes)
7. [Docker Security](#docker-security)
8. [Performance & Optimization](#performance--optimization)
9. [Debugging & Troubleshooting](#debugging--troubleshooting)
10. [Production Best Practices](#production-best-practices)
11. [Advanced Scenarios](#advanced-scenarios)
12. [Complete Examples](#complete-examples)

---

## Docker Fundamentals

### Core Concepts

#### Container vs Image
```bash
# Image: Blueprint/Template (read-only)
docker images

# Container: Running instance of an image
docker ps
```

#### Docker Architecture
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Docker CLI    │───▶│  Docker Daemon  │───▶│   Containers    │
│   (Client)      │    │   (dockerd)     │    │    Images       │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

#### Container Lifecycle
```bash
# Create → Start → Run → Stop → Remove
docker create → docker start → docker run → docker stop → docker rm
```

---

## Dockerfile Complete Reference

### All Instructions Explained

#### FROM - Base Image
```dockerfile
# Basic usage
FROM ubuntu:20.04
FROM node:18-alpine
FROM python:3.9-slim

# Multi-stage builds
FROM node:18-alpine AS builder
FROM nginx:alpine AS runtime

# Scratch (empty base)
FROM scratch

# Platform-specific
FROM --platform=linux/amd64 node:18-alpine

# Using ARG before FROM
ARG BASE_IMAGE=node:18-alpine
FROM ${BASE_IMAGE}
```

#### MAINTAINER (Deprecated) vs LABEL
```dockerfile
# Deprecated
MAINTAINER John Doe <john@example.com>

# Preferred
LABEL maintainer="john@example.com"
LABEL version="1.0"
LABEL description="My awesome application"
LABEL org.opencontainers.image.source="https://github.com/user/repo"
```

#### RUN - Execute Commands
```dockerfile
# Shell form (runs in shell)
RUN apt-get update && apt-get install -y curl

# Exec form (no shell)
RUN ["apt-get", "update"]
RUN ["/bin/bash", "-c", "echo hello"]

# Multi-line with line continuation
RUN apt-get update && \
    apt-get install -y \
        curl \
        wget \
        git && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Using here documents (BuildKit)
RUN <<EOF
    apt-get update
    apt-get install -y curl
    apt-get clean
EOF

# Cache busting
RUN apt-get update && apt-get install -y \
    package1 \
    package2 \
    && rm -rf /var/lib/apt/lists/*
```

#### COPY vs ADD
```dockerfile
# COPY - Simple file copying (preferred)
COPY package.json /app/
COPY src/ /app/src/
COPY --chown=node:node . /app/

# Wildcard copying
COPY package*.json ./
COPY *.conf /etc/

# Multi-stage copy
COPY --from=builder /app/dist /usr/share/nginx/html/

# Copy with specific permissions
COPY --chmod=755 script.sh /usr/local/bin/

# ADD - Extended functionality
ADD https://example.com/file.tar.gz /tmp/  # Download from URL
ADD archive.tar.gz /extracted/             # Auto-extract archives
ADD --chown=user:group file.txt /app/      # With ownership
```

#### WORKDIR - Working Directory
```dockerfile
WORKDIR /app
WORKDIR $HOME/app        # Using environment variable
WORKDIR /path/to/workdir

# Creates directory if it doesn't exist
WORKDIR /app/nested/deep/directory
```

#### ENV - Environment Variables
```dockerfile
# Single variable
ENV NODE_ENV=production

# Multiple variables (legacy format)
ENV NODE_ENV=production \
    PORT=3000 \
    DEBUG=false

# Multiple variables (preferred format)
ENV NODE_ENV=production
ENV PORT=3000
ENV DEBUG=false

# Using variables
ENV PATH="/app/bin:${PATH}"
ENV DATABASE_URL="postgresql://${DB_USER}:${DB_PASS}@${DB_HOST}:${DB_PORT}/${DB_NAME}"
```

#### ARG - Build Arguments
```dockerfile
# Simple ARG
ARG VERSION=latest
ARG BUILD_DATE
ARG VCS_REF

# Using ARG
FROM node:${VERSION}
LABEL build-date=${BUILD_DATE}
LABEL vcs-ref=${VCS_REF}

# ARG scope
ARG GLOBAL_ARG=value
FROM ubuntu
ARG GLOBAL_ARG          # Re-declare to use in this stage
ARG STAGE_ARG=value     # Only available in this stage

# Predefined ARGs (automatically available)
ARG TARGETPLATFORM      # linux/amd64, linux/arm64, etc.
ARG TARGETOS           # linux, windows
ARG TARGETARCH         # amd64, arm64, etc.
```

#### EXPOSE - Port Documentation
```dockerfile
EXPOSE 80
EXPOSE 443
EXPOSE 8080/tcp
EXPOSE 8081/udp
EXPOSE 3000-3005       # Port range

# Multiple ports
EXPOSE 80 443 8080
```

#### VOLUME - Mount Points
```dockerfile
VOLUME ["/data"]
VOLUME ["/var/log", "/var/db"]
VOLUME /data /logs      # Multiple volumes

# Anonymous volumes
VOLUME /app/uploads
```

#### USER - Security
```dockerfile
# Switch to existing user
USER nginx
USER 1001
USER node:node

# Create and switch to user
RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser

# Alpine Linux
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup
USER appuser
```

#### CMD vs ENTRYPOINT
```dockerfile
# CMD - Default command (can be overridden)
CMD ["npm", "start"]
CMD npm start                    # Shell form
CMD ["node", "server.js"]

# ENTRYPOINT - Fixed command (cannot be overridden)
ENTRYPOINT ["docker-entrypoint.sh"]
ENTRYPOINT ["npm", "start"]

# Combined usage
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["npm", "start"]             # Default parameter

# Variable ENTRYPOINT
ENTRYPOINT ["sh", "-c", "exec $0 $@"]
CMD ["npm", "start"]

# Script ENTRYPOINT
COPY docker-entrypoint.sh /usr/local/bin/
ENTRYPOINT ["docker-entrypoint.sh"]
```

#### HEALTHCHECK - Container Health
```dockerfile
# Basic health check
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# Detailed configuration
HEALTHCHECK --interval=5m --timeout=3s --start-period=5m --retries=3 \
  CMD curl -f http://localhost/ || exit 1

# Disable health check
HEALTHCHECK NONE

# Custom health check script
COPY healthcheck.sh /usr/local/bin/
HEALTHCHECK --interval=30s CMD /usr/local/bin/healthcheck.sh
```

#### ONBUILD - Trigger Instructions
```dockerfile
# Base image with ONBUILD
ONBUILD COPY package*.json ./
ONBUILD RUN npm install
ONBUILD COPY . .

# Use case: Creating base images for teams
FROM node:alpine
ONBUILD WORKDIR /app
ONBUILD COPY package*.json ./
ONBUILD RUN npm ci --only=production
ONBUILD COPY . .
ONBUILD EXPOSE 3000
ONBUILD CMD ["npm", "start"]
```

#### SHELL - Default Shell
```dockerfile
# Change default shell (Linux)
SHELL ["/bin/bash", "-c"]

# Windows PowerShell
SHELL ["powershell", "-command"]

# Fish shell
SHELL ["/usr/bin/fish", "-c"]
```

#### STOPSIGNAL - Stop Signal
```dockerfile
STOPSIGNAL SIGTERM  # Default
STOPSIGNAL SIGKILL
STOPSIGNAL 9        # Signal number
```

### Advanced Dockerfile Techniques

#### Multi-stage Builds
```dockerfile
# Build stage
FROM node:18-alpine AS dependencies
WORKDIR /app
COPY package*.json ./
RUN npm ci

# Build application
FROM dependencies AS build
COPY . .
RUN npm run build

# Test stage
FROM build AS test
RUN npm test

# Production stage
FROM node:18-alpine AS production
WORKDIR /app
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001
COPY --from=dependencies --chown=nextjs:nodejs /app/node_modules ./node_modules
COPY --from=build --chown=nextjs:nodejs /app/dist ./dist
COPY --chown=nextjs:nodejs package*.json ./
USER nextjs
EXPOSE 3000
CMD ["npm", "start"]
```

#### BuildKit Features
```dockerfile
# syntax=docker/dockerfile:1
FROM alpine

# Here documents
RUN <<EOF
apk add --no-cache curl
curl -o /tmp/file https://example.com/file
chmod +x /tmp/file
EOF

# Heredoc for scripts
COPY <<EOF /app/script.sh
#!/bin/bash
echo "Hello World"
exit 0
EOF

# Multi-line variables
ENV MY_VAR=<<EOF
line1
line2
line3
EOF
```

#### Cache Optimization
```dockerfile
# Bad - Cache invalidated on any file change
FROM node:alpine
COPY . .
RUN npm install

# Good - Separate package.json copy
FROM node:alpine
COPY package*.json ./
RUN npm install  # This layer is cached unless package.json changes
COPY . .
```

---

## Docker Compose Complete Reference

### Complete docker-compose.yml Structure
```yaml
version: '3.8'  # Compose file version

x-common-variables: &common-variables  # YAML anchors
  POSTGRES_USER: ${POSTGRES_USER:-postgres}
  POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-password}

x-logging: &default-logging             # Reusable logging config
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"

services:
  web:
    # Service configuration
  
  db:
    # Database configuration

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true

volumes:
  postgres_data:
    driver: local
  redis_data:
    external: true

configs:
  nginx_config:
    file: ./nginx.conf

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

### Service Configuration Deep Dive

#### Build Configuration
```yaml
services:
  web:
    # Simple build
    build: .
    
    # Advanced build
    build:
      context: .                    # Build context
      dockerfile: Dockerfile.prod   # Custom Dockerfile
      args:                        # Build arguments
        - NODE_ENV=production
        - BUILD_DATE=${BUILD_DATE}
      target: production           # Multi-stage build target
      cache_from:                  # Cache sources
        - myapp:latest
      labels:                      # Build labels
        - "version=1.0"
      shm_size: '2gb'             # Shared memory size
      platforms:                   # Target platforms
        - linux/amd64
        - linux/arm64
```

#### Image and Container Settings
```yaml
services:
  app:
    image: myapp:latest
    container_name: my-app-container
    hostname: app-server
    domainname: example.com
    
    # Restart policies
    restart: "no"          # Default
    restart: always        # Always restart
    restart: on-failure    # Restart on failure
    restart: unless-stopped # Restart unless manually stopped
    
    # Process and user settings
    user: "1001:1001"      # UID:GID
    working_dir: /app
    
    # Resource constraints (v3.8+)
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
          pids: 100
        reservations:
          cpus: '0.25'
          memory: 256M
      
      # Restart policy for swarm mode
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
```

#### Networking
```yaml
services:
  web:
    # Port mapping
    ports:
      - "3000:3000"              # host:container
      - "127.0.0.1:8080:80"      # bind to specific interface
      - "3000-3005:3000-3005"    # port range
      - "6060:6060/udp"          # UDP port
    
    # Expose ports to other services only
    expose:
      - "3000"
      - "8000"
    
    # Network configuration
    networks:
      - frontend
      - backend
    
    # Advanced network settings
    networks:
      frontend:
        aliases:
          - web-server
        ipv4_address: 172.16.238.10
      backend: {}
    
    # Network mode
    network_mode: "bridge"     # Default
    network_mode: "host"       # Use host networking
    network_mode: "none"       # No networking
    network_mode: "service:db" # Use another service's network
    network_mode: "container:my-container" # Use another container's network
    
    # External links (legacy)
    external_links:
      - redis_1
      - project_db_1:mysql
```

#### Environment and Configuration
```yaml
services:
  app:
    # Environment variables
    environment:
      - NODE_ENV=production
      - DEBUG=app:*
      - DATABASE_URL=postgresql://user:pass@db:5432/myapp
    
    # Environment from file
    env_file:
      - .env
      - .env.local
      - path/to/custom.env
    
    # Command and entrypoint
    command: npm start
    command: ["npm", "start"]
    command: >
      sh -c "
        npm run migrate &&
        npm start
      "
    
    entrypoint: /app/entrypoint.sh
    entrypoint: ["python", "app.py"]
    
    # Working directory
    working_dir: /app
    
    # User configuration
    user: nginx
    user: "1001"
    user: "1001:1001"
```

#### Volumes and Storage
```yaml
services:
  app:
    volumes:
      # Named volume
      - postgres_data:/var/lib/postgresql/data
      
      # Bind mount
      - ./src:/app/src
      - ./config:/app/config:ro  # Read-only
      
      # Anonymous volume
      - /app/node_modules
      
      # Tmpfs mount
      - type: tmpfs
        target: /app/cache
        tmpfs:
          size: 100M
      
      # Advanced volume configuration
      - type: volume
        source: postgres_data
        target: /var/lib/postgresql/data
        volume:
          nocopy: true
      
      # Bind mount with advanced options
      - type: bind
        source: ./src
        target: /app/src
        bind:
          propagation: cached
      
    # Volumes from another container
    volumes_from:
      - data-container
      - another-service:ro
```

#### Dependencies and Health
```yaml
services:
  web:
    # Simple dependencies
    depends_on:
      - db
      - redis
    
    # Dependencies with conditions
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    
    # Health check
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
      disable: false
    
    # Links (legacy)
    links:
      - db
      - redis:cache
```

#### Logging
```yaml
services:
  web:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
        labels: "production_status"
        env: "os,customer"
    
    # Other logging drivers
    logging:
      driver: syslog
      options:
        syslog-address: "tcp://192.168.1.42:123"
    
    logging:
      driver: none  # Disable logging
```

#### Advanced Service Settings
```yaml
services:
  app:
    # Process settings
    pid: "host"           # Use host PID namespace
    ipc: "host"           # Use host IPC namespace
    uts: "host"           # Use host UTS namespace
    
    # Capabilities
    cap_add:
      - NET_ADMIN
      - SYS_TIME
    cap_drop:
      - MKNOD
    
    # Security
    privileged: true      # Run in privileged mode
    read_only: true       # Read-only root filesystem
    
    # Devices
    devices:
      - "/dev/ttyUSB0:/dev/ttyUSB0"
    
    # DNS
    dns:
      - 8.8.8.8
      - 9.9.9.9
    dns_search:
      - example.com
    
    # Extra hosts
    extra_hosts:
      - "somehost:162.242.195.82"
      - "otherhost:50.31.209.229"
    
    # System controls
    sysctls:
      - net.core.somaxconn=1024
      - kernel.sem=60 32000 32 128
    
    # Ulimits
    ulimits:
      nproc: 65535
      nofile:
        soft: 20000
        hard: 40000
    
    # Labels
    labels:
      - "com.example.description=Accounting webapp"
      - "com.example.department=Finance"
      - "com.example.label-with-empty-value"
    
    # Platform
    platform: linux/amd64
```

### Networks Configuration
```yaml
networks:
  # Basic network
  frontend:
  
  # Bridge network with configuration
  backend:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.name: br-backend
    ipam:
      config:
        - subnet: 172.28.0.0/16
          ip_range: 172.28.5.0/24
          gateway: 172.28.5.254
  
  # Overlay network (Swarm mode)
  overlay-net:
    driver: overlay
    attachable: true
    
  # External network
  existing-network:
    external: true
    name: my-existing-network
  
  # Host network
  host-network:
    external: true
    name: host
```

### Volumes Configuration
```yaml
volumes:
  # Basic named volume
  postgres_data:
  
  # Local driver with options
  app_data:
    driver: local
    driver_opts:
      type: 'none'
      o: 'bind'
      device: '/path/to/data'
  
  # External volume
  external_volume:
    external: true
    name: my-existing-volume
  
  # NFS volume
  nfs_volume:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.1,rw
      device: ":/path/to/nfs/share"
```

### Configs and Secrets
```yaml
configs:
  nginx_config:
    file: ./nginx.conf
  app_config:
    external: true
    name: my-app-config

secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    external: true
    name: my-api-key

services:
  web:
    configs:
      - source: nginx_config
        target: /etc/nginx/nginx.conf
        uid: '103'
        gid: '103'
        mode: 0440
    
    secrets:
      - source: db_password
        target: /run/secrets/db_password
        uid: '103'
        gid: '103'
        mode: 0400
```

---

## Docker CLI Commands

### Container Management
```bash
# Run containers
docker run -d nginx                          # Detached mode
docker run -it ubuntu bash                   # Interactive with TTY
docker run --rm alpine echo "hello"          # Remove after exit
docker run -p 8080:80 nginx                 # Port mapping
docker run -v /host:/container nginx        # Volume mount
docker run --name myapp nginx               # Named container
docker run -e NODE_ENV=production node      # Environment variable
docker run --restart unless-stopped nginx   # Restart policy
docker run --memory=512m nginx              # Memory limit
docker run --cpus="1.5" nginx               # CPU limit

# Container lifecycle
docker create nginx                          # Create without starting
docker start container_name                 # Start stopped container
docker stop container_name                  # Graceful stop
docker restart container_name               # Restart container
docker pause container_name                 # Pause container
docker unpause container_name               # Unpause container
docker kill container_name                  # Force kill
docker rm container_name                    # Remove container
docker rm -f container_name                 # Force remove running container

# List containers
docker ps                                   # Running containers
docker ps -a                               # All containers
docker ps -q                               # Container IDs only
docker ps --filter "status=exited"         # Filter by status
docker ps --format "table {{.Names}}\t{{.Status}}" # Custom format

# Container inspection
docker inspect container_name               # Detailed information
docker logs container_name                  # View logs
docker logs -f container_name              # Follow logs
docker logs --tail 100 container_name     # Last 100 lines
docker stats                               # Resource usage
docker stats container_name                # Specific container stats
docker top container_name                  # Running processes
docker port container_name                 # Port mappings

# Execute commands in containers
docker exec -it container_name bash        # Interactive shell
docker exec container_name ls -la          # Single command
docker exec -u root container_name bash    # As specific user
docker exec -w /app container_name pwd     # In specific directory

# Copy files
docker cp file.txt container_name:/app/    # Host to container
docker cp container_name:/app/file.txt .   # Container to host

# Container export/import
docker export container_name > backup.tar  # Export container
docker import backup.tar new_image:tag     # Import as image
```

### Image Management
```bash
# Build images
docker build .                             # Build from current directory
docker build -t myapp:latest .            # With tag
docker build -f Dockerfile.prod .         # Custom Dockerfile
docker build --build-arg NODE_ENV=prod .  # Build arguments
docker build --target production .        # Multi-stage target
docker build --no-cache .                 # No cache
docker build --platform linux/amd64 .    # Specific platform

# List and inspect images
docker images                              # List images
docker images -q                           # Image IDs only
docker images --filter "dangling=true"    # Dangling images
docker image inspect image_name           # Detailed information
docker history image_name                 # Layer history

# Tag and push images
docker tag image_name new_name:tag         # Tag image
docker push image_name:tag                # Push to registry
docker pull image_name:tag               # Pull from registry

# Remove images
docker rmi image_name                      # Remove image
docker rmi -f image_name                  # Force remove
docker image prune                        # Remove dangling images
docker image prune -a                     # Remove unused images

# Save and load images
docker save image_name > backup.tar       # Save to file
docker load < backup.tar                  # Load from file
docker save image_name | gzip > backup.tar.gz # Compressed save
```

### System Management
```bash
# System information
docker version                            # Docker version
docker info                              # System information
docker system df                         # Disk usage
docker system events                     # Real-time events

# System cleanup
docker system prune                      # Remove unused data
docker system prune -a                  # Remove all unused data
docker system prune --volumes           # Include volumes
docker container prune                  # Remove stopped containers
docker image prune                      # Remove dangling images
docker volume prune                     # Remove unused volumes
docker network prune                    # Remove unused networks

# Resource monitoring
docker stats                            # Live resource usage
docker events                           # Real-time events
docker events --filter container=myapp  # Filter events
```

### Registry Operations
```bash
# Login/logout
docker login                            # Login to Docker Hub
docker login registry.example.com      # Login to private registry
docker logout                          # Logout

# Search images
docker search nginx                     # Search Docker Hub
docker search --limit 10 nginx        # Limit results

# Repository management
docker tag local_image registry.com/repo:tag
docker push registry.com/repo:tag
docker pull registry.com/repo:tag
```

### Docker Context
```bash
# Manage Docker contexts
docker context ls                       # List contexts
docker context create mycontext \      # Create context
  --docker "host=tcp://myhost:2376"
docker context use mycontext           # Switch context
docker context inspect mycontext       # Inspect context
docker context rm mycontext           # Remove context
```

---

## Docker Networks

### Network Types
```bash
# Bridge (default)
docker network create my-bridge-network

# Host networking
docker run --network host nginx

# None (no networking)
docker run --network none alpine

# Container networking
docker run --network container:other-container alpine

# Custom bridge with configuration
docker network create \
  --driver bridge \
  --subnet=172.20.0.0/16 \
  --ip-range=172.20.240.0/20 \
  --gateway=172.20.0.1 \
  my-custom-network
```

### Network Management
```bash
# Create networks
docker network create mynetwork                    # Basic bridge
docker network create -d bridge mynetwork         # Explicit driver
docker network create -d overlay mynetwork        # Overlay network
docker network create --internal mynetwork        # Internal only

# Advanced network creation
docker network create \
  --driver=bridge \
  --subnet=192.168.1.0/24 \
  --ip-range=192.168.1.128/25 \
  --gateway=192.168.1.1 \
  --aux-address="host1=192.168.1.5" \
  --aux-address="host2=192.168.1.6" \
  my-network

# List and inspect
docker network ls                                  # List networks
docker network inspect mynetwork                  # Network details
docker network inspect mynetwork --format='{{.IPAM.Config}}'

# Connect/disconnect containers
docker network connect mynetwork container_name   # Connect container
docker network disconnect mynetwork container_name # Disconnect

# Remove networks
docker network rm mynetwork                       # Remove network
docker network prune                             # Remove unused networks
```

### Network Communication
```bash
# Container communication by name
docker run -d --name web --network mynetwork nginx
docker run -it --network mynetwork alpine ping web

# Port publishing
docker run -p 8080:80 nginx                      # Host port 8080 → container port 80
docker run -p 127.0.0.1:8080:80 nginx          # Bind to specific interface
docker run -p 8080:80/udp nginx                 # UDP port
docker run -P nginx                             # Publish all exposed ports

# Network aliases
docker run --network mynetwork --network-alias web nginx
docker run --network mynetwork --network-alias api nginx
```

---

## Docker Volumes

### Volume Types
```bash
# Anonymous volumes
docker run -v /data alpine

# Named volumes
docker run -v myvolume:/data alpine

# Bind mounts
docker run -v /host/path:/container/path alpine
docker run -v $(pwd):/app alpine

# Tmpfs mounts
docker run --tmpfs /app/cache alpine
```

### Volume Management
```bash
# Create volumes
docker volume create myvolume                    # Basic volume
docker volume create --driver local myvolume    # With driver
docker volume create \                          # With options
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.1,rw \
  --opt device=:/path/to/dir \
  nfs-volume

# List and inspect
docker volume ls                                 # List volumes
docker volume inspect myvolume                  # Volume details
docker volume ls --filter "dangling=true"      # Dangling volumes

# Remove volumes
docker volume rm myvolume                       # Remove volume
docker volume prune                            # Remove unused volumes
```

### Advanced Volume Options
```bash
# Volume with specific mount options
docker run \
  --mount type=volume,source=myvolume,target=/data,readonly \
  alpine

# Bind mount with options
docker run \
  --mount type=bind,source=$(pwd),target=/app,bind-propagation=cached \
  alpine

# Tmpfs with size limit
docker run \
  --mount type=tmpfs,target=/app/cache,tmpfs-size=100m \
  alpine

# Volume backup and restore
docker run --rm \
  -v myvolume:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/backup.tar.gz /data

docker run --rm \
  -v myvolume:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/backup.tar.gz -C /
```

---

## Docker Security

### Container Security Best Practices

#### User Management
```dockerfile
# Create non-root user
FROM alpine
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup
USER appuser

# Drop capabilities
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx

# Run as specific user
docker run --user 1001:1001 alpine
```

#### Security Options
```bash
# AppArmor
docker run --security-opt apparmor:profile_name alpine

# SELinux
docker run --security-opt label:type:container_runtime_t alpine

# No new privileges
docker run --security-opt no-new-privileges alpine

# Seccomp profile
docker run --security-opt seccomp:profile.json alpine

# Read-only root filesystem
docker run --read-only alpine
```

#### Resource Limits
```bash
# Memory limits
docker run --memory=512m alpine
docker run --memory=1g --memory-swap=2g alpine

# CPU limits
docker run --cpus="1.5" alpine
docker run --cpu-shares=512 alpine

# Process limits
docker run --pids-limit=100 alpine

# Ulimits
docker run --ulimit nofile=1024:1024 alpine
```

### Image Security

#### Vulnerability Scanning
```bash
# Docker Scout (built-in)
docker scout cves myimage:latest
docker scout recommendations myimage:latest

# Trivy
trivy image myimage:latest

# Snyk
snyk container test myimage:latest
```

#### Secure Image Building
```dockerfile
# Use official images
FROM node:18-alpine

# Use specific tags (not latest)
FROM nginx:1.21-alpine

# Multi-stage builds to reduce attack surface
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine AS runtime
WORKDIR /app
COPY --from=build /app/node_modules ./node_modules
COPY . .
USER node
CMD ["npm", "start"]

# Sign images
docker trust sign myimage:latest
```

#### Runtime Security
```bash
# Run with restricted capabilities
docker run --cap-drop=ALL --cap-add=CHOWN --cap-add=DAC_OVERRIDE alpine

# Use security profiles
docker run --security-opt apparmor:docker-default alpine
docker run --security-opt seccomp:default.json alpine

# Network security
docker run --network none alpine  # No network access
docker run --network mynetwork alpine  # Isolated network
```

---

## Performance & Optimization

### Image Optimization

#### Layer Optimization
```dockerfile
# Bad - Multiple layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# Good - Single layer
RUN apt-get update && \
    apt-get install -y curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Use .dockerignore
node_modules
.git
.env
*.log
.DS_Store
coverage/
.nyc_output
```

#### Multi-stage Builds
```dockerfile
# Efficient multi-stage build
FROM node:18-alpine AS dependencies
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build && npm prune --production

FROM node:18-alpine AS runtime
RUN addgroup -g 1001 -S nodejs && adduser -S nextjs -u 1001
WORKDIR /app
COPY --from=dependencies --chown=nextjs:nodejs /app/node_modules ./node_modules
COPY --from=build --chown=nextjs:nodejs /app/dist ./dist
COPY --chown=nextjs:nodejs package*.json ./
USER nextjs
EXPOSE 3000
CMD ["npm", "start"]
```

### Container Performance

#### Resource Management
```yaml
# docker-compose.yml
services:
  app:
    image: myapp
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          cpus: '1.0'
          memory: 1G
    
    # Logging optimization
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    
    # Health checks
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

#### Storage Optimization
```bash
# Use efficient storage drivers
docker info | grep "Storage Driver"

# Volume optimization
docker volume create --driver local \
  --opt type=tmpfs \
  --opt device=tmpfs \
  --opt o=size=100m \
  cache-volume

# Bind mount optimization
docker run -v /host/path:/container/path:cached myapp
```

### Build Performance

#### BuildKit Optimization
```bash
# Enable BuildKit
export DOCKER_BUILDKIT=1

# Use BuildKit cache
docker build --cache-from myapp:latest .

# Multi-platform builds
docker buildx create --use
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest .
```

#### Cache Strategies
```dockerfile
# Optimize package installation
COPY package*.json ./
RUN npm ci --only=production  # Cached unless package.json changes
COPY . .

# Use build cache mount
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production
```

---

## Debugging & Troubleshooting

### Container Debugging

#### Inspection Commands
```bash
# Container details
docker inspect container_name
docker inspect container_name | jq '.[0].State'
docker inspect container_name | jq '.[0].NetworkSettings'

# Process information
docker top container_name
docker exec container_name ps aux
docker exec container_name netstat -tulpn

# Resource usage
docker stats container_name
docker exec container_name df -h
docker exec container_name free -m

# Network debugging
docker exec container_name ping google.com
docker exec container_name nslookup google.com
docker exec container_name curl -I http://example.com
```

#### Log Analysis
```bash
# View logs
docker logs container_name
docker logs -f container_name                    # Follow logs
docker logs --tail 100 container_name          # Last 100 lines
docker logs --since "2023-01-01T00:00:00" container_name
docker logs --until "2023-01-01T12:00:00" container_name

# Structured log analysis
docker logs container_name | grep ERROR
docker logs container_name | jq '.'  # If JSON logs
```

#### Interactive Debugging
```bash
# Access container shell
docker exec -it container_name bash
docker exec -it container_name sh
docker exec -it --user root container_name bash

# Debug with specific tools
docker exec -it container_name strace -p 1
docker exec -it container_name tcpdump -i eth0
docker exec -it container_name lsof -p 1
```

### Build Debugging

#### Build Issues
```bash
# Verbose build output
docker build --progress=plain .

# Debug specific build stage
docker build --target build-stage .

# Build with no cache
docker build --no-cache .

# Inspect build history
docker history image_name

# Run intermediate container
docker run -it intermediate_image_id bash
```

#### Common Issues and Solutions

```bash
# Permission issues
# Problem: Permission denied
# Solution: Check user permissions
docker exec -it container ls -la /app
docker exec -it --user root container chown -R appuser:appuser /app

# Network connectivity issues
# Problem: Cannot connect to service
# Solution: Check network configuration
docker network ls
docker network inspect network_name
docker exec container_name ping other_container

# Storage issues
# Problem: No space left on device
# Solution: Clean up Docker resources
docker system df
docker system prune -a
docker volume prune

# Memory issues
# Problem: Container killed (OOMKilled)
# Solution: Increase memory limit or optimize application
docker run --memory=2g myapp
docker stats container_name
```

### Performance Debugging

#### Performance Analysis
```bash
# CPU profiling
docker stats --no-stream
docker exec container_name top
docker exec container_name htop

# Memory analysis
docker exec container_name free -m
docker exec container_name ps aux --sort=-%mem

# I/O monitoring
docker exec container_name iostat -x 1
docker exec container_name iotop
```

#### Network Performance
```bash
# Network latency
docker exec container_name ping -c 10 target_host

# Bandwidth testing
docker exec container_name iperf3 -c target_host

# Connection tracking
docker exec container_name ss -tulpn
docker exec container_name netstat -i
```

---

## Production Best Practices

### Security Hardening

#### Container Security
```dockerfile
# Use minimal base images
FROM scratch
FROM alpine:latest
FROM distroless/java:11

# Run as non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup
USER appuser

# Set read-only filesystem
docker run --read-only --tmpfs /tmp myapp

# Drop capabilities
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp
```

#### Secret Management
```yaml
# docker-compose.yml
services:
  app:
    image: myapp
    secrets:
      - db_password
      - api_key
    environment:
      - DB_PASSWORD_FILE=/run/secrets/db_password
      - API_KEY_FILE=/run/secrets/api_key

secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    external: true
    name: production_api_key
```

### Monitoring and Logging

#### Centralized Logging
```yaml
# ELK Stack example
version: '3.8'

services:
  app:
    image: myapp
    logging:
      driver: gelf
      options:
        gelf-address: "udp://logstash:12201"
        tag: "myapp"

  elasticsearch:
    image: elasticsearch:7.17.0
    environment:
      - discovery.type=single-node
    volumes:
      - es_data:/usr/share/elasticsearch/data

  logstash:
    image: logstash:7.17.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf

  kibana:
    image: kibana:7.17.0
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

volumes:
  es_data:
```

#### Health Monitoring
```yaml
services:
  app:
    image: myapp
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    
    # Restart policy
    restart: unless-stopped
    
    # Resource limits
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: '1.0'
```

### Deployment Strategies

#### Blue-Green Deployment
```bash
# Blue-green deployment script
#!/bin/bash

# Build new version
docker build -t myapp:v2 .

# Start new version (green)
docker-compose -f docker-compose.yml -f docker-compose.green.yml up -d

# Health check
curl -f http://localhost:8081/health

# Switch traffic (update load balancer)
# ... update nginx config or load balancer ...

# Stop old version (blue)
docker-compose -f docker-compose.yml -f docker-compose.blue.yml down
```

#### Rolling Updates
```yaml
# docker-compose.yml for rolling updates
version: '3.8'

services:
  app:
    image: myapp:${VERSION:-latest}
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
        order: start-first
      rollback_config:
        parallelism: 1
        delay: 10s
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

### Backup and Disaster Recovery

#### Data Backup
```bash
# Volume backup
docker run --rm \
  -v postgres_data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/postgres_backup_$(date +%Y%m%d_%H%M%S).tar.gz /data

# Database backup
docker exec postgres_container pg_dump -U user database > backup.sql
docker exec mysql_container mysqldump -u user -p database > backup.sql

# Automated backup script
#!/bin/bash
BACKUP_DIR="/backups"
DATE=$(date +%Y%m%d_%H%M%S)

# Create backup
docker-compose exec -T db pg_dump -U user database > $BACKUP_DIR/db_$DATE.sql

# Compress backup
gzip $BACKUP_DIR/db_$DATE.sql

# Keep only last 7 days
find $BACKUP_DIR -name "db_*.sql.gz" -mtime +7 -delete
```

#### Configuration Backup
```bash
# Backup Docker Compose files
tar czf docker_config_backup.tar.gz \
  docker-compose.yml \
  docker-compose.override.yml \
  .env \
  secrets/ \
  configs/

# Backup volumes list
docker volume ls > volumes_list.txt
```

---

## Advanced Scenarios

### Multi-Architecture Builds

#### Building for Multiple Platforms
```bash
# Create builder
docker buildx create --name mybuilder --use

# Build for multiple architectures
docker buildx build \
  --platform linux/amd64,linux/arm64,linux/arm/v7 \
  -t myapp:latest \
  --push .

# Inspect multi-arch image
docker buildx imagetools inspect myapp:latest
```

#### Architecture-Specific Dockerfiles
```dockerfile
# Dockerfile with architecture conditionals
FROM --platform=$TARGETPLATFORM alpine:latest

ARG TARGETPLATFORM
ARG TARGETARCH

RUN if [ "$TARGETARCH" = "arm64" ]; then \
      echo "Building for ARM64"; \
    elif [ "$TARGETARCH" = "amd64" ]; then \
      echo "Building for AMD64"; \
    fi

# Copy architecture-specific binaries
COPY --from=builder /app/bin/${TARGETARCH}/myapp /usr/local/bin/
```

### Docker Swarm

#### Swarm Initialization
```bash
# Initialize swarm
docker swarm init --advertise-addr 192.168.1.100

# Join as worker
docker swarm join --token TOKEN 192.168.1.100:2377

# Join as manager
docker swarm join-token manager
```

#### Stack Deployment
```yaml
# docker-stack.yml
version: '3.8'

services:
  web:
    image: myapp:latest
    ports:
      - "80:80"
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    networks:
      - webnet

  db:
    image: postgres:13
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    volumes:
      - db_data:/var/lib/postgresql/data
    deploy:
      placement:
        constraints: [node.role == manager]
    secrets:
      - db_password
    networks:
      - webnet

networks:
  webnet:
    driver: overlay

volumes:
  db_data:
    driver: local

secrets:
  db_password:
    external: true
```

```bash
# Deploy stack
docker stack deploy -c docker-stack.yml myapp

# Scale service
docker service scale myapp_web=5

# Update service
docker service update --image myapp:v2 myapp_web
```

### Custom Networks

#### Advanced Network Configuration
```bash
# Create custom bridge network
docker network create \
  --driver bridge \
  --subnet 192.168.100.0/24 \
  --gateway 192.168.100.1 \
  --ip-range 192.168.100.128/25 \
  custom-bridge

# Create MACVLAN network
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  macvlan-net

# Create overlay network
docker network create \
  --driver overlay \
  --subnet 10.0.0.0/24 \
  --attachable \
  overlay-net
```

### Service Discovery

#### Using DNS
```yaml
services:
  web:
    image: nginx
    networks:
      - app-net
  
  api:
    image: myapi
    networks:
      - app-net
  
  worker:
    image: myworker
    environment:
      - API_URL=http://api:3000  # Service discovery by name
    networks:
      - app-net

networks:
  app-net:
    driver: bridge
```

#### External Service Discovery
```yaml
services:
  app:
    image: myapp
    environment:
      - CONSUL_HOST=consul.service.consul
      - ETCD_ENDPOINTS=http://etcd1:2379,http://etcd2:2379
    extra_hosts:
      - "consul.service.consul:192.168.1.10"
      - "etcd1:192.168.1.11"
      - "etcd2:192.168.1.12"
```

---

## Complete Examples

### Full-Stack Application (MEAN Stack)

#### Project Structure
```
project/
├── docker-compose.yml
├── docker-compose.override.yml
├── docker-compose.prod.yml
├── .env
├── frontend/
│   ├── Dockerfile
│   ├── nginx.conf
│   └── src/
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
└── nginx/
    ├── Dockerfile
    └── nginx.conf
```

#### Frontend Dockerfile (Angular)
```dockerfile
# frontend/Dockerfile
FROM node:18-alpine AS build

WORKDIR /app
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM nginx:alpine AS runtime
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

#### Backend Dockerfile (Node.js)
```dockerfile
# backend/Dockerfile
FROM node:18-alpine AS dependencies

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:18-alpine AS runtime
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodeuser -u 1001 -G nodejs

WORKDIR /app
COPY --from=dependencies --chown=nodeuser:nodejs /app/node_modules ./node_modules
COPY --from=build --chown=nodeuser:nodejs /app/dist ./dist
COPY --chown=nodeuser:nodejs package*.json ./

USER nodeuser
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

CMD ["npm", "start"]
```

#### Complete Docker Compose
```yaml
# docker-compose.yml
version: '3.8'

x-common-variables: &common-variables
  MONGO_INITDB_ROOT_USERNAME: ${MONGO_ROOT_USER:-admin}
  MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD:-password}

x-logging: &default-logging
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"

services:
  frontend:
    build:
      context: ./frontend
      target: runtime
    ports:
      - "${FRONTEND_PORT:-80}:80"
    depends_on:
      - backend
    networks:
      - frontend-net
    restart: unless-stopped
    logging: *default-logging

  backend:
    build:
      context: ./backend
      target: runtime
    ports:
      - "${BACKEND_PORT:-3000}:3000"
    environment:
      - NODE_ENV=${NODE_ENV:-production}
      - MONGO_URL=mongodb://${MONGO_ROOT_USER:-admin}:${MONGO_ROOT_PASSWORD:-password}@mongodb:27017/myapp?authSource=admin
      - REDIS_URL=redis://redis:6379
    depends_on:
      mongodb:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - frontend-net
      - backend-net
    restart: unless-stopped
    logging: *default-logging
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  mongodb:
    image: mongo:6.0
    environment: *common-variables
    volumes:
      - mongo_data:/data/db
      - ./mongodb/init:/docker-entrypoint-initdb.d:ro
    networks:
      - backend-net
    restart: unless-stopped
    logging: *default-logging
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD:-password}
    volumes:
      - redis_data:/data
    networks:
      - backend-net
    restart: unless-stopped
    logging: *default-logging
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3

  nginx:
    image: nginx:alpine
    ports:
      - "${NGINX_PORT:-8080}:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - frontend
      - backend
    networks:
      - frontend-net
    restart: unless-stopped
    logging: *default-logging

networks:
  frontend-net:
    driver: bridge
  backend-net:
    driver: bridge
    internal: true

volumes:
  mongo_data:
    driver: local
  redis_data:
    driver: local
```

#### Development Override
```yaml
# docker-compose.override.yml
version: '3.8'

services:
  frontend:
    build:
      target: build
    volumes:
      - ./frontend/src:/app/src
    environment:
      - NODE_ENV=development
    command: npm run dev

  backend:
    build:
      target: build
    volumes:
      - ./backend/src:/app/src
      - ./backend/package*.json:/app/
    environment:
      - NODE_ENV=development
      - DEBUG=app:*
    command: npm run dev

  mongodb:
    ports:
      - "27017:27017"

  redis:
    ports:
      - "6379:6379"
```

#### Production Configuration
```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  frontend:
    deploy:
      replicas: 2
      resources:
        limits:
          memory: 256M
          cpus: '0.5'
        reservations:
          memory: 128M
          cpus: '0.25'

  backend:
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 512M
          cpus: '1.0'
        reservations:
          memory: 256M
          cpus: '0.5'

  mongodb:
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: '2.0'
        reservations:
          memory: 512M
          cpus: '1.0'

  redis:
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: '0.5'
        reservations:
          memory: 128M
          cpus: '0.25'
```

### Microservices Architecture

#### API Gateway Pattern
```yaml
# docker-compose.microservices.yml
version: '3.8'

services:
  api-gateway:
    image: kong:latest
    ports:
      - "8000:8000"
      - "8443:8443"
      - "8001:8001"
      - "8444:8444"
    environment:
      - KONG_DATABASE=postgres
      - KONG_PG_HOST=kong-database
      - KONG_PG_USER=kong
      - KONG_PG_PASSWORD=kong
      - KONG_PROXY_ACCESS_LOG=/dev/stdout
      - KONG_ADMIN_ACCESS_LOG=/dev/stdout
      - KONG_PROXY_ERROR_LOG=/dev/stderr
      - KONG_ADMIN_ERROR_LOG=/dev/stderr
      - KONG_ADMIN_LISTEN=0.0.0.0:8001
    depends_on:
      - kong-database
    networks:
      - kong-net
      - services-net

  kong-database:
    image: postgres:13
    environment:
      - POSTGRES_USER=kong
      - POSTGRES_PASSWORD=kong
      - POSTGRES_DB=kong
    volumes:
      - kong_data:/var/lib/postgresql/data
    networks:
      - kong-net

  user-service:
    build: ./services/user-service
    environment:
      - DATABASE_URL=postgresql://user:password@user-db:5432/users
    depends_on:
      - user-db
    networks:
      - services-net
      - user-net

  user-db:
    image: postgres:13
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=users
    volumes:
      - user_data:/var/lib/postgresql/data
    networks:
      - user-net

  order-service:
    build: ./services/order-service
    environment:
      - DATABASE_URL=postgresql://order:password@order-db:5432/orders
      - USER_SERVICE_URL=http://user-service:3000
    depends_on:
      - order-db
    networks:
      - services-net
      - order-net

  order-db:
    image: postgres:13
    environment:
      - POSTGRES_USER=order
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=orders
    volumes:
      - order_data:/var/lib/postgresql/data
    networks:
      - order-net

  notification-service:
    build: ./services/notification-service
    environment:
      - REDIS_URL=redis://redis:6379
      - RABBITMQ_URL=amqp://rabbitmq:5672
    depends_on:
      - redis
      - rabbitmq
    networks:
      - services-net
      - notification-net

  redis:
    image: redis:7-alpine
    networks:
      - notification-net

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "15672:15672"
    environment:
      - RABBITMQ_DEFAULT_USER=admin
      - RABBITMQ_DEFAULT_PASS=password
    networks:
      - notification-net

networks:
  kong-net:
  services-net:
  user-net:
  order-net:
  notification-net:

volumes:
  kong_data:
  user_data:
  order_data:
```

### CI/CD Pipeline Integration

#### GitHub Actions Workflow
```yaml
# .github/workflows/docker.yml
name: Docker Build and Deploy

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
    - name: Checkout repository
      uses: actions/checkout@v3

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2

    - name: Log in to Container Registry
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
          type=sha

    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        file: ./Dockerfile
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

    - name: Run security scan
      uses: docker/scout-action@v0.18.1
      with:
        command: cves
        image: ${{ steps.meta.outputs.tags }}
        only-severities: critical,high

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
    - name: Deploy to production
      uses: appleboy/ssh-action@v0.1.5
      with:
        host: ${{ secrets.HOST }}
        username: ${{ secrets.USERNAME }}
        key: ${{ secrets.SSH_KEY }}
        script: |
          cd /app
          docker-compose pull
          docker-compose up -d
          docker image prune -f
```

### Monitoring Stack

#### Prometheus + Grafana + AlertManager
```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-lifecycle'
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    networks:
      - monitoring

  alertmanager:
    image: prom/alertmanager:latest
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.ignored-mount-points=^/(sys|proc|dev|host|etc)($|/)'
    networks:
      - monitoring

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    ports:
      - "8080:
