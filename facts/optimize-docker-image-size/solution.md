# Docker Image Optimization Techniques

## 1. Multi-Stage Builds

**Separate build and runtime stages:**

```dockerfile
# Stage 1: Builder
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Stage 2: Runtime
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

**Key benefit:** Excludes build tools, dev dependencies, and build artifacts from final image.

## 2. Use Alpine Linux Base Images

**Smaller base images:**
- `node:20-alpine` - 200MB (vs 1GB for `node:20`)
- `python:3.11-alpine` - 50MB (vs 1GB for full Python)
- `busybox:latest` - 2MB for minimal utilities

```dockerfile
# Reduced from 1GB to 200MB
FROM node:20-alpine
```

## 3. Layer Optimization

**Order matters - cache frequently changing layers last:**

```dockerfile
FROM node:20-alpine
WORKDIR /app

# Layer 1: Copy package files first (changes rarely)
COPY package*.json ./
RUN npm ci --only=production

# Layer 2: Copy source code (changes frequently)
COPY . .

# Layer 3: Build/compile step
RUN npm run build

EXPOSE 3000
CMD ["node", "dist/server.js"]
```

## 4. Remove Unnecessary Files

**Use .dockerignore to exclude files:**

```
node_modules
npm-debug.log
.git
.gitignore
.DS_Store
dist/
coverage/
.env
.env.local
.vscode
.idea
```

**Remove package manager caches:**

```dockerfile
RUN npm ci --only=production \
    && npm cache clean --force \
    && rm -rf /tmp/*
```

## 5. Minimize Layers

**Combine RUN commands:**

```dockerfile
# Bad: 3 layers
RUN apt-get update
RUN apt-get install -y curl wget
RUN rm -rf /var/lib/apt/lists/*

# Good: 1 layer
RUN apt-get update \
    && apt-get install -y curl wget \
    && rm -rf /var/lib/apt/lists/*
```

## 6. Use Distroless Base Images

**For production: Google distroless images (no shell, no package manager)**

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
COPY . .
RUN npm ci --only=production && npm run build

# Stage 2: Runtime (100-150MB)
FROM gcr.io/distroless/nodejs20-debian11
COPY --from=builder /app/dist /app/dist
COPY --from=builder /app/node_modules /app/node_modules
WORKDIR /app
EXPOSE 3000
CMD ["dist/server.js"]
```

## 7. Compress Build Output

**For Java/large applications:**

```dockerfile
FROM maven:3.8-openjdk-17-slim AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

FROM eclipse-temurin:17-jre-alpine
COPY --from=builder /app/target/app.jar /app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

## Size Comparison

| Approach | Size |
|----------|------|
| Full node:20 + unoptimized | 1.2GB |
| node:20 + multi-stage | 400MB |
| node:20-alpine | 250MB |
| alpine + multi-stage | 120MB |
| distroless + multi-stage | 85MB |
