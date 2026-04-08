# The Tiered Observability Architecture

Achieve massive cost reduction by decoupling log ingestion from expensive hot storage via tiered routing.

Logging 100% of `INFO` and `DEBUG` events to a hot index (like Elasticsearch or Datadog) at terabyte scale is an architectural anti-pattern. 90% of those logs are never read, yet they consume massive compute and storage budgets.

The modern observability stack decouples data ingestion from hot storage using three core strategies:

## 1. Dynamic Log Levels & Smart Sampling

- **Production Default:** Hardcode the default log level to `WARN`.
- **Smart Sampling:** Drop 95% of routine `INFO` logs (e.g., "request received"), keeping only a small statistical baseline.
- **On-Demand Debugging:** Expose an internal API to temporarily flip specific services or user sessions to `DEBUG` level for **a strict time window** during an incident.

## 2. Distributed Tracing Over Verbose Logging

Replace procedural "step-by-step" log lines with OpenTelemetry distributed tracing. Traces provide parent-child execution context natively, rendering dozens of `DEBUG` lines obsolete.

## 3. Tiered Storage Routing

Do not send all logs to the same database. Route them at the infrastructure edge (using Fluentd or Vector) based on severity:

| Tier     | Storage                 | Retention                  | Routing Rules                                                |
| :------- | :---------------------- | :------------------------- | :----------------------------------------------------------- |
| **Hot**  | Elasticsearch / Datadog | **Short-term**             | `ERROR`, `WARN`, and explicitly tagged business events.      |
| **Warm** | S3 + AWS Athena         | **Medium-term**            | Sampled `INFO` logs, HTTP access logs.                       |
| **Cold** | S3 Glacier (Compressed) | **Long-term (Compliance)** | 100% of raw logs (gzip 10:1 ratio) for audit and compliance. |
