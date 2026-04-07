## The Problem with `latest`

```yaml
# Kubernetes deployment using "latest"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:latest  # DANGER!
```

**Problems:**
- `latest` is mutable (can change anytime)
- No one knows what version is running
- Image pulled from registry → cache miss → old image
- Kubernetes caches images → may use old `latest`
- Rollback impossible (what to rollback to?)

**Scenario:**
```
Monday 9 AM: Deploy myapp:latest (v1.0.0)
Monday 10 AM: Someone builds v1.0.1 → overwrites myapp:latest
Monday 11 AM: Pod restarts → pulls new myapp:latest (v1.0.1)
           But v1.0.1 has a bug!
           Production broken
           No way to know what happened
```

## Solution: Immutable Tags

### Strategy 1: Git SHA Tags (Recommended)

```bash
# .github/workflows/deploy.yml
name: Build and Push

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Get short SHA
        id: sha
        run: echo "short_sha=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT

      - name: Build image
        run: docker build -t myapp:${{ steps.sha.outputs.short_sha }} .

      - name: Push to registry
        run: docker push myapp:${{ steps.sha.outputs.short_sha }}

      - name: Deploy
        run: |
          kubectl set image deployment/myapp \
            myapp=myapp:${{ steps.sha.outputs.short_sha }}
```

**Result:**
```
Commit abc1234 → Image: myapp:abc1234
Commit def5678 → Image: myapp:def5678

Immutable! No overwriting.
```

**Advantages:**
- Unique per commit
- Traceable to exact code
- Immutable forever
- Easy rollback (git history)

### Strategy 2: Semantic Versioning

```bash
# Only tag releases in main/master branch
git tag -a v1.2.3 -m "Release v1.2.3"
git push origin v1.2.3

# CI automatically builds and pushes
# Trigger: on v*.*.* tag
# Image: myapp:v1.2.3
```

**Example:**
```
v1.0.0 → myapp:v1.0.0 (immutable, released)
v1.1.0 → myapp:v1.1.0 (immutable, released)
v1.1.1 → myapp:v1.1.1 (immutable, patch)

Production uses: myapp:v1.1.1
Can rollback to: myapp:v1.1.0 or myapp:v1.0.0
```

**Advantages:**
- Clear release versions
- Semantic meaning
- Professional version scheme
- Good for public releases

### Strategy 3: Build Number / Timestamp

```bash
# .gitlab-ci.yml
variables:
  BUILD_ID: "$CI_PIPELINE_ID"
  BUILD_TIMESTAMP: "$(date +%Y%m%d_%H%M%S)"

build:
  script:
    - docker build -t myapp:${BUILD_ID} .
    - docker build -t myapp:${BUILD_TIMESTAMP} .
    - docker push myapp:${BUILD_ID}
    - docker push myapp:${BUILD_TIMESTAMP}
```

**Result:**
```
Pipeline 12345 → myapp:12345
Timestamp 20240406_140230 → myapp:20240406_140230
```

## Complete Strategy: Multi-Tag Approach

Combine strategies for flexibility:

```bash
#!/bin/bash
# build-and-push.sh

GIT_SHA=$(git rev-parse --short HEAD)
VERSION=$(git describe --tags --always)
BUILD_ID=${CI_PIPELINE_ID}
REGISTRY="docker.io/company"
IMAGE_NAME="myapp"

# Build
docker build -t ${REGISTRY}/${IMAGE_NAME}:${GIT_SHA} .

# Push multiple tags (same image, different labels)
docker tag ${REGISTRY}/${IMAGE_NAME}:${GIT_SHA} \
           ${REGISTRY}/${IMAGE_NAME}:${VERSION}

docker tag ${REGISTRY}/${IMAGE_NAME}:${GIT_SHA} \
           ${REGISTRY}/${IMAGE_NAME}:${BUILD_ID}

docker tag ${REGISTRY}/${IMAGE_NAME}:${GIT_SHA} \
           ${REGISTRY}/${IMAGE_NAME}:latest-dev

# On main branch only:
if [ $CI_COMMIT_BRANCH == "main" ]; then
  docker tag ${REGISTRY}/${IMAGE_NAME}:${GIT_SHA} \
             ${REGISTRY}/${IMAGE_NAME}:latest
fi

# Push all tags
docker push ${REGISTRY}/${IMAGE_NAME}:${GIT_SHA}
docker push ${REGISTRY}/${IMAGE_NAME}:${VERSION}
docker push ${REGISTRY}/${IMAGE_NAME}:${BUILD_ID}
docker push ${REGISTRY}/${IMAGE_NAME}:latest-dev
docker push ${REGISTRY}/${IMAGE_NAME}:latest
```

**Tags meaning:**
- `myapp:abc1234` - Exact commit (immutable)
- `myapp:v1.2.3` - Release version (immutable)
- `myapp:12345` - Build number (for CI logs)
- `myapp:latest-dev` - Latest dev build (mutable)
- `myapp:latest` - Latest stable (only on main)

## Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp
        # Use SPECIFIC SHA tag (immutable)
        image: docker.io/company/myapp:abc1234

        # NEVER use latest!
        # image: docker.io/company/myapp:latest ❌

        # Optional: Image pull policy
        imagePullPolicy: IfNotPresent  # Use cached if exists

  # Trigger rollout on image change
  revisionHistoryLimit: 10  # Keep 10 old ReplicaSets
```

## Rollback Strategy

```bash
# Check deployment history
kubectl rollout history deployment/myapp

# Output:
# REVISION  CHANGE-CAUSE
# 1         Build 12345: myapp:v1.1.0
# 2         Build 12346: myapp:v1.1.1 (broken!)
# 3         Build 12347: myapp:v1.1.2

# Rollback to previous
kubectl rollout undo deployment/myapp

# Rollback to specific revision
kubectl rollout undo deployment/myapp --to-revision=1

# Get exact image of a revision
kubectl rollout history deployment/myapp --revision=1
```

## Image Registry Best Practices

```bash
# Use specific registries with OCI compliance
docker push registry.example.com/myapp:abc1234

# Tag format: registry/namespace/image:tag
# ✓ registry.example.com/prod/myapp:v1.2.3
# ✓ docker.io/company/myapp:abc1234
# ❌ myapp:latest
# ❌ latest (no registry, dangerous)

# Set image pull secrets for private registries
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass
```
