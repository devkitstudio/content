## Why 1TB/Day Happens

Most teams log everything at DEBUG/INFO level in production because "we might need it." The reality:

```
Typical log breakdown:
  DEBUG:  40%  ← "entering function X", "cache hit for key Y"
  INFO:   35%  ← "request received", "response sent", health checks
  WARN:   15%  ← retries, slow queries, deprecation warnings
  ERROR:  10%  ← actual failures you need to investigate
```

**90% of logs are never read by anyone.** But you pay to ingest, index, and store every byte.

## Strategy 1: Log Levels + Dynamic Sampling

### Set Production Default to WARN

```typescript
// In production, only WARN and above are shipped
const LOG_LEVEL = process.env.NODE_ENV === 'production' ? 'warn' : 'debug';

logger.debug('Cache hit for user 123');  // ← dropped in production
logger.info('Request received');          // ← dropped in production
logger.warn('Slow query: 2.3s');          // ✓ shipped
logger.error('Payment failed', { err });  // ✓ shipped
```

### Dynamic Log Level (Turn on DEBUG When Needed)

Don't lose the ability to debug. Use a feature flag or config endpoint to temporarily enable verbose logging for specific services or users:

```typescript
// Check dynamic config every 30s
const dynamicLevel = await configService.get('log_level'); // 'debug' | 'warn'
logger.level = dynamicLevel;

// Or per-request: enable DEBUG for a specific user/trace
if (request.headers['x-debug-token'] === SECRET_DEBUG_TOKEN) {
  requestLogger.level = 'debug';
}
```

**Impact**: Drops 75% of log volume immediately.

## Strategy 2: Structured Logging + Smart Sampling

### Sample INFO Logs Instead of Dropping Them All

Keep 1-5% of INFO logs for baseline visibility:

```typescript
function shouldLog(level: string, sampleRate: number): boolean {
  if (level === 'error' || level === 'warn') return true; // always log
  return Math.random() < sampleRate;
}

// Log 2% of INFO messages
if (shouldLog('info', 0.02)) {
  logger.info('Request processed', { duration: 45, path: '/api/orders' });
}
```

### Always Log Business-Critical Flows at 100%

Some flows must NEVER be sampled — payment, auth, order state changes:

```typescript
const CRITICAL_EVENTS = new Set([
  'order.created', 'order.paid', 'order.refunded',
  'payment.charged', 'payment.failed',
  'auth.login', 'auth.failed', 'auth.token_revoked',
]);

function businessLog(event: string, data: Record<string, unknown>) {
  // Always log, regardless of sampling
  logger.info({ event, ...data, _critical: true });
}
```

## Strategy 3: Replace Verbose Logs with Traces

Instead of logging every step of a request, use **distributed tracing** (OpenTelemetry). One trace replaces 20+ log lines:

```
// BEFORE: 22 log lines for one order
[INFO] POST /api/orders received
[INFO] Validating order payload
[INFO] User 456 authenticated
[DEBUG] Checking inventory for SKU-789
[DEBUG] Inventory query: 23ms
[INFO] Stock available: 5 units
[DEBUG] Calculating shipping...
[INFO] Shipping: $5.99
[INFO] Creating payment intent
[DEBUG] Stripe API call: 340ms
[INFO] Payment authorized: pi_xxx
[INFO] Order created: ORD-123
... (10 more lines)

// AFTER: 1 trace with spans
Trace: ORD-123 (total: 580ms)
├─ validate (12ms)
├─ check_inventory (23ms) → stock: 5
├─ calculate_shipping (8ms) → $5.99
├─ charge_payment (340ms) → pi_xxx ✓
└─ create_order (45ms) → ORD-123 ✓
```

**Same traceability, 95% less storage.**

## Strategy 4: Tiered Storage

Not all logs need to be in hot storage (Elasticsearch). Use a tiered approach:

| Tier | Storage | Retention | What Goes Here |
|------|---------|-----------|----------------|
| **Hot** | Elasticsearch/Datadog | 3-7 days | ERROR, WARN, critical business events |
| **Warm** | S3 + Athena | 30 days | Sampled INFO, access logs |
| **Cold** | S3 Glacier | 1 year | Compressed raw logs for compliance |

```yaml
# Fluentd/Vector config: route by severity
<match app.**>
  @type copy

  # Hot: errors and business events → Elasticsearch
  <store>
    @type elasticsearch
    <buffer>
      flush_interval 5s
    </buffer>
    <filter>
      grep level /error|warn/
      # OR: grep _critical true
    </filter>
  </store>

  # Cold: everything → S3 compressed
  <store>
    @type s3
    s3_bucket logs-archive
    path "raw/%Y/%m/%d/"
    <buffer>
      flush_interval 60s
      compress gzip     # 10:1 compression ratio
    </buffer>
  </store>
</match>
```
