# GitOps with ArgoCD

## 1. GitOps Principles

**Core concepts:**
- Git is single source of truth for infrastructure
- Desired state declared in Git repositories
- Automatic synchronization between Git and cluster
- All changes tracked, auditable, reversible

## 2. ArgoCD Installation

```bash
# Install ArgoCD namespace
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Login with: admin / $(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)

# Change default password
argocd account update-password --account admin --new-password <new-password>
```

## 3. Repository Structure

**Recommended layout:**

```
infrastructure/
├── apps/
│   ├── api/
│   │   ├── base/
│   │   │   ├── kustomization.yaml
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── configmap.yaml
│   │   └── overlays/
│   │       ├── dev/
│   │       │   └── kustomization.yaml
│   │       ├── staging/
│   │       │   └── kustomization.yaml
│   │       └── prod/
│   │           ├── kustomization.yaml
│   │           └── values.yaml
│   └── web/
│       └── [similar structure]
├── infrastructure/
│   ├── prometheus/
│   ├── argocd/
│   └── cert-manager/
└── docs/
    └── README.md
```

## 4. Basic Application Manifest

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: myapp:v1.2.0
        ports:
        - containerPort: 3000
        env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: api-config
              key: log-level
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"

---
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: myapp
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 3000
  type: ClusterIP

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: myapp
data:
  log-level: "info"
  database-url: "postgres://db:5432/myapp"
```

## 5. ArgoCD Application Resource

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api-app
  namespace: argocd
spec:
  # Git repo source
  source:
    repoURL: https://github.com/myorg/infrastructure
    targetRevision: main
    path: apps/api/overlays/prod

  # Destination cluster
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp

  # Sync policy
  syncPolicy:
    automated:
      prune: true      # Delete resources not in Git
      selfHeal: true   # Auto-sync if cluster drifts
    syncOptions:
    - CreateNamespace=true

  # Health assessment
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas  # Ignore HPA changes

  # Notification on sync
  notifications:
  - slack:
      channel: '#deployments'
      message: 'Deployment {{.app.metadata.name}} synced'
```

## 6. Kustomization for Environment Management

**Base configuration:**
```yaml
# apps/api/base/kustomization.yaml
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

**Production overlay:**
```yaml
# apps/api/overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

patchesStrategicMerge:
  - deployment.yaml

replicas:
  - name: api
    count: 5

configMapGenerator:
  - name: api-config
    behavior: merge
    literals:
      - log-level=warn
      - database-url=postgres://prod-db:5432/myapp
```

**Production patch:**
```yaml
# apps/api/overlays/prod/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 5
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
```

## 7. ArgoCD Sync Policies

**Manual sync (safest - requires review):**
```yaml
syncPolicy:
  syncOptions:
  - CreateNamespace=true
  # No automated sync
```

**Auto-sync with safety:**
```yaml
syncPolicy:
  automated:
    prune: false      # Don't delete unknown resources
    selfHeal: false   # Manual review before sync
  syncOptions:
  - CreateNamespace=true
```

**Full auto-sync (aggressive):**
```yaml
syncPolicy:
  automated:
    prune: true       # Delete resources not in Git
    selfHeal: true    # Auto-sync on drift
  syncOptions:
  - CreateNamespace=true
  retry:
    limit: 5
    backoff:
      duration: 5s
      factor: 2
      maxDuration: 3m
```

## 8. Enforcing PR Reviews

**Prevent direct cluster modifications:**

```yaml
# 1. Create RBAC to prevent direct kubectl apply
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: argocd-only
  namespace: myapp
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list"]
  # No create/update/patch/delete!

---
# 2. Bind to developers
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: argocd-only
  namespace: myapp
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: argocd-only
subjects:
- kind: Group
  name: "developers"
  apiGroup: rbac.authorization.k8s.io
```

**GitHub branch protection:**
```yaml
# Require PR review before merge to main
settings:
  branch_protection_rules:
  - pattern: "main"
    required_pull_request_reviews:
      required_approving_review_count: 2
      dismiss_stale_reviews: true
    required_status_checks:
      strict: true
      contexts:
        - "CI/CD Pipeline"
        - "Kustomize Validation"
        - "Policy Enforcement"
```

