## Real Numbers: The $84,000/Year Savings

Transitioning from a "Log Everything" approach to a tiered architecture with smart sampling produces immediate, drastic cost reductions.

### Before Optimization (1 TB/day)

Ingesting and indexing 100% of logs in hot storage (e.g., Datadog or self-hosted Elasticsearch) is financially unsustainable at scale.

- **Volume:** 1 TB/day = 30 TB/month
- **Ingestion Cost:** $0.10/GB = $3,000/month
- **Indexing Cost:** ~$1.70/Million events = ~$5,000/month
- **Total Legacy Cost: ~$8,000/month**

### After Optimization (95% Volume Reduction)

By implementing dynamic log levels (`WARN+`), sampling `INFO` logs at 2%, and replacing verbose procedural logs with distributed traces, the volume drops exponentially.

- **New Hot Volume:** 50 GB/day = 1.5 TB/month
- **Hot Storage Cost:** ~$400/month
- **Tracing Infrastructure (OpenTelemetry):** ~$600/month
- **Cold Storage Archive (S3):** 30 TB raw data compressed at 10:1 ratio (3 TB) = $69/month
- **Total Optimized Cost: ~$1,069/month**

**Bottom Line: You save ~$7,000 per month (~$84,000 per year) while retaining exact traceability for every single request.**
