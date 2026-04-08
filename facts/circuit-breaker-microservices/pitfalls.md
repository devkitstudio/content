## The Traps of Circuit Breaking

### 1. The "Thundering Herd" on Recovery

When a Circuit Breaker enters the `HALF_OPEN` state, it allows test requests through. If the service recovers and the breaker closes, pending retries from thousands of clients might flood the service simultaneously, instantly killing it again.
**Fix:** Always implement **Jitter (randomized delays)** in your client-side Retry mechanisms to spread out the load when the circuit closes.

### 2. Ignoring Latency Degradation

Many engineers configure breakers to trip only on HTTP 5xx errors. However, a downstream service hanging for 30 seconds (without returning an error) will exhaust your application's connection pool/threads just as quickly as a hard crash.
**Fix:** Always configure the breaker to trip on **Timeouts** or **P99 Latency spikes**, not just raw error counts. Set aggressive timeouts (e.g., 2 seconds max).

### 3. The "Silent Array" Fallback Trap

Providing a fallback value is dangerous for write operations or state-mutating requests. If your `UserService.getRoles()` fails, and your fallback returns an empty array `[]`, the system might interpret this as "The user has no roles" and revoke their admin access, rather than realizing the service is simply down.
**Fix:** Only use default value fallbacks for non-critical, read-only data (like recommendations). For critical data, it is safer to fail the request with a `503 Service Unavailable` than to return incorrect mock data.
