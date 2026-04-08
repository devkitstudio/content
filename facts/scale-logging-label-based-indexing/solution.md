# Loki Architecture & Core Configuration

Centralized logging at scale requires separating the indexing mechanism from the log storage. Loki achieves massive cost reduction by acting like Prometheus for logs: it indexes metadata (labels) but leaves the raw log text unindexed and compressed in object storage.

## 1. The Architecture

The log pipeline consists of three decoupled components:

- **Promtail (Agent):** Deployed as a DaemonSet. Reads container stdout, attaches Kubernetes metadata (pod, namespace) as labels, and streams to Loki.
- **Loki (Aggregator):** Receives streams, builds an index _only_ from the labels, and chunks the compressed JSON logs into S3/GCS.
- **Grafana (Visualization):** Queries the label index to locate the correct chunks, then dynamically parses the JSON at query time.

## 2. Minimal Production Configuration

A standard Loki configuration strictly separating the fast index (BoltDB) from cheap long-term storage (S3):

```yaml
auth_enabled: false

ingester:
  chunk_idle_period: 3m
  chunk_retain_period: 1m
  max_chunk_age: 1h
  chunk_encoding: snappy

schema_config:
  configs:
    - from: 2024-01-01
      store: boltdb_shipper
      object_store: s3
      schema: v12
      index:
        prefix: index_
        period: 24h

storage_config:
  s3:
    s3: s3://aws-region/loki-logs-bucket
    access_key_id: ${AWS_ACCESS_KEY_ID}
    secret_access_key: ${AWS_SECRET_ACCESS_KEY}
  boltdb_shipper:
    active_index_directory: /loki/boltdb-shipper-active
    cache_location: /loki/boltdb-shipper-cache
    shared_store: s3
```
