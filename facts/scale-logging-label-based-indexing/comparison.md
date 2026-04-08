## Architectural Trade-offs: Full-Text vs. Label-Based Indexing

Logging at terabyte scale introduces massive storage and compute costs. Choosing the right index strategy is an architectural necessity.

| Feature               | Full-Text Indexing (e.g., Elasticsearch/Splunk)            | Label-Based Indexing (e.g., Grafana Loki)                  |
| :-------------------- | :--------------------------------------------------------- | :--------------------------------------------------------- |
| **Indexing Strategy** | Indexes full log text (Compute & Storage Heavy)            | Indexes labels only (Extremely Lightweight)                |
| **Storage Engine**    | Fast disk (SSD mandatory for performance)                  | Object storage (S3, GCS) for log chunks                    |
| **Query Power**       | Powerful Lucene/Regex full-text search across all fields   | LogQL (Requires static label filtering before parsing)     |
| **Cost Ratio**        | 100% Baseline (High Compute/SSD dependencies)              | ~2-5% of Baseline (Leverages cheap object storage)         |
| **Best For**          | Complex log mining, un-structured data, business analytics | High-volume structured application logs, metrics, alerting |
