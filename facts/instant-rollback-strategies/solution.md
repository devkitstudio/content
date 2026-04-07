# Instant Rollback Strategies

## 1. Blue-Green Deployment

**Two identical production environments running simultaneously.**

**How it works:**
- Blue = current production (v1.0)
- Green = new version (v1.1)
- Route 100% traffic to green
- If issue detected, switch back to blue instantly
- Zero downtime, instant rollback

```yaml
# Kubernetes example
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    version: blue  # Point to blue
  ports:
    - port: 80
      targetPort: 3000
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      version: blue
  template:
    metadata:
      labels:
        version: blue
    spec:
      containers:
      - name: app
        image: myapp:v1.0
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
spec:
  replicas: 3
  selector:
    matchLabels:
      version: green
  template:
    metadata:
      labels:
        version: green
    spec:
      containers:
      - name: app
        image: myapp:v1.1
```

**Instant rollback (swap selector):**
```bash
kubectl patch service app-service -p '{"spec":{"selector":{"version":"blue"}}}'
```

## 2. Image Tag Rollback (Kubernetes)

**Keep running different versions, switch instantly.**

```bash
# Current state: running v1.1, detected bug
kubectl set image deployment/app-service app=myapp:v1.0 --record
kubectl rollout status deployment/app-service

# If still problematic, rollback
kubectl rollout undo deployment/app-service
kubectl rollout history deployment/app-service
```

**Automatic rollback on failed readiness probe:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
```

## 3. Feature Flags for Instant Disabling

**Deploy code but disable problematic features without redeploying.**

```javascript
// Feature flag configuration
const featureFlags = {
  'new-checkout': process.env.FEATURE_NEW_CHECKOUT === 'true',
  'v2-api': process.env.FEATURE_V2_API === 'true',
  'ai-recommendations': process.env.FEATURE_AI_RECS === 'true',
};

// In your code
app.post('/checkout', (req, res) => {
  if (featureFlags['new-checkout']) {
    return handleNewCheckout(req, res);
  }
  return handleLegacyCheckout(req, res);
});

// Instant rollback: change env var without redeploy
// kubectl set env deployment/app FEATURE_NEW_CHECKOUT=false
```

**Using a feature flag service:**
```javascript
const flagClient = new LaunchDarkly.LDClient(process.env.LD_SDK_KEY);

app.get('/api/products', async (req, res) => {
  const showNewUI = await flagClient.variation(
    'show-new-product-ui',
    { key: req.user.id },
    false
  );

  if (showNewUI) {
    return res.json(await getProductsV2());
  }
  return res.json(await getProductsV1());
});

// Disable feature instantly via LaunchDarkly dashboard
```

## 4. Database Rollback Considerations

**Forward-compatible migrations:**
```sql
-- Deploy v1.1: Add new column (backward compatible)
ALTER TABLE users ADD COLUMN preferences JSONB DEFAULT '{}';

-- Application code handles both old and new
const prefs = user.preferences || {};

-- Rollback: App continues working with legacy column
-- Remove new code in next deploy
```

**Avoid breaking schema changes:**
```sql
-- BAD: Breaking change (rollback fails)
ALTER TABLE users DROP COLUMN legacy_id;
ALTER TABLE orders MODIFY customer_id INT NOT NULL;

-- GOOD: Safe rollback
ALTER TABLE users ADD COLUMN new_id INT UNIQUE;
-- Backfill data gradually
-- Keep legacy column for rollback safety
```

## 5. Canary Deployment (Gradual Rollout)

**Route small % traffic to new version, increase if healthy.**

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: app
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  service:
    port: 80
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 5
    metrics:
    - name: error-rate
      thresholdRange:
        max: 1
      interval: 1m
    - name: latency
      thresholdRange:
        max: 500
      interval: 1m
  webhooks:
    - name: smoke-tests
      url: http://flagger-loadtester/
      timeout: 30s
      metadata:
        type: smoke
        cmd: "curl http://app-canary/health"
```

**If metrics degrade, automatically rolls back to previous version.**

## Recovery Time Comparison

| Strategy | Rollback Time | Downtime |
|----------|---------------|----------|
| Rolling restart | 10-20 min | 30 sec |
| Blue-green | < 1 sec | 0 sec |
| Feature flags | < 1 sec | 0 sec |
| Canary (auto rollback) | < 2 min | 0 sec |
| Image tag rollback | 2-5 min | 0 sec |
