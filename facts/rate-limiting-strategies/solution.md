## The Three Main Algorithms

### 1. Fixed Window Counter
- Divide time into fixed intervals (e.g., every 60 seconds)
- Count requests in the current window
- Reset counter when the window expires

```
Window: [00:00 - 01:00] → 98/100 requests used
Window: [01:00 - 02:00] → counter resets to 0
```

**Problem**: Burst at window boundary. A user sends 100 requests at `00:59` and 100 more at `01:01` → 200 requests in 2 seconds.

### 2. Sliding Window Log
- Store timestamp of every request
- On each new request, count timestamps within the last N seconds
- Most accurate, but **memory-expensive** (stores every timestamp)

```
Requests: [00:45, 00:47, 00:59, 01:01, 01:02]
At 01:02, look back 60s → count requests since 00:02 → 5 requests
```

### 3. Token Bucket
- A "bucket" holds tokens (max capacity = burst limit)
- Tokens are added at a fixed rate (e.g., 10 tokens/second)
- Each request consumes 1 token. No tokens = rejected.

```
Bucket capacity: 100 tokens
Refill rate: 10 tokens/second
Burst: can handle 100 requests instantly
Sustained: 10 req/s after burst
```

**Best for**: APIs that need to allow occasional bursts but enforce a sustained rate.

## Which Algorithm for Which API?

| API Type | Recommended | Why |
|----------|------------|-----|
| **Payment API** | Token Bucket with low burst | Allow small retries, but strictly limit sustained rate to prevent abuse |
| **Search API** | Sliding Window | Smooth rate enforcement, no boundary bursts, good UX |
| **Login API** | Fixed Window + exponential backoff | Simple, plus progressive penalties on failure |
| **Webhook receiver** | Token Bucket with high burst | Partners may send batches, need burst tolerance |

## Production Implementation

In practice, rate limiting is done at the **edge layer** (API Gateway, Cloudflare, Nginx) rather than in application code:

```nginx
# Nginx: Token Bucket via limit_req
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

server {
    location /api/ {
        limit_req zone=api burst=20 nodelay;
        # burst=20 → token bucket capacity
        # rate=10r/s → refill rate
    }
}
```

```typescript
// Cloudflare Workers: Sliding Window with KV
async function rateLimit(ip: string, limit: number, windowSec: number): Promise<boolean> {
  const key = `rl:${ip}`;
  const now = Date.now();
  const data = await KV.get(key, 'json') as number[] || [];

  // Remove expired timestamps
  const valid = data.filter(ts => now - ts < windowSec * 1000);

  if (valid.length >= limit) return false; // Rejected

  valid.push(now);
  await KV.put(key, JSON.stringify(valid), { expirationTtl: windowSec });
  return true; // Allowed
}
```
