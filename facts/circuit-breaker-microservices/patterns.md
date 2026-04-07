## Patterns That Work With Circuit Breaker

### 1. Bulkhead Pattern
Isolate resources per service so one failing dependency can't consume all threads.

```typescript
// Separate thread pools per dependency
const paymentPool = new ThreadPool({ max: 20 });
const fraudPool = new ThreadPool({ max: 10 });
const emailPool = new ThreadPool({ max: 5 });

// Fraud service crashing only exhausts its 10 threads
// Payment and Email pools remain unaffected
```

### 2. Retry with Exponential Backoff
Before the circuit trips, retry with increasing delays.

```typescript
async function retryWithBackoff<T>(fn: () => Promise<T>, maxRetries = 3): Promise<T> {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (err) {
      if (i === maxRetries - 1) throw err;
      const delay = Math.min(1000 * Math.pow(2, i) + Math.random() * 500, 10000);
      await new Promise(r => setTimeout(r, delay));
    }
  }
  throw new Error('Exhausted retries');
}
```

**Important**: Add **jitter** (random delay) to prevent a "thundering herd" when the service recovers.

### 3. Timeout + Circuit Breaker Combo
Always set aggressive timeouts. Don't wait 30 seconds for a service that normally responds in 200ms.

```typescript
const result = await fraudBreaker.call(
  () => withTimeout(fraudService.check(order), 2000), // 2s max
  () => ({ safe: true })
);
```

### 4. Fallback Strategies

| Strategy | When to Use |
|----------|------------|
| **Default value** | Non-critical data (recommendations, analytics) |
| **Cached response** | Data that doesn't change often (product catalog) |
| **Queue for later** | Operations that can be async (email, notifications) |
| **Graceful degradation** | Show partial page without the failing section |
| **Manual review flag** | Critical operations (fraud check → approve but flag) |
