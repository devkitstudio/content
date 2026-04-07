## Idempotency Storage Strategies

### PostgreSQL: Unique Constraint Approach

The simplest production pattern — store the idempotency key as a unique constraint:

```typescript
// Schema
await db.query(`
  CREATE TABLE idempotency_keys (
    key TEXT PRIMARY KEY,
    user_id TEXT NOT NULL,
    request_hash TEXT NOT NULL,    -- hash of request body
    response_code INTEGER,
    response_body JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    expires_at TIMESTAMPTZ DEFAULT NOW() + INTERVAL '24 hours'
  );
  CREATE INDEX idx_idempotency_expires ON idempotency_keys(expires_at);
`);

// Middleware
async function idempotencyMiddleware(req, res, next) {
  const key = req.headers['idempotency-key'];
  if (!key) return next(); // non-idempotent request

  const requestHash = hash(JSON.stringify(req.body));

  try {
    // Try to insert (will fail if key exists due to UNIQUE)
    const { rows } = await db.query(`
      INSERT INTO idempotency_keys (key, user_id, request_hash)
      VALUES ($1, $2, $3)
      ON CONFLICT (key) DO NOTHING
      RETURNING key
    `, [key, req.user.id, requestHash]);

    if (rows.length === 0) {
      // Key exists — return cached response
      const cached = await db.query(
        'SELECT response_code, response_body FROM idempotency_keys WHERE key = $1',
        [key]
      );

      if (cached.rows[0].response_code) {
        return res.status(cached.rows[0].response_code).json(cached.rows[0].response_body);
      }
      // Still processing — return 409 Conflict
      return res.status(409).json({ error: 'Request is being processed' });
    }

    // New request — proceed
    const originalJson = res.json.bind(res);
    res.json = (body) => {
      // Cache the response
      db.query(
        'UPDATE idempotency_keys SET response_code = $1, response_body = $2 WHERE key = $3',
        [res.statusCode, body, key]
      );
      return originalJson(body);
    };

    next();
  } catch (err) {
    next(err);
  }
}
```

### Redis: High-Performance Idempotency

For high-throughput APIs where database latency matters:

```typescript
async function checkIdempotency(key: string, ttlSeconds = 86400) {
  // SET NX = only if not exists (atomic check-and-set)
  const acquired = await redis.set(
    `idem:${key}`,
    JSON.stringify({ status: 'processing', timestamp: Date.now() }),
    'EX', ttlSeconds,
    'NX'
  );

  if (!acquired) {
    const cached = await redis.get(`idem:${key}`);
    return JSON.parse(cached!); // return cached response
  }

  return null; // new request, proceed
}

async function cacheIdempotencyResult(key: string, response: any) {
  await redis.set(
    `idem:${key}`,
    JSON.stringify({ status: 'completed', response, timestamp: Date.now() }),
    'KEEPTTL' // keep existing TTL
  );
}
```

### Stripe's Approach (Industry Standard)

Stripe uses a two-phase idempotency system:

```
Phase 1: Lock
  Client sends POST /v1/charges with Idempotency-Key header
  → Stripe creates a "locked" record (prevents duplicate processing)
  → If key exists and locked → return 409 (in-progress)
  → If key exists and completed → return cached response

Phase 2: Execute + Cache
  → Process the charge
  → Store the response against the key
  → Key expires after 24 hours
```

Key design decisions from Stripe:
- Keys are **scoped per API key** (different API keys can use same idempotency key)
- Keys work on **POST** requests only (GET/DELETE are already idempotent)
- If request body differs for same key → **400 error** (not a retry)
- Results cached for **24 hours** then garbage collected

## Idempotency Beyond HTTP

### Message Queue Idempotency

When consumers process messages from Kafka/RabbitMQ, duplicates are common (at-least-once delivery):

```typescript
// Consumer with deduplication
async function processMessage(message: Message) {
  const messageId = message.headers['message-id'];

  // Check if already processed
  const processed = await redis.get(`msg:${messageId}`);
  if (processed) {
    console.log(`Duplicate message ${messageId}, skipping`);
    return; // acknowledge without processing
  }

  // Process the message
  await handleOrder(message.body);

  // Mark as processed (with TTL for cleanup)
  await redis.set(`msg:${messageId}`, '1', 'EX', 7 * 24 * 3600);
}
```

### Database-Level Idempotency (Upsert Pattern)

Make the operation itself idempotent — no external key needed:

```sql
-- Instead of INSERT (fails on duplicate):
INSERT INTO payments (order_id, amount, status)
VALUES ('order-123', 99.99, 'completed');

-- Use UPSERT (idempotent by design):
INSERT INTO payments (order_id, amount, status)
VALUES ('order-123', 99.99, 'completed')
ON CONFLICT (order_id) DO NOTHING;
-- or: ON CONFLICT (order_id) DO UPDATE SET status = 'completed';
```

```typescript
// Application-level: make handlers naturally idempotent
async function processPayment(orderId: string, amount: number) {
  // Check current state FIRST
  const order = await db.orders.findUnique({ where: { id: orderId } });

  if (order.status === 'paid') {
    return order; // Already processed, return existing result
  }

  if (order.status !== 'pending') {
    throw new Error(`Cannot process order in ${order.status} state`);
  }

  // State machine transition (atomic)
  return db.orders.update({
    where: { id: orderId, status: 'pending' }, // optimistic lock
    data: { status: 'paid', paidAt: new Date(), amount },
  });
}
```

## Interview Tips

When asked about idempotency, structure your answer:

1. **Define**: Same request, same result, no side effects on retry
2. **When**: Payment APIs, webhooks, message queues, any non-GET request
3. **How**: Idempotency keys (Stripe pattern) OR natural idempotency (upserts)
4. **Storage**: PostgreSQL for durability, Redis for speed, both for production
5. **Edge cases**: Concurrent requests with same key, request body mismatch, TTL expiration
