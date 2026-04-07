## Scaling Beyond a Single Database

### Redis Distributed Lock (Redlock)

When your inventory is sharded across databases or microservices:

```typescript
import Redis from 'ioredis';

const redis = new Redis();

async function purchaseWithRedisLock(
  productId: string,
  userId: string,
): Promise<boolean> {
  const lockKey = `lock:product:${productId}`;
  const lockValue = `${userId}:${Date.now()}`;
  const lockTTL = 5000; // 5 second max lock time

  // Acquire lock (SET NX = only if not exists)
  const acquired = await redis.set(lockKey, lockValue, 'PX', lockTTL, 'NX');

  if (!acquired) {
    return false; // Someone else is buying
  }

  try {
    // Check and decrement atomically in Redis
    const stock = await redis.get(`stock:${productId}`);
    if (!stock || parseInt(stock) <= 0) return false;

    await redis.decr(`stock:${productId}`);

    // Sync to database async (eventual consistency)
    await queue.publish('inventory.decremented', { productId, userId });

    return true;
  } finally {
    // Release lock (only if we still own it)
    const script = `
      if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
      else
        return 0
      end
    `;
    await redis.eval(script, 1, lockKey, lockValue);
  }
}
```

### Redis Atomic Decrement (No Lock Needed)

For simple stock deduction, Redis DECR is atomic — no lock required:

```typescript
async function purchaseAtomic(productId: string): Promise<boolean> {
  const key = `stock:${productId}`;

  // DECR is atomic: returns new value after decrement
  const remaining = await redis.decr(key);

  if (remaining < 0) {
    // Oversold! Roll back
    await redis.incr(key);
    return false;
  }

  // Stock reserved successfully
  return true;
}

// Pre-load stock before flash sale
async function preloadStock(productId: string, quantity: number) {
  await redis.set(`stock:${productId}`, quantity);
}
```

**Why this works**: Redis is single-threaded — `DECR` is inherently atomic. No race condition possible.

### Lua Script for Complex Atomic Operations

When you need to check multiple conditions atomically:

```lua
-- reserve_stock.lua
local stockKey = KEYS[1]
local reserveKey = KEYS[2]
local userId = ARGV[1]
local quantity = tonumber(ARGV[2])
local ttl = tonumber(ARGV[3])  -- reservation TTL in seconds

-- Check available stock
local stock = tonumber(redis.call('GET', stockKey) or 0)
if stock < quantity then
  return {0, "INSUFFICIENT_STOCK", stock}
end

-- Check per-user purchase limit
local userPurchases = tonumber(redis.call('GET', 'purchases:' .. userId) or 0)
if userPurchases >= 2 then  -- max 2 per user
  return {0, "PURCHASE_LIMIT", userPurchases}
end

-- Reserve stock (decrement + create reservation)
redis.call('DECRBY', stockKey, quantity)
redis.call('SETEX', reserveKey, ttl, quantity)  -- auto-expire reservation
redis.call('INCR', 'purchases:' .. userId)

return {1, "RESERVED", stock - quantity}
```

```typescript
const result = await redis.eval(luaScript, 2,
  `stock:${productId}`,
  `reserve:${orderId}`,
  userId,
  1,    // quantity
  300,  // 5 min reservation TTL
);
```

### Queue-Based Serialization

Convert concurrent writes into sequential processing:

```typescript
// Instead of letting 10,000 users hit the DB directly,
// funnel all purchases through a single queue

// Producer: API handler
app.post('/api/purchase', async (req, res) => {
  const orderId = uuid();

  // Enqueue and return immediately
  await redis.rpush('purchase_queue', JSON.stringify({
    orderId,
    productId: req.body.productId,
    userId: req.user.id,
    timestamp: Date.now(),
  }));

  // Client polls for result
  res.json({ orderId, status: 'processing' });
});

// Consumer: single worker processes sequentially
async function processQueue() {
  while (true) {
    const item = await redis.blpop('purchase_queue', 0);
    const order = JSON.parse(item![1]);

    const result = await db.query(
      'UPDATE products SET stock = stock - 1 WHERE id = $1 AND stock > 0 RETURNING stock',
      [order.productId]
    );

    const success = result.rowCount > 0;
    await redis.set(`order:${order.orderId}`, JSON.stringify({
      status: success ? 'confirmed' : 'sold_out',
    }), 'EX', 3600);
  }
}
```

**Pros**: Zero race conditions (sequential by design). Can handle extreme spikes (queue absorbs burst).
**Cons**: Higher latency (queue delay). Need polling/WebSocket for result. Single point of failure (mitigate with multiple queues per product).

## Solution Comparison for Interviews

| Solution | Throughput | Consistency | Complexity | Best For |
|----------|-----------|-------------|------------|----------|
| **Atomic UPDATE (WHERE stock > 0)** | High | Strong | Simple | Single DB, most cases |
| **Optimistic Locking (version)** | Medium | Strong | Medium | ORM-heavy apps, low contention |
| **Pessimistic Locking (FOR UPDATE)** | Low | Strong | Medium | High-value items, few buyers |
| **Redis DECR** | Very High | Eventual | Low | Flash sales, read-heavy |
| **Redis Lock (Redlock)** | High | Strong | High | Distributed systems |
| **Redis Lua Script** | Very High | Strong (in Redis) | Medium | Complex conditions |
| **Queue Serialization** | Medium | Strong | High | Extreme concurrency, fairness |
