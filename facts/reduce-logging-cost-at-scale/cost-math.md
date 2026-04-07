## Real Numbers: Before vs After

### Before Optimization

```
Volume:    1 TB/day = 30 TB/month
Datadog:   $0.10/GB ingested = $3,000/month ingestion
           + $1.70/M events indexed = ~$5,000/month
           Total: ~$8,000/month

Elasticsearch (self-hosted):
           3x replication × 30 TB = 90 TB storage
           Hot nodes: 6× i3.2xlarge = ~$6,000/month
           Total: ~$6,000/month + ops time
```

### After Optimization

| Strategy | Volume Reduction |
|----------|-----------------|
| Log level WARN+ only | -75% |
| Sample INFO at 2% | -15% more |
| Replace verbose logs with traces | -5% more |
| **Total** | **~95% reduction** |

```
New volume: 50 GB/day = 1.5 TB/month

Datadog: ~$400/month (vs $8,000)
  + OpenTelemetry traces: ~$600/month
  Total: ~$1,000/month

S3 cold archive (full raw logs):
  30 TB × gzip 10:1 = 3 TB × $0.023/GB = $69/month
```

**Monthly savings: $7,000 ($84,000/year)**

## The Traceability Question

"But how do we debug issues without all those logs?"

### Answer: Correlation IDs + On-Demand Debug

Every request gets a `trace_id`. When an error occurs, you have:

1. **The error log** (always captured) with `trace_id`
2. **The trace** (OpenTelemetry) showing the full request flow
3. **Cold logs** (S3) searchable via Athena if you need raw DEBUG lines

```typescript
// Middleware: inject trace_id into every request
app.use((req, res, next) => {
  req.traceId = req.headers['x-trace-id'] || crypto.randomUUID();
  // Attach to all logs automatically
  req.logger = logger.child({ traceId: req.traceId });
  next();
});

// When debugging: query cold storage by trace_id
// AWS Athena query on S3 raw logs:
// SELECT * FROM logs WHERE trace_id = 'abc-123' ORDER BY timestamp
```

### Emergency: Turn On Full Logging Temporarily

```bash
# Via feature flag or config API
curl -X POST https://internal-api/config \
  -d '{"log_level": "debug", "service": "payment", "ttl": "15m"}'

# Automatically reverts to WARN after 15 minutes
```

You get full DEBUG output for 15 minutes to catch the bug, then costs drop back to normal.
