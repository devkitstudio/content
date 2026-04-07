## Additional Rate Limiting Algorithms

### 4. Leaky Bucket

Like a bucket with a hole at the bottom — requests flow in at any rate but are processed at a **fixed rate**. Excess requests queue up (or are dropped if the bucket overflows).

```
Incoming: [burst of 50 requests]
         ↓
    ┌─────────────┐
    │  Queue (20)  │ ← bucket capacity
    └──────┬──────┘
           │  leak rate: 10 req/s
           ▼
    [processed at constant rate]
```

```typescript
class LeakyBucket {
  private queue: Array<() => void> = [];
  private processing = false;

  constructor(
    private capacity: number,    // max queue size
    private leakRateMs: number,  // ms between processing each request
  ) {}

  async add(request: () => Promise<void>): Promise<boolean> {
    if (this.queue.length >= this.capacity) {
      return false; // Bucket overflow → reject
    }

    this.queue.push(request);
    this.drain();
    return true; // Queued
  }

  private async drain() {
    if (this.processing) return;
    this.processing = true;

    while (this.queue.length > 0) {
      const next = this.queue.shift()!;
      await next();
      await new Promise(r => setTimeout(r, this.leakRateMs));
    }

    this.processing = false;
  }
}

// Usage: process 10 req/s, queue up to 50
const bucket = new LeakyBucket(50, 100); // 100ms = 10/s
```

**Key difference from Token Bucket**: Token Bucket allows bursts (spend all tokens at once). Leaky Bucket **smooths output** to a constant rate. Token Bucket is better for bursty APIs; Leaky Bucket is better for protecting downstream services.

### 5. Sliding Window Counter (Hybrid)

Combines Fixed Window and Sliding Window Log — approximates sliding window accuracy with fixed window memory efficiency:

```typescript
function slidingWindowCounter(
  userId: string,
  limit: number,
  windowSec: number,
): boolean {
  const now = Date.now();
  const windowMs = windowSec * 1000;
  const currentWindow = Math.floor(now / windowMs);
  const previousWindow = currentWindow - 1;

  const currentCount = getCount(userId, currentWindow);
  const previousCount = getCount(userId, previousWindow);

  // Weight the previous window by how much time has elapsed in current window
  const elapsed = (now % windowMs) / windowMs;
  const weightedCount = previousCount * (1 - elapsed) + currentCount;

  return weightedCount < limit;
}

// Example: limit = 100/min
// At 30 seconds into current window:
// Previous window had 80 requests
// Current window has 30 requests
// Weighted = 80 * 0.5 + 30 = 70 → allowed (< 100)
```

**Used by**: Cloudflare, Redis-based rate limiters. Best balance of accuracy and memory.

### 6. Concurrency Limiter

Instead of requests-per-time-window, limit **simultaneous** active requests:

```typescript
class ConcurrencyLimiter {
  private active = 0;

  constructor(private maxConcurrent: number) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.active >= this.maxConcurrent) {
      throw new Error('Too many concurrent requests');
    }

    this.active++;
    try {
      return await fn();
    } finally {
      this.active--;
    }
  }
}

// Max 10 concurrent database queries
const dbLimiter = new ConcurrencyLimiter(10);
const result = await dbLimiter.execute(() => db.query(heavySQL));
```

**Best for**: Protecting database connection pools, external API concurrent call limits.

## Complete Algorithm Comparison

| Algorithm | Memory | Accuracy | Burst Handling | Complexity | Best For |
|-----------|--------|----------|---------------|------------|----------|
| **Fixed Window** | O(1) | Low (boundary burst) | Poor | Simple | Internal APIs, prototyping |
| **Sliding Window Log** | O(N) | Perfect | Excellent | Medium | Small-scale, exact limits |
| **Sliding Window Counter** | O(1) | High (~99.7%) | Good | Medium | Production APIs (Cloudflare) |
| **Token Bucket** | O(1) | High | Allows bursts | Medium | Public APIs, bursty traffic |
| **Leaky Bucket** | O(N) queue | High | Smooths output | Medium | Protecting downstream |
| **Concurrency** | O(1) | N/A | Limits parallel | Simple | DB pools, external APIs |
