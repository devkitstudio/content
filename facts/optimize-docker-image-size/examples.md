# Complete Dockerfile Comparison

## BEFORE: Unoptimized Dockerfile (2.3GB)

```dockerfile
FROM node:20
WORKDIR /app

COPY . .
RUN npm install
RUN npm run build

EXPOSE 3000
CMD ["node", "dist/server.js"]
```

**Problems:**
- Includes dev dependencies (2GB)
- Includes source files and build tools
- No layer optimization
- No cache utilization
- .git, .vscode, node_modules duplicated

## AFTER: Optimized Dockerfile (120MB)

```dockerfile
# Stage 1: Dependencies
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Stage 2: Builder
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Stage 3: Runtime
FROM gcr.io/distroless/nodejs20-debian11
COPY --from=builder /app/dist /app/dist
COPY --from=deps /app/node_modules /app/node_modules

WORKDIR /app
EXPOSE 3000
CMD ["dist/server.js"]
```

## Build Script for Optimization Verification

```bash
#!/bin/bash

echo "Building unoptimized image..."
docker build -f Dockerfile.before -t myapp:before .
docker images | grep myapp:before

echo "Building optimized image..."
docker build -f Dockerfile.after -t myapp:after .
docker images | grep myapp:after

echo "Size comparison:"
docker images myapp:before myapp:after --format "table {{.Repository}}:{{.Tag}}\t{{.Size}}"
```

## .dockerignore File

```
# Version control
.git
.gitignore
.gitattributes

# Dependencies (rebuilt in container)
node_modules
npm-debug.log
yarn-error.log

# Development files
.env.local
.env.*.local
.vscode
.idea
.DS_Store
*.swp
*.swo

# Build artifacts
dist
build
coverage
*.tsbuildinfo

# Documentation
README.md
CHANGELOG.md
docs/

# CI/CD
.github
.gitlab-ci.yml
.travis.yml

# Testing
__tests__
__mocks__
*.test.js
*.spec.js
```

## Real-World Example: Node.js + TypeScript

### Before (2.1GB)
```dockerfile
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
CMD ["npm", "start"]
```

### After (95MB)
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json tsconfig.json ./
RUN npm ci

COPY src ./src
RUN npm run build
RUN npm prune --production

FROM gcr.io/distroless/nodejs20-debian11
COPY --from=builder /app/dist /app/dist
COPY --from=builder /app/node_modules /app/node_modules
WORKDIR /app
CMD ["dist/index.js"]
```

## Performance Impact

**Pull time reduction:**
- Before: 5 minutes (2.3GB)
- After: 18 seconds (120MB)
- Improvement: 16.7x faster

**Deployment time:**
- 45 second reduction per deployment
- 30 deployments/month = 22.5 minutes saved
