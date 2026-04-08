## Log Collection & Execution Pipeline

To make label-based indexing work effectively, applications must output structured JSON, and Promtail must be configured to map Kubernetes metadata to Loki labels safely.

### 1. Application-Level Structured Logging (Node.js)

Your application must output single-line JSON strings. Never log unstructured text.

```javascript
const winston = require("winston");

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || "info",
  format: winston.format.json(), // Crucial: Output JSON
  defaultMeta: {
    service: "api",
    environment: process.env.NODE_ENV,
  },
  transports: [new winston.transports.Console()],
});

// Example Usage
logger.error("Database connection failed", {
  error_code: err.code,
  retries: 3,
});
```

### 2. Safe Promtail Configuration

Map broad Kubernetes metadata to labels. Do not map highly variable data (like `error_code` or `user_id`) to labels.

```yaml
clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # Map K8s namespace to Loki label
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: namespace
      # Map K8s app label to Loki label
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: replace
        target_label: app
    pipeline_stages:
      # Parse the incoming JSON log line
      - json:
          expressions:
            level: level
            msg: message
      # ONLY promote safe, low-cardinality fields to labels
      - labels:
          level:
```

### 3. LogQL Query Examples

Querying in Grafana follows a strict two-step process: Filter by label first, then parse text/JSON.

```logql
# 1. Broad Label Filter (Fast) -> 2. Text Search (Slower)
{namespace="production", app="api"} |= "Database connection failed"

# 1. Broad Label Filter -> 2. JSON Parser -> 3. Condition
{namespace="production", app="api"} | json | duration > 1000

# Metric Extraction: Error rate per minute by application
sum(rate({namespace="production", level="error"}[1m])) by (app)
```
