## Execution Pipeline

### 1. Smart Sampling with Critical Exemptions (Node.js)

Drop routine logs but guarantee 100% capture of business-critical events.

```typescript
const CRITICAL_EVENTS = new Set(["order.paid", "payment.failed", "auth.login"]);

function logInfo(event: string, data: Record<string, any>, sampleRate = 0.05) {
  const isCritical = CRITICAL_EVENTS.has(event);

  // Always log critical events. For others, use random sampling.
  if (isCritical || Math.random() < sampleRate) {
    logger.info({
      event,
      ...data,
      _critical: isCritical, // Flag for log router
    });
  }
}
```

### 2. Emergency Debug Toggle

Expose an endpoint to dynamically change log levels without restarting pods.

```bash
# Temporarily enable DEBUG for the payment service for 15 minutes
curl -X POST https://internal-api/config/logging \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{"service": "payment", "level": "debug", "ttl_minutes": 15}'
```

### 3. Log Routing (Fluentd Example)

Route critical data to hot storage, and everything else to cheap S3 archives.

```yaml
<match app.**>
  @type copy

  # HOT TIER: Only send Errors, Warns, or Critical flags to Elasticsearch
  <store>
    @type elasticsearch
    <filter>
      # Regex match on log level or the _critical flag injected by the app
      grep /"level":"(?:error|warn)"|"_critical":true/
    </filter>
  </store>

  # COLD TIER: Send 100% of logs to S3, heavily compressed
  <store>
    @type s3
    s3_bucket company-logs-archive
    path "raw/%Y/%m/%d/"
    <buffer>
      flush_interval 60s
      compress gzip
    </buffer>
  </store>
</match>
```
