# Secure Secrets Management

## 1. Problem: Plain Environment Variables

**Why it's insecure:**

```bash
# Anyone with server access can read
env | grep -i password
# DATABASE_PASSWORD=MySecretPassword123

# Visible in running processes
ps aux | grep node
# /usr/bin/node server.js --db-password=secret

# In Kubernetes pod
kubectl exec pod -- env
# DATABASE_PASSWORD=secret (visible to anyone with pod access)
```

**Exposure vectors:**
- Server compromise: attacker reads /proc/pid/environ
- Container inspection: docker inspect shows env vars
- Kubernetes audit: kubectl exec reveals secrets
- Log files: secrets logged accidentally
- Code review: secrets checked into git

## 2. HashiCorp Vault (Enterprise Solution)

**Architecture:**

```
┌─────────────────────┐
│   Your Application  │
└─────────┬───────────┘
          │
          │ (Request secret)
          ↓
    ┌──────────────┐
    │  Vault Server │  (Encrypted secret store)
    └──────────────┘
          ↓
    ┌──────────────┐
    │ Audit Logs   │  (Access tracked)
    └──────────────┘
```

**Install Vault:**

```bash
# Using Docker
docker run --cap-add=IPC_LOCK \
  -e VAULT_DEV_ROOT_TOKEN_ID=myroot \
  -p 8200:8200 \
  vault:latest

# Or on Kubernetes
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault \
  --namespace vault --create-namespace
```

**Store secrets:**

```bash
# Initialize Vault
vault operator init

# Unseal Vault
vault operator unseal

# Store secret
vault kv put secret/my-app \
  database-password=secret123 \
  api-key=abc-def-ghi

# Retrieve secret
vault kv get secret/my-app
# ===== Metadata =====
# Key              Value
# ---              -----
# database-password    secret123
# api-key              abc-def-ghi
```

**Application integration:**

```javascript
// Node.js with node-vault
const vault = require('node-vault')({
  endpoint: 'http://vault:8200',
  token: process.env.VAULT_TOKEN,  // Only token in env vars
});

async function getSecrets() {
  const secrets = await vault.read('secret/my-app');
  return {
    dbPassword: secrets.data.data['database-password'],
    apiKey: secrets.data.data['api-key'],
  };
}

const secrets = await getSecrets();
const pool = new Pool({
  connectionString: `postgres://user:${secrets.dbPassword}@db:5432/app`,
});
```

**Kubernetes authentication:**

```bash
# Pod mounts SA token
# Pod authenticates to Vault with SA token
# Vault verifies pod identity
# Returns secrets without storing in etcd

vault write auth/kubernetes/role/my-app \
  bound_service_account_names=my-app \
  bound_service_account_namespaces=default \
  policies=my-app-policy \
  ttl=1h
```

## 3. Sealed Secrets (Kubernetes Native)

**Simpler alternative: encrypt secrets with cluster key**

```
┌──────────────────────────┐
│ Sealed Secret (encrypted)│
└──────────────────────────┘
          ↓
      ┌────────┐
      │ K8s    │  (Decrypt with cluster key)
      │ Sealed │
      │Secrets │
      │ control│
      └────────┘
          ↓
┌──────────────────────────┐
│  Secret (readable in pod)│
└──────────────────────────┘
```

**Install Sealed Secrets:**

```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.0/controller.yaml
```

**Create sealed secret:**

```bash
# Create regular secret
kubectl create secret generic my-secret \
  --from-literal=password=secret123 \
  --dry-run=client -o yaml > secret.yaml

# Seal it (encrypt with cluster public key)
kubeseal -f secret.yaml -w sealed-secret.yaml

# Safe to commit sealed-secret.yaml to git!
git add sealed-secret.yaml
git push
```

**Sealed secret example:**

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: my-secret
  namespace: default
spec:
  encryptedData:
    password: AgBvV8FWa2j4x9k2n3k2j1h9x8v7c6b5a4z3y2w1...
  template:
    metadata:
      name: my-secret
      namespace: default
    type: Opaque
```

**Usage in pods:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: DATABASE_PASSWORD
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: password
```

## 4. AWS Secrets Manager (Cloud Native)

**Store secrets in AWS:**

```bash
# Store secret
aws secretsmanager create-secret \
  --name my-app/database-password \
  --secret-string "password123"

# Retrieve secret
aws secretsmanager get-secret-value \
  --secret-id my-app/database-password \
  --query SecretString
```

**Pod retrieves from AWS:**

```javascript
const AWS = require('aws-sdk');
const secretsManager = new AWS.SecretsManager();

async function getSecret(secretName) {
  const secret = await secretsManager.getSecretValue({
    SecretId: secretName,
  }).promise();
  return JSON.parse(secret.SecretString);
}

const secrets = await getSecret('my-app/database-password');
const dbPassword = secrets['database-password'];
```

**IAM policy for pod:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:*:*:secret:my-app/*"
    }
  ]
}
```

## 5. External Secrets Operator (For Kubernetes)

**Sync Vault/AWS secrets to K8s Secret:**

```bash
# Install operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets-system --create-namespace
```

**Define SecretStore:**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-secret-store
spec:
  provider:
    vault:
      server: "http://vault:8200"
      path: "secret"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "my-app"
```

**Define ExternalSecret:**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-secrets
spec:
  refreshInterval: 1h  # Sync every hour
  secretStoreRef:
    name: vault-secret-store
    kind: SecretStore
  target:
    name: my-app-secret  # Create K8s Secret with this name
    creationPolicy: Owner
  data:
  - secretKey: database-password
    remoteRef:
      key: my-app
      property: database-password
  - secretKey: api-key
    remoteRef:
      key: my-app
      property: api-key
```

**Automatic syncing:**
- K8s Secret `my-app-secret` created automatically
- Updated every hour from Vault
- Pod uses normal Secret reference

## 6. Secret Rotation

**Rotate API keys without redeployment:**

```javascript
// With Vault
const vault = require('node-vault')({
  endpoint: 'http://vault:8200',
  token: process.env.VAULT_TOKEN,
});

// Fetch secrets on each request (short TTL)
app.use(async (req, res, next) => {
  const secrets = await vault.read('secret/my-app');
  req.secrets = secrets.data.data;
  next();
});

// Use fresh secret every time
app.post('/api/data', async (req, res) => {
  const apiKey = req.secrets['api-key'];  // Always latest
  const result = await externalAPI(req.body, { apiKey });
  res.json(result);
});
```

**Periodic rotation in Vault:**

```bash
# Update secret in Vault
vault kv put secret/my-app api-key=new-key-2024

# All pods immediately use new key
# No restart needed!
```

## 7. Secret Best Practices

**Do:**
- Use strong encryption (AES-256, TLS)
- Rotate secrets regularly
- Audit all secret access
- Limit secret exposure window
- Use service accounts with minimal permissions
- Encrypt secrets at rest

**Don't:**
- Never commit secrets to git
- Never log secrets
- Never hardcode in code
- Never store in environment variables (production)
- Never share credentials
- Never reuse secrets across environments

## 8. Comparison Table

| Solution | Encryption | Rotation | Cost | Complexity |
|----------|-----------|----------|------|-----------|
| **Env Vars** | None | Manual | Free | Easy |
| **Sealed Secrets** | AES-256 | Manual | Free | Low |
| **Vault** | AES-256 | Auto | Self-hosted | Medium |
| **AWS Secrets Manager** | AES-256 | Auto | $0.40/secret/month | Low |
| **External Secrets** | Via backend | Auto | Free | Medium |

