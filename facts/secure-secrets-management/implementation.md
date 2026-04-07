# Complete Vault Integration Guide

## Step 1: Install Vault on Kubernetes

```bash
# Add Helm repo
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

# Create namespace
kubectl create namespace vault

# Install Vault
helm install vault hashicorp/vault \
  --namespace vault \
  --set server.dataStorage.size=10Gi \
  --set ui.enabled=true
```

## Step 2: Initialize and Unseal Vault

```bash
# Wait for Vault to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=vault -n vault --timeout=300s

# Initialize Vault
kubectl exec -n vault vault-0 -- vault operator init \
  -key-shares=3 \
  -key-threshold=2 \
  -format=json > vault-init.json

# Extract unseal keys and root token
export UNSEAL_KEY_1=$(jq -r '.unseal_keys_b64[0]' vault-init.json)
export UNSEAL_KEY_2=$(jq -r '.unseal_keys_b64[1]' vault-init.json)
export ROOT_TOKEN=$(jq -r '.root_token' vault-init.json)

# Unseal Vault (need 2 of 3 keys)
kubectl exec -n vault vault-0 -- vault operator unseal $UNSEAL_KEY_1
kubectl exec -n vault vault-0 -- vault operator unseal $UNSEAL_KEY_2

# Verify unsealed
kubectl exec -n vault vault-0 -- vault status
```

## Step 3: Configure Kubernetes Authentication

**Enable Kubernetes auth method:**

```bash
kubectl exec -n vault vault-0 -- vault auth enable kubernetes

# Get K8s host and CA certificate
VAULT_SA_NAME=$(kubectl get sa -n vault vault -o jsonpath='{.secrets[0].name}')
SA_JWT_TOKEN=$(kubectl get secret $VAULT_SA_NAME -n vault -o jsonpath='{.data.token}' | base64 --decode)
SA_CA_CRT=$(kubectl get secret $VAULT_SA_NAME -n vault -o jsonpath='{.data.ca\.crt}' | base64 --decode)
K8S_HOST=$(kubectl cluster-info | grep 'Kubernetes master' | awk '/https/ {print $NF}')

# Configure auth
kubectl exec -n vault vault-0 -- vault write auth/kubernetes/config \
  token_reviewer_jwt="$SA_JWT_TOKEN" \
  kubernetes_host="$K8S_HOST" \
  kubernetes_ca_cert="$SA_CA_CRT" \
  issuer="https://kubernetes.default.svc.cluster.local"
```

## Step 4: Create Secret and Policy

**Store secrets:**

```bash
# Enable secret engine
kubectl exec -n vault vault-0 -- vault secrets enable -path=secret kv-v2

# Create secret
kubectl exec -n vault vault-0 -- vault kv put secret/myapp \
  database_password="super-secret-password" \
  api_key="api-key-12345" \
  jwt_secret="jwt-secret-key"
```

**Create policy for application:**

```bash
# Create policy file
cat > myapp-policy.hcl <<EOF
path "secret/data/myapp" {
  capabilities = ["read", "list"]
}

path "secret/metadata/myapp" {
  capabilities = ["read", "list"]
}
EOF

# Write policy to Vault
kubectl exec -n vault vault-0 -- vault write -f /dev/stdin auth/kubernetes/role/myapp \
  bound_service_account_names=myapp \
  bound_service_account_namespaces=default \
  policies=myapp \
  ttl=24h < myapp-policy.hcl
```

## Step 5: Create Kubernetes Role and Service Account

```yaml
# ServiceAccount for app
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp
  namespace: default

---
# ClusterRole for reading secrets from pods
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: vault-auth-delegator
rules:
- apiGroups: ["auth.k8s.io"]
  resources: ["tokenreviews"]
  verbs: ["create"]

---
# Bind vault SA to the role
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vault-auth-delegator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: vault-auth-delegator
subjects:
- kind: ServiceAccount
  name: vault
  namespace: vault
```

## Step 6: Deploy Application with Vault

**Application code accessing Vault:**

```javascript
// app.js - Using node-vault
const vault = require('node-vault')();

async function authenticateWithKubernetes() {
  // Read JWT token from service account
  const fs = require('fs');
  const jwt = fs.readFileSync(
    '/var/run/secrets/kubernetes.io/serviceaccount/token',
    'utf8'
  );

  // Authenticate to Vault
  const result = await vault.kubernetesLogin({
    role: 'myapp',
    jwt: jwt,
  });

  // Use token for future requests
  vault.token = result.auth.client_token;

  return result.auth.client_token;
}

async function getSecrets() {
  await authenticateWithKubernetes();

  const secrets = await vault.read('secret/data/myapp');
  return secrets.data.data;
}

// Express app
const express = require('express');
const app = express();

app.get('/health', (req, res) => {
  res.json({ status: 'ok' });
});

app.post('/api/data', async (req, res) => {
  const secrets = await getSecrets();

  // Use secrets
  const dbPassword = secrets.database_password;
  const apiKey = secrets.api_key;

  // Never log secrets!
  console.log('Processing request (secrets redacted)');

  res.json({ success: true });
});

app.listen(3000, () => {
  console.log('App listening on port 3000');
});

module.exports = app;
```

**Kubernetes Deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      serviceAccountName: myapp  # Use custom SA
      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 3000
        env:
        - name: VAULT_ADDR
          value: "http://vault.vault:8200"
        - name: VAULT_NAMESPACE
          value: ""
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
```

## Step 7: Secret Rotation

**Manual rotation:**

```bash
# Update secret in Vault
kubectl exec -n vault vault-0 -- vault kv put secret/myapp \
  database_password="new-password-2024" \
  api_key="api-key-12345" \
  jwt_secret="jwt-secret-key"

# All pods automatically use new secret
# No restart needed!
```

**Automatic rotation with ExternalSecret:**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault
spec:
  provider:
    vault:
      server: "http://vault.vault:8200"
      path: "secret"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "myapp"

---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secrets
spec:
  refreshInterval: 1h  # Check every hour
  secretStoreRef:
    name: vault
    kind: SecretStore
  target:
    name: myapp-secret
    creationPolicy: Owner
  data:
  - secretKey: database_password
    remoteRef:
      key: myapp
      property: database_password
  - secretKey: api_key
    remoteRef:
      key: myapp
      property: api_key
  - secretKey: jwt_secret
    remoteRef:
      key: myapp
      property: jwt_secret
```

## Step 8: Audit and Monitoring

**Enable audit logging:**

```bash
# Verify audit logs enabled
kubectl exec -n vault vault-0 -- vault audit list

# View audit logs
kubectl exec -n vault vault-0 -- tail -f /vault/logs/audit.log
```

**Monitor secret access:**

```bash
# See who accessed what secret
kubectl exec -n vault vault-0 -- vault audit list

# Example log:
# {"type":"request","auth":{"client_token":"s.xxxxxxx","entity_id":"client-id"},...,"request_path":"secret/data/myapp","operation":"read"}
```

## Step 9: Secret Backup

**Backup Vault state:**

```bash
# Take snapshot
kubectl exec -n vault vault-0 -- vault operator raft snapshot save backup.snap

# Copy out
kubectl cp vault/vault-0:backup.snap ./vault-backup.snap

# Store safely (encrypted, offline)
gpg --encrypt vault-backup.snap
```

## Step 10: Testing

**Test secret access:**

```bash
# Port forward to Vault
kubectl port-forward -n vault svc/vault 8200:8200 &

# Test API
curl -s http://localhost:8200/v1/secret/data/myapp \
  -H "X-Vault-Token: $ROOT_TOKEN" | jq .

# Test pod access
kubectl run -it --rm test --image=vault:latest \
  --overrides='{"spec":{"serviceAccountName":"myapp"}}' \
  -- /bin/sh

# Inside pod:
# VAULT_ADDR=http://vault.vault:8200
# vault login -method=kubernetes role=myapp
# vault kv get secret/myapp
```

