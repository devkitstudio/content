# Centralized Logging Architecture

## 1. ELK Stack vs Loki Comparison

**ELK (Elasticsearch, Logstash, Kibana):**
- Indexes full log text (expensive storage)
- Powerful queries with Lucene syntax
- Best for: Debugging, full-text search
- Cost: High (storage + compute)
- Scaling: 50GB+ logs/day = challenge

**Loki (Grafana Labs):**
- Indexes labels only (cheap storage)
- Logs stored compressed
- Best for: Monitoring, alerting, high volume
- Cost: Low (only metadata indexed)
- Scaling: 1TB+ logs/day = easy

## 2. Loki Architecture

**Three components:**

```
Promtail (log collector)
       ↓
Loki (log aggregation)
       ↓
Grafana (visualization/alerting)
```

**How Loki works:**
1. Promtail reads logs from container stdout
2. Adds labels (app, environment, pod, etc.)
3. Ships to Loki
4. Loki indexes only labels, stores raw logs compressed
5. Grafana queries Loki for visualization

## 3. Loki Installation (Kubernetes)

**Install with Helm:**

```bash
# Add Grafana repo
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Loki
helm install loki grafana/loki-stack \
  --namespace loki --create-namespace \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=50Gi \
  --set promtail.enabled=true \
  --set grafana.enabled=true
```

**Verify installation:**

```bash
kubectl get pods -n loki
# loki-0
# loki-promtail-xxxx (on each node)
# loki-grafana-xxxx
```

## 4. Loki Configuration

**Minimal loki-config.yaml:**

```yaml
auth_enabled: false

ingester:
  chunk_idle_period: 3m
  chunk_retain_period: 1m
  max_chunk_age: 1h
  chunk_encoding: snappy

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/boltdb-shipper-active
    cache_location: /loki/boltdb-shipper-cache
    shared_store: filesystem
  filesystem:
    directory: /loki/chunks

schema_config:
  configs:
  - from: 2024-01-01
    store: boltdb_shipper
    object_store: filesystem
    schema: v12
    index:
      prefix: index_
      period: 24h

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  ingestion_rate_mb: 100
  ingestion_burst_size_mb: 200
```

## 5. Promtail Configuration

**Ship logs from Docker/Kubernetes:**

```yaml
clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
- job_name: kubernetes-pods
  kubernetes_sd_configs:
  - role: pod
  relabel_configs:
  - source_labels: [__meta_kubernetes_pod_name]
    action: replace
    target_label: pod
  - source_labels: [__meta_kubernetes_namespace]
    action: replace
    target_label: namespace
  - source_labels: [__meta_kubernetes_pod_label_app]
    action: replace
    target_label: app
  - source_labels: [__meta_kubernetes_pod_container_name]
    action: replace
    target_label: container
  pipeline_stages:
  - json:
      expressions:
        level: level
        msg: message
        error: error
  - labels:
      level:
      msg:
```

## 6. Application Logging Best Practices

**Structured logging (JSON):**

```javascript
// Good: Structured logs
const logger = (req, res, next) => {
  console.log(JSON.stringify({
    timestamp: new Date().toISOString(),
    level: 'info',
    message: 'Request received',
    method: req.method,
    path: req.path,
    ip: req.ip,
    duration: res.responseTime,
  }));
};

// Bad: Unstructured logs
console.log(`Request: ${req.method} ${req.path}`);
```

**Log levels properly:**

```javascript
const logger = require('winston');

// Setup Winston with JSON output
const winston = logger.createLogger({
  format: logger.format.json(),
  transports: [
    new logger.transports.Console(),
  ],
});

// Use different levels
winston.error('Critical error', { error: err.message, stack: err.stack });
winston.warn('Deprecated endpoint used', { endpoint: '/api/v1/old' });
winston.info('User logged in', { userId: user.id, email: user.email });
winston.debug('Cache hit', { key: cacheKey, ttl: 3600 });
```

## 7. Querying Logs in Grafana

**Basic Loki query syntax:**

```logql
# All logs from app=api namespace=production
{app="api", namespace="production"}

# With level filtering
{app="api"} | json | level="error"

# Pattern matching
{app="api"} |= "error" |= "database"

# Regex patterns
{app="api"} | regexp "user_id=(?P<user_id>\\d+)"

# Metrics from logs
rate({app="api", level="error"}[5m])
```

**Grafana dashboard queries:**

```
# Error rate per minute
sum(rate({app="api", level="error"}[1m])) by (namespace)

# Top 10 error messages
topk(10, sum by (message) (count_over_time({level="error"}[5m])))

# Latency percentiles
quantile_over_time(0.95, {job="api"} | duration > 1000 [5m])
```

## 8. Alerting on Logs

**Alert on error rate:**

```yaml
groups:
- name: application-alerts
  rules:
  - alert: HighErrorRate
    expr: sum(rate({app="api", level="error"}[5m])) > 10
    for: 5m
    annotations:
      summary: "High error rate detected"
      description: "Error rate is {{ $value }} errors/sec"

  - alert: OutOfMemory
    expr: count_over_time({app="api"} |= "OOMKilled" [5m]) > 0
    annotations:
      summary: "Pod OOMKilled"
      description: "Pod {{ $labels.pod }} was killed due to memory"
```

## 9. ELK Stack Alternative

**If you prefer Elasticsearch:**

```yaml
# Elasticsearch (stores all logs)
# Logstash (processes logs)
# Kibana (visualization)

# Deployment with Docker Compose
version: '3.8'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.5.0
    environment:
      discovery.type: single-node
      ELASTIC_PASSWORD: changeme
    volumes:
      - es-data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  logstash:
    image: docker.elastic.co/logstash/logstash:8.5.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/config/logstash.conf
    ports:
      - "5000:5000"

  kibana:
    image: docker.elastic.co/kibana/kibana:8.5.0
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_URL: http://elasticsearch:9200

volumes:
  es-data:
```

## 10. Cost Comparison

| Scenario | Loki | ELK |
|----------|------|-----|
| 100GB/day | $100/month | $5,000/month |
| 1TB/day | $1,000/month | $50,000/month |
| 10TB/day | $10,000/month | $500,000+/month |

**Why Loki is cheaper:**
- Only indexes labels, not log text
- Compresses logs (1:10 ratio)
- Scales horizontally
- Uses object storage (S3)

