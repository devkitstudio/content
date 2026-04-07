# Environment Configuration Management

## 1. Environment Variables (12-Factor App)

**Recommended approach - simple and clean:**

```javascript
// config.js
module.exports = {
  port: process.env.PORT || 3000,
  nodeEnv: process.env.NODE_ENV || 'development',
  logLevel: process.env.LOG_LEVEL || 'info',
  database: {
    url: process.env.DATABASE_URL || 'postgres://localhost/dev',
    pool: parseInt(process.env.DB_POOL_SIZE || 10),
  },
  redis: {
    url: process.env.REDIS_URL || 'redis://localhost:6379',
  },
  api: {
    key: process.env.API_KEY,  // Must be provided
    timeout: parseInt(process.env.API_TIMEOUT || 5000),
  },
};
```

**Usage in code:**

```javascript
const config = require('./config');

console.log(`Starting server on port ${config.port}`);
console.log(`Environment: ${config.nodeEnv}`);

app.listen(config.port, () => {
  console.log('Server ready');
});
```

**Docker with env vars:**

```bash
# Run with environment variables
docker run -e NODE_ENV=production \
           -e DATABASE_URL=postgres://prod-db:5432/app \
           -e LOG_LEVEL=warn \
           myapp:latest

# Or from file
docker run --env-file .env.prod myapp:latest
```

## 2. .env Files (Development Only)

**.env for development:**

```bash
# .env
NODE_ENV=development
PORT=3000
LOG_LEVEL=debug
DATABASE_URL=postgres://localhost/dev
API_KEY=dev-key-12345
REDIS_URL=redis://localhost:6379
```

**.env.prod (NOT in Git):**

```bash
# .env.prod (add to .gitignore)
NODE_ENV=production
PORT=3000
LOG_LEVEL=warn
DATABASE_URL=postgres://prod-db:5432/app
API_KEY=actual-prod-key
REDIS_URL=redis://redis.prod:6379
```

**Load .env in development:**

```javascript
// app.js - ONLY in development
if (process.env.NODE_ENV !== 'production') {
  require('dotenv').config();
}

const config = require('./config');
```

**.gitignore:**

```
.env
.env.*.local
.env.prod
.env.staging
.env.*.secret
```

## 3. Kubernetes ConfigMaps and Secrets

**ConfigMap for non-sensitive config:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: myapp
data:
  LOG_LEVEL: "info"
  DATABASE_POOL_SIZE: "20"
  CACHE_TTL: "3600"
  API_TIMEOUT: "5000"
  FEATURE_FLAG_NEW_UI: "true"
```

**Secret for sensitive data:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: myapp
type: Opaque
stringData:
  DATABASE_URL: "postgres://user:pass@db:5432/app"
  API_KEY: "sk-1234567890abcdef"
  JWT_SECRET: "super-secret-key-change-me"
```

**Pod consuming both:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secrets
        env:
        - name: NODE_ENV
          value: "production"
```

## 4. Environment-Specific Deployment Files

**Kustomize overlays approach:**

```
config/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── dev/
│   ├── configmap.yaml
│   └── kustomization.yaml
├── staging/
│   ├── configmap.yaml
│   └── kustomization.yaml
└── prod/
    ├── configmap.yaml
    └── kustomization.yaml
```

**base/kustomization.yaml:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
```

**dev/kustomization.yaml:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../base

configMapGenerator:
  - name: app-config
    behavior: merge
    literals:
      - LOG_LEVEL=debug
      - DATABASE_POOL_SIZE=5
      - FEATURE_FLAG_NEW_UI=true
```

**prod/kustomization.yaml:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../base

configMapGenerator:
  - name: app-config
    behavior: merge
    literals:
      - LOG_LEVEL=warn
      - DATABASE_POOL_SIZE=50
      - FEATURE_FLAG_NEW_UI=false
```

**Deploy by environment:**

```bash
# Development
kubectl apply -k config/dev

# Production
kubectl apply -k config/prod
```

## 5. Application Configuration Library

**Using node-config library:**

```javascript
// config/default.js
module.exports = {
  port: 3000,
  logLevel: 'info',
  database: {
    url: 'postgres://localhost/app',
    pool: 10,
  },
};

// config/production.js
module.exports = {
  port: 3000,
  logLevel: 'warn',
  database: {
    url: 'postgres://prod-db:5432/app',
    pool: 50,
  },
};

// config/test.js
module.exports = {
  port: 9999,
  logLevel: 'debug',
  database: {
    url: 'postgres://localhost/test',
    pool: 2,
  },
};

// app.js
const config = require('config');
const logger = require('./logger')(config.logLevel);

logger.info(`Starting app`, { env: process.env.NODE_ENV });
```

**Usage:**

```bash
# Development (uses config/default.js)
npm start

# Production (uses config/production.js + config/default.js merged)
NODE_ENV=production npm start

# Test (uses config/test.js)
NODE_ENV=test npm test
```

## 6. Environment Variables Validation

**Ensure required vars are set:**

```javascript
// validateConfig.js
const required = [
  'DATABASE_URL',
  'API_KEY',
  'JWT_SECRET',
];

const missing = required.filter(key => !process.env[key]);

if (missing.length > 0) {
  console.error(`Missing required environment variables: ${missing.join(', ')}`);
  process.exit(1);
}
```

**Schema validation:**

```javascript
// config.js
const schema = {
  PORT: { type: 'number', default: 3000 },
  NODE_ENV: { type: 'string', enum: ['dev', 'staging', 'prod'], required: true },
  DATABASE_URL: { type: 'string', required: true },
  API_KEY: { type: 'string', required: true },
  LOG_LEVEL: { type: 'string', enum: ['debug', 'info', 'warn', 'error'], default: 'info' },
  DB_POOL_SIZE: { type: 'number', default: 10, min: 1, max: 100 },
};

function validateEnv() {
  const errors = [];

  for (const [key, rules] of Object.entries(schema)) {
    const value = process.env[key];

    if (rules.required && !value) {
      errors.push(`${key} is required`);
      continue;
    }

    if (value && rules.enum && !rules.enum.includes(value)) {
      errors.push(`${key} must be one of: ${rules.enum.join(', ')}`);
    }

    if (value && rules.type === 'number') {
      const num = parseInt(value);
      if (isNaN(num)) errors.push(`${key} must be a number`);
      if (rules.min && num < rules.min) errors.push(`${key} must be >= ${rules.min}`);
      if (rules.max && num > rules.max) errors.push(`${key} must be <= ${rules.max}`);
    }
  }

  if (errors.length > 0) {
    console.error('Configuration errors:');
    errors.forEach(err => console.error(`  - ${err}`));
    process.exit(1);
  }
}

validateEnv();
```

## 7. Configuration Priority

**Load configuration with proper precedence:**

```
Highest Priority:
1. Command line arguments
2. Environment variables
3. .env file (development only)
4. config/NODE_ENV.js file
5. config/default.js
6. Hardcoded defaults

Lowest Priority
```

**Example:**

```javascript
const defaultConfig = require('./config/default');
const envConfig = require(`./config/${process.env.NODE_ENV || 'development'}`);

// Deep merge
const config = {
  ...defaultConfig,
  ...envConfig,
  // Override with env vars
  port: process.env.PORT || envConfig.port || defaultConfig.port,
  logLevel: process.env.LOG_LEVEL || envConfig.logLevel || defaultConfig.logLevel,
};

module.exports = config;
```

