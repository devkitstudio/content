## Distributed Rate Limiting with Redis

### Atomic Token Bucket in Redis (Lua Script)

The most production-proven approach — a single Lua script runs atomically in Redis:

```lua
-- token_bucket.lua
local key = KEYS[1]
local capacity = tonumber(ARGV[1])    -- max tokens
local refillRate = tonumber(ARGV[2])  -- tokens per second
local now = tonumber(ARGV[3])         -- current timestamp (ms)
local requested = tonumber(ARGV[4])   -- tokens to consume (usually 1)

local data = redis.call('HMGET', key, 'tokens', 'lastRefill')
local tokens = tonumber(data[1]) or capacity
local lastRefill = tonumber(data[2]) or now

-- Refill tokens based on elapsed time
local elapsed = (now - lastRefill) / 1000
local newTokens = math.min(capacity, tokens + elapsed * refillRate)

if newTokens >= requested then
  newTokens = newTokens - requested
  redis.call('HMSET', key, 'tokens', newTokens, 'lastRefill', now)
  redis.call('PEXPIRE', key, math.ceil(capacity / refillRate) * 1000 + 1000)
  return {1, math.floor(newTokens)} -- allowed, remaining
else
  redis.call('HMSET', key, 'tokens', newTokens, 'lastRefill', now)
  redis.call('PEXPIRE', key, math.ceil(capacity / refillRate) * 1000 + 1000)
  return {0, math.floor(newTokens)} -- rejected, remaining
end
```

```typescript
import Redis from 'ioredis';
import { readFileSync } from 'fs';

const redis = new Redis();
const luaScript = readFileSync('./token_bucket.lua', 'utf-8');

// Define the command once
redis.defineCommand('tokenBucket', {
  numberOfKeys: 1,
  lua: luaScript,
});

async function rateLimit(
  identifier: string,
  capacity: number,
  refillRate: number,
): Promise<{ allowed: boolean; remaining: number }> {
  const [allowed, remaining] = await (redis as any).tokenBucket(
    `rl:${identifier}`,
    capacity,
    refillRate,
    Date.now(),
    1,
  );
  return { allowed: !!allowed, remaining };
}

// Middleware
app.use(async (req, res, next) => {
  const key = `${req.ip}:${req.path}`;
  const { allowed, remaining } = await rateLimit(key, 100, 10);

  res.set('X-RateLimit-Limit', '100');
  res.set('X-RateLimit-Remaining', String(remaining));

  if (!allowed) {
    res.set('Retry-After', '10');
    return res.status(429).json({ error: 'Rate limit exceeded' });
  }
  next();
});
```

### Multi-Region Rate Limiting

For globally distributed services, a single Redis instance creates latency:

```typescript
// Strategy 1: Local counters with async sync
// Each region has its own Redis, sync totals periodically
class RegionalRateLimiter {
  constructor(
    private localRedis: Redis,
    private globalRedis: Redis,
    private region: string,
  ) {}

  async check(key: string, globalLimit: number): Promise<boolean> {
    // Local limit = global limit / number of regions (approximate)
    const localLimit = Math.ceil(globalLimit / this.regionCount);
    const localCount = await this.localRedis.incr(`local:${key}`);

    if (localCount === 1) {
      await this.localRedis.expire(`local:${key}`, 60);
    }

    // Async sync to global counter every 10 requests
    if (localCount % 10 === 0) {
      await this.globalRedis.incrby(`global:${key}`, 10);
    }

    return localCount <= localLimit;
  }
}

// Strategy 2: Token bucket with regional allocation
// Central coordinator distributes token budgets to regions
// Each region operates independently within its allocation
```

## Adaptive Rate Limiting

### Dynamic Limits Based on Server Health

Instead of fixed limits, adjust based on real-time metrics:

```typescript
class AdaptiveRateLimiter {
  private currentLimit: number;
  private readonly minLimit: number;
  private readonly maxLimit: number;

  constructor(config: {
    baseLimit: number;
    minLimit: number;
    maxLimit: number;
  }) {
    this.currentLimit = config.baseLimit;
    this.minLimit = config.minLimit;
    this.maxLimit = config.maxLimit;
  }

  // Called periodically (e.g., every 10 seconds)
  adjustLimit(metrics: {
    cpuUsage: number;        // 0-100
    memoryUsage: number;     // 0-100
    p99Latency: number;      // ms
    errorRate: number;        // 0-1
    activeConnections: number;
  }) {
    const healthScore = this.calculateHealth(metrics);

    if (healthScore > 0.8) {
      // System healthy → gradually increase limit
      this.currentLimit = Math.min(
        this.maxLimit,
        Math.floor(this.currentLimit * 1.1)
      );
    } else if (healthScore < 0.5) {
      // System stressed → aggressively reduce limit
      this.currentLimit = Math.max(
        this.minLimit,
        Math.floor(this.currentLimit * 0.7)
      );
    }
    // Between 0.5-0.8: maintain current limit
  }

  private calculateHealth(m: any): number {
    const cpuScore = 1 - m.cpuUsage / 100;
    const memScore = 1 - m.memoryUsage / 100;
    const latencyScore = Math.max(0, 1 - m.p99Latency / 5000);
    const errorScore = 1 - m.errorRate;

    return cpuScore * 0.3 + memScore * 0.2 + latencyScore * 0.3 + errorScore * 0.2;
  }

  getLimit(): number {
    return this.currentLimit;
  }
}
```

### Priority-Based Rate Limiting

Different clients get different treatment:

```typescript
const TIER_LIMITS = {
  free:       { rate: 10,   burst: 20,   priority: 0 },
  pro:        { rate: 100,  burst: 200,  priority: 1 },
  enterprise: { rate: 1000, burst: 5000, priority: 2 },
  internal:   { rate: 5000, burst: 10000, priority: 3 },
};

async function priorityRateLimit(req: Request): Promise<boolean> {
  const tier = await getTier(req.apiKey);
  const limits = TIER_LIMITS[tier];

  // Under system stress, lower-priority tiers get limited first
  const serverLoad = await getServerLoad(); // 0-1
  const adjustedRate = serverLoad > 0.8
    ? limits.rate * (0.5 + limits.priority * 0.15) // reduce, but less for higher tiers
    : limits.rate;

  return tokenBucket(req.apiKey, adjustedRate, limits.burst);
}
```

### Interview Tips: What Interviewers Want to Hear

When asked about rate limiting, mention these in order:

1. **Algorithm choice** — Token Bucket for public APIs, Sliding Window for internal
2. **Distributed state** — Redis with Lua scripts for atomicity
3. **Proper HTTP headers** — 429 status, Retry-After, X-RateLimit-*
4. **Granular limits** — per-endpoint, per-tier, read vs write
5. **Edge enforcement** — Nginx, API Gateway, CDN (not just app code)
6. **Adaptive limits** — adjust based on server health (bonus points)
7. **Multi-region** — how to handle globally distributed rate limiting (senior level)
