# Configuration Management Approaches

## Side-by-Side Comparison

| Approach | Simplicity | Scalability | Security | Flexibility | Best For |
|----------|-----------|------------|----------|------------|----------|
| **Environment Variables** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | All projects |
| **.env Files** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐⭐ | Development only |
| **ConfigMaps** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | Kubernetes apps |
| **Secrets** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | Sensitive data |
| **Config Files** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐ | ⭐⭐⭐⭐⭐ | Complex setups |
| **HashiCorp Consul** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Enterprise |

## Detailed Comparison

### Environment Variables
**Pros:**
- Universal (works everywhere)
- Simple to use and understand
- Platform agnostic
- Built into shell/Docker
- No extra dependencies

**Cons:**
- Hard to manage many variables
- Can pollute environment
- Limited to string values
- Hard to track changes

**Use case:** All new projects (12-factor compliant)

```bash
export DATABASE_URL="postgres://..."
export API_KEY="secret"
npm start
```

### .env Files
**Pros:**
- Convenient for local development
- One file per environment
- IDE friendly
- Easy to share template (.env.example)

**Cons:**
- Development-only solution
- Not suitable for production
- Can accidentally commit secrets
- Requires dotenv library

**Use case:** Local development workflow

```bash
npm i dotenv
# .env file loaded automatically
npm start
```

### Kubernetes ConfigMaps
**Pros:**
- First-class Kubernetes object
- Easy to update without redeploying
- Mounted as volume or env vars
- Tracked by kubectl/GitOps
- Supports large configs

**Cons:**
- Kubernetes-only solution
- Not encrypted at rest (by default)
- Increases cluster manifests
- Requires RBAC configuration

**Use case:** Non-sensitive config in Kubernetes

```yaml
# kubectl apply -f configmap.yaml
# Easy updates: kubectl edit configmap app-config
```

### Kubernetes Secrets
**Pros:**
- Encrypted at rest (etcd encryption)
- First-class Kubernetes object
- Easy rotation
- Tracks secret access (audit logs)
- Supports multiple secret types

**Cons:**
- Kubernetes-only solution
- Still stored in etcd (risk)
- Default encryption weak
- Requires external secret management for best practices

**Use case:** Sensitive data in Kubernetes

```yaml
# Better: Use external Secrets (Vault, AWS Secrets Manager)
# Direct K8s secrets okay for non-critical data
```

### Configuration Libraries
**Pros:**
- Highly flexible
- Supports complex hierarchies
- Multiple file formats (YAML, JSON, TOML)
- Environment override support
- Validation built-in

**Cons:**
- More setup required
- Adds dependency
- Overkill for simple apps
- Can get complex

**Use case:** Complex multi-tier applications

```javascript
const config = require('config');
// Supports dev/staging/prod configs
```

### HashiCorp Consul
**Pros:**
- Enterprise-grade
- Distributed configuration
- Dynamic secrets
- Health checks
- Service discovery

**Cons:**
- Operational overhead
- Learning curve
- Expensive/complex
- Overkill for small projects

**Use case:** Large distributed systems, security-critical apps

## Decision Matrix

```
Is this a production app?
├─ NO (local dev): .env files + environment variables
└─ YES: environment variables only
   ├─ Kubernetes cluster?
   │  ├─ YES: ConfigMaps + Secrets
   │  │  ├─ Sensitive data? Secrets Manager / Vault
   │  │  └─ Config? ConfigMaps
   │  └─ NO: Environment variables
   ├─ Multiple sensitive values?
   │  ├─ YES: HashiCorp Vault / AWS Secrets Manager
   │  └─ NO: Environment variables
   └─ Complex config structure?
      ├─ YES: Config library (node-config)
      └─ NO: Environment variables
```

## Recommended Setup by Project Type

### Small Project (1 developer)
```
Development:   .env file
Production:    Environment variables via Docker/platform
Configuration: Environment variables in code
Security:      None (non-critical)
```

### Growing Team (5+ developers)
```
Development:   .env file + validation
Staging:       ConfigMaps + Kubernetes
Production:    Secrets + ConfigMaps + Kubernetes
Configuration: node-config library with env overrides
Security:      K8s Secrets Manager integration
```

### Enterprise (100+ developers)
```
Development:   .env file with strict rules
Staging:       ArgoCD + Kustomize + ConfigMaps
Production:    HashiCorp Vault + ConfigMaps + Secrets
Configuration: Multi-tier config system
Security:      Vault integration, audit logging, RBAC
```

## Configuration Checklist

- [ ] Never hardcode environment-specific values
- [ ] All configuration comes from environment
- [ ] Secrets never in version control
- [ ] .gitignore includes *.env, *.env.*.local
- [ ] Development and production use same code
- [ ] Configuration validated on startup
- [ ] Defaults provided for non-sensitive values
- [ ] Documentation of required variables (.env.example)
- [ ] Logs never contain secrets (redaction)
- [ ] Rotation plan for long-lived secrets

