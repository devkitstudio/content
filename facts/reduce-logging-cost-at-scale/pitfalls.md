## The Dangers of Aggressive Sampling

### 1. Dropping the "Needle in the Haystack"

If you implement a blind `Math.random() < 0.05` sampling rate across all `INFO` logs, you will inevitably drop logs related to successful payments, password resets, or account deletions. When a customer complains about a missing order, Customer Support will have zero record of the transaction.
**Fix:** Always maintain a strict `CRITICAL_EVENTS` allowlist that bypasses all sampling logic.

### 2. The Orphaned Trace (Missing Correlation IDs)

When an error occurs in production, you will only have the `ERROR` log in your hot storage. To find out what led to the error, you must query your S3 Cold Storage or Tracing backend. If your logs do not share a unified `trace_id` with your OpenTelemetry spans, correlating the error back to the user's journey is impossible.
**Fix:** Ensure a middleware injects an `x-trace-id` into both the logger context and the OpenTelemetry span the moment a request hits the API Gateway.
