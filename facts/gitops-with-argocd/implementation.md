# Complete ArgoCD Implementation

## Step 1: Create Git Repository Structure

```bash
git init infrastructure
cd infrastructure

# Create directories
mkdir -p apps/{api,web}/overlays/{dev,staging,prod}
mkdir -p apps/{api,web}/base
mkdir -p infrastructure/{prometheus,argocd,cert-manager}
mkdir -p docs

# Initialize
git add .
git commit -m "Initial repository structure"
git push
```

## Step 2: Define Base Deployment

**apps/api/base/kustomization.yaml:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: myapp

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml

commonLabels:
  app: api
  managed-by: argocd
```

**apps/api/base/deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  selector:
    matchLabels:
      app: api
  replicas: 2
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: myapp:v1.0.0
        ports:
        - containerPort: 3000
          name: http
        env:
        - name: NODE_ENV
          valueFrom:
            configMapKeyRef:
              name: api-config
              key: node-env
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: api-config
              key: log-level
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

**apps/api/base/service.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: http
    protocol: TCP
  selector:
    app: api
```

**apps/api/base/configmap.yaml:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
data:
  node-env: "production"
  log-level: "info"
```

## Step 3: Create Production Overlay

**apps/api/overlays/prod/kustomization.yaml:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

replicas:
  - name: api
    count: 5

patchesStrategicMerge:
  - deployment.yaml

configMapGenerator:
  - name: api-config
    behavior: merge
    literals:
      - log-level=warn

images:
  - name: myapp
    newTag: v1.2.0
```

**apps/api/overlays/prod/deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  template:
    spec:
      containers:
      - name: api
        image: myapp:v1.2.0
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: database-url
```

## Step 4: Create ArgoCD Applications

**argocd-apps/api-prod.yaml:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api-prod
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  source:
    repoURL: https://github.com/myorg/infrastructure
    targetRevision: main
    path: apps/api/overlays/prod

  destination:
    server: https://kubernetes.default.svc
    namespace: myapp

  syncPolicy:
    automated:
      prune: false      # Require manual approval to delete
      selfHeal: false   # Require manual approval to sync
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  notifications:
    slack:
      channel: '#deployments'
      onSuccess: 'Deployment {{.app.metadata.name}} succeeded'
      onFailed: 'Deployment {{.app.metadata.name}} failed'

  revisionHistoryLimit: 10
```

**argocd-apps/api-dev.yaml:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api-dev
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/myorg/infrastructure
    targetRevision: dev
    path: apps/api/overlays/dev

  destination:
    server: https://kubernetes.default.svc
    namespace: myapp-dev

  syncPolicy:
    automated:
      prune: true      # Aggressive sync for dev
      selfHeal: true
    syncOptions:
    - CreateNamespace=true

  revisionHistoryLimit: 5
```

## Step 5: Apply ArgoCD Application

```bash
# Create namespace and application
kubectl create namespace argocd
kubectl apply -f argocd-apps/api-prod.yaml
kubectl apply -f argocd-apps/api-dev.yaml

# Monitor sync status
kubectl get application -n argocd
kubectl describe application api-prod -n argocd

# View UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# https://localhost:8080
```

## Step 6: Workflow

**Developer pushes code changes:**
```bash
# 1. Developer creates feature branch
git checkout -b feature/new-endpoint

# 2. Updates image tag in kustomization
# apps/api/overlays/prod/kustomization.yaml
# images:
#   - name: myapp
#     newTag: v1.3.0

# 3. Creates pull request
git push origin feature/new-endpoint

# 4. CI runs tests and builds image
# 5. Reviewer approves PR
# 6. Merge to main
```

**ArgoCD automatically deploys:**
```bash
# ArgoCD detects main branch change
# Syncs production deployment
# Pulls image myapp:v1.3.0
# Performs rolling update with health checks
# Notifies Slack on success/failure
```

**If issue detected - instant rollback:**
```bash
# Option 1: Revert Git commit
git revert <commit-hash>
git push origin main
# ArgoCD syncs within 3-5 minutes

# Option 2: Manual ArgoCD rollback
argocd app rollback api-prod <revision>

# Option 3: Immediate manual sync to specific revision
kubectl patch application api-prod -n argocd \
  -p '{"spec":{"source":{"targetRevision":"v1.2.0"}}}' --type merge
```

## Step 7: Monitoring and Alerts

**Check application status:**
```bash
# CLI
argocd app get api-prod
argocd app wait api-prod

# Kubernetes events
kubectl get events -n argocd | grep api-prod

# Real-time logs
kubectl logs -n argocd deploy/argocd-application-controller -f
```

**Set up Slack notifications:**
```yaml
# Configure ArgoCD notifications
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token
  template.app-deployed: |
    message: |
      Deployment {{.app.metadata.name}}:{{.app.status.operationState.finishedAt}}
      Status: {{.app.status.operationState.phase}}
      {{if eq .app.status.operationState.phase "Succeeded"}}:white_check_mark:{{end}}
```

