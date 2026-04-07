# Complete Loki Implementation

## Step 1: Install Loki Stack

```bash
# Create namespace
kubectl create namespace observability

# Add Grafana Helm repo
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Loki with Promtail and Grafana
helm install loki-stack grafana/loki-stack \
  --namespace observability \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=50Gi \
  --set loki.persistence.storageClassName=standard \
  --set promtail.enabled=true \
  --set grafana.enabled=true \
  --set grafana.adminPassword=admin123
```

## Step 2: Configure Promtail for Kubernetes

**Create Promtail ConfigMap:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: promtail-config
  namespace: observability
data:
  promtail-config.yaml: |
    clients:
      - url: http://loki:3100/loki/api/v1/push

    scrape_configs:
    # Kubernetes pod logs
    - job_name: kubernetes-pods
      kubernetes_sd_configs:
        - role: pod
      relabel_configs:
        # Pod name
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: pod
        # Namespace
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: namespace
        # App label
        - source_labels: [__meta_kubernetes_pod_label_app]
          action: replace
          target_label: app
        # Container name
        - source_labels: [__meta_kubernetes_pod_container_name]
          action: replace
          target_label: container
        # Pod IP
        - source_labels: [__meta_kubernetes_pod_ip]
          action: replace
          target_label: pod_ip
      pipeline_stages:
        # Parse JSON logs
        - json:
            expressions:
              level: level
              msg: message
              error: error
              user_id: user_id
              trace_id: trace_id
        # Extract labels
        - labels:
            level:
            app:
```

## Step 3: Deploy Promtail DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: promtail
  namespace: observability
spec:
  selector:
    matchLabels:
      app: promtail
  template:
    metadata:
      labels:
        app: promtail
    spec:
      serviceAccountName: promtail
      containers:
      - name: promtail
        image: grafana/promtail:2.9.0
        args:
          - -config.file=/etc/promtail/promtail-config.yaml
        volumeMounts:
        - name: config
          mountPath: /etc/promtail
        - name: varlog
          mountPath: /var/log
          readOnly: true
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "256Mi"
            cpu: "500m"
      volumes:
      - name: config
        configMap:
          name: promtail-config
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: promtail
  namespace: observability

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: promtail
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: promtail
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: promtail
subjects:
- kind: ServiceAccount
  name: promtail
  namespace: observability
```

## Step 4: Application JSON Logging

**Node.js with Winston:**

```javascript
const winston = require('winston');

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.json(),
  defaultMeta: {
    service: 'api',
    environment: process.env.NODE_ENV,
    version: process.env.APP_VERSION,
  },
  transports: [
    new winston.transports.Console({
      format: winston.format.json(),
    }),
  ],
});

// Middleware to log requests
app.use((req, res, next) => {
  const start = Date.now();

  res.on('finish', () => {
    const duration = Date.now() - start;
    logger.info('Request completed', {
      method: req.method,
      path: req.path,
      status: res.statusCode,
      duration,
      user_id: req.user?.id,
      trace_id: req.id,
    });
  });

  next();
});

// Log errors
app.use((err, req, res, next) => {
  logger.error('Unhandled error', {
    error: err.message,
    stack: err.stack,
    path: req.path,
    user_id: req.user?.id,
  });
  res.status(500).json({ error: 'Internal server error' });
});

module.exports = logger;
```

## Step 5: Access Grafana

```bash
# Port forward to Grafana
kubectl port-forward svc/loki-grafana 3000:80 -n observability

# Login at http://localhost:3000
# Default: admin / admin123
```

**Add Loki data source:**

1. Settings → Data Sources → Add data source
2. Select Loki
3. URL: http://loki:3100
4. Save & Test

## Step 6: Create Log Dashboard

**Query examples:**

```logql
# All error logs
{level="error"}

# Errors from production API
{app="api", namespace="production", level="error"}

# Response time > 1000ms
{app="api"} | json | duration > 1000

# Specific error pattern
{app="api"} |= "database connection"

# Error rate per minute
sum(rate({level="error"}[1m]))

# Top 10 error messages
topk(10, sum by (msg) (count_over_time({level="error"}[5m])))
```

**Create dashboard panel:**

1. New Dashboard
2. Add Panel
3. Query:
   ```logql
   sum(rate({app="api", level="error"}[5m]))
   ```
4. Visualization: Graph
5. Set title: "Error Rate"

## Step 7: Set Up Alerting

**Create alert rule:**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: loki-alerts
  namespace: observability
spec:
  groups:
  - name: loki-alerts
    interval: 1m
    rules:
    - alert: HighErrorRate
      expr: sum(rate({level="error"}[5m])) > 10
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "High error rate detected"
        description: "Error rate is {{ $value }} errors/sec"

    - alert: PodOOMKilled
      expr: count_over_time({level="error"} |= "OOMKilled" [5m]) > 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Pod Out of Memory"
        description: "Pod {{ $labels.pod }} was killed"

    - alert: DatabaseConnectionErrors
      expr: count_over_time({app="api"} |= "database connection" [5m]) > 20
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "Database connection issues"
        description: "{{ $value }} connection errors in 5m"
```

## Step 8: Monitor Logs

**Check Promtail status:**

```bash
# Verify Promtail is running
kubectl get pods -n observability -l app=promtail

# Check logs being collected
kubectl logs -n observability -l app=promtail -f

# Verify connection to Loki
kubectl logs -n observability deployment/loki -f
```

**Query in Grafana Explore:**

1. Go to Explore
2. Select Loki data source
3. Enter query:
   ```logql
   {namespace="production"} | json
   ```
4. Run query

## Step 9: Storage Configuration

**Use S3 for production:**

```yaml
# In Loki config
schema_config:
  configs:
  - from: 2024-01-01
    store: boltdb_shipper
    object_store: s3
    schema: v12

storage_config:
  s3:
    s3: s3://aws-region/bucket-name
    access_key_id: ${AWS_ACCESS_KEY_ID}
    secret_access_key: ${AWS_SECRET_ACCESS_KEY}
  boltdb_shipper:
    active_index_directory: /loki/boltdb-shipper-active
    cache_location: /loki/boltdb-shipper-cache
    shared_store: s3
```

## Step 10: Retention Policy

**Set log retention:**

```yaml
# In Loki config
limits_config:
  retention_enabled: true
  retention_period: 30d  # Keep 30 days of logs
```

**Automated cleanup:**

```bash
# Add to scheduler
0 2 * * * loki_cleanup_old_chunks.sh  # Run daily at 2am
```

