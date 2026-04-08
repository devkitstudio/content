## Full-Text Indexing (ELK) vs. Label-Based Indexing (Loki)

Logging at terabyte scale introduces massive storage and compute costs. Choosing the right index strategy is an architectural necessity.

| Feature               | Elasticsearch (ELK)                    | Grafana Loki                                    |
| :-------------------- | :------------------------------------- | :---------------------------------------------- |
| **Indexing Strategy** | Indexes full log text (Heavy)          | Indexes labels only (Extremely Lightweight)     |
| **Storage Engine**    | Fast disk (SSD recommended)            | Object storage (S3, GCS) for old chunks         |
| **Query Power**       | Powerful Lucene full-text search       | LogQL (Requires label filtering first)          |
| **Cost at 1TB/day**   | ~$50,000/month                         | ~$1,000/month                                   |
| **Best For**          | Complex log mining, business analytics | High-volume application logs, metrics, alerting |
