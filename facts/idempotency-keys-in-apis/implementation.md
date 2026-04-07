Stripe popularized this pattern. Here is how you would implement it in a Node.js / Redis environment.

### 1. Redis Lua Script (Atomic Lock)
You cannot just do a `.get()` then `.set()` because of race conditions! Two requests arriving at the exact same millisecond might both see nothing in the cache. You must use an atomic operation like `SETNX` (Set if Not eXists).

```typescript
async function processPaymentWithIdempotency(req, res) {
  const idempotencyKey = req.headers['x-idempotency-key'];
  if (!idempotencyKey) return res.status(400).send("Header required");

  const cacheKey = `idempotent_req:${idempotencyKey}`;
  
  // 1. Try to lock it atomically!
  const isFirstArrival = await redis.set(cacheKey, 'PROCESSING', 'NX', 'EX', 86400);

  if (!isFirstArrival) {
    // 2. We are request #2. Wait for Request #1 to finish or return cached result.
    const cachedResult = await redis.get(cacheKey);
    if (cachedResult === 'PROCESSING') {
      return res.status(409).json({ error: "Request is already in progress, please wait." });
    }
    // Return the ghosted response!
    return res.status(200).json(JSON.parse(cachedResult));
  }

  // 3. We are Request #1. Safe to process!
  try {
    const paymentResult = await Stripe.charge(req.body.amount);
    
    // 4. Save result for future identical requests
    await redis.set(cacheKey, JSON.stringify(paymentResult), 'EX', 86400);
    
    return res.status(200).json(paymentResult);
  } catch (error) {
    // Release lock so client can retry on true failure
    await redis.del(cacheKey);
    throw error;
  }
}
```
