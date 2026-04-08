## Execution Pipeline

### 1. CI/CD Build & Push (GitHub Actions)

Extract the short SHA and use it as the primary immutable tag.

```yaml
# .github/workflows/deploy.yml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Get short SHA
        id: sha
        run: echo "short_sha=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT

      - name: Build and Push Immutable Image
        run: |
          docker build -t myapp:${{ steps.sha.outputs.short_sha }} .
          docker push myapp:${{ steps.sha.outputs.short_sha }}
```

### 2. Strict Kubernetes Deployment

Never reference `latest`. Lock the deployment to the Git SHA and leverage Kubernetes revision history.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  revisionHistoryLimit: 10 # Retain 10 ReplicaSets for instant rollbacks
  template:
    spec:
      containers:
        - name: myapp
          # EXACT immutable tag
          image: docker.io/company/myapp:a1b2c3d

          # Enforces local cache usage if the exact SHA exists, saving bandwidth
          imagePullPolicy: IfNotPresent
```

### 3. Instant Rollback Execution

Because the tags are immutable, rolling back is deterministic and safe.

```bash
# Revert to the previous known-good ReplicaSet (and its immutable image)
kubectl rollout undo deployment/myapp

# Or target a specific revision
kubectl rollout undo deployment/myapp --to-revision=3
```
