# Error Handling in Parallel Execution

## Pitfall 1: Promise.all() Fails on First Error
If any promise rejects, the entire operation fails and others keep running.

```javascript
// WRONG - one failure stops everything
async function getData() {
  return Promise.all([
    fetch('/api/users'),    // Takes 2 seconds
    fetch('/api/broken'),   // Fails after 1 second
    fetch('/api/products')  // Takes 1.5 seconds but never used
  ]);
  // Result: Everything fails, products fetch still runs in background
}

// RIGHT - continue on partial failure
async function getData() {
  const results = await Promise.allSettled([
    fetch('/api/users'),
    fetch('/api/broken'),
    fetch('/api/products')
  ]);

  return {
    users: results[0].status === 'fulfilled' ? results[0].value : null,
    products: results[2].status === 'fulfilled' ? results[2].value : null,
    error: results[1].reason
  };
}
```

## Pitfall 2: Unhandled Promise Rejections
Fire-and-forget promises without proper error handling.

```javascript
// WRONG - uncaught rejection
function triggerParallelTasks() {
  Promise.all([
    heavyTask1(),
    heavyTask2(),
    heavyTask3()
  ]); // No .catch() - errors disappear
}

// RIGHT - always handle promise chains
function triggerParallelTasks() {
  return Promise.all([
    heavyTask1(),
    heavyTask2(),
    heavyTask3()
  ]).catch(err => {
    logger.error('Parallel task failed:', err);
    // Handle gracefully - retry, fallback, notify user
  });
}
```

## Pitfall 3: Cascading Timeouts
One slow service delays all others.

```javascript
// WRONG - wait forever
async function fetchAllData() {
  return Promise.all([
    fetchFastAPI(),      // 100ms
    fetchSlowAPI(),      // 30 seconds (network issue)
    fetchMediumAPI()     // 500ms
  ]);
  // Result: Waits 30+ seconds total
}

// RIGHT - timeout individual promises
function withTimeout(promise, ms) {
  return Promise.race([
    promise,
    new Promise((_, reject) =>
      setTimeout(() => reject(new Error('Timeout')), ms)
    )
  ]);
}

async function fetchAllData() {
  const results = await Promise.allSettled([
    withTimeout(fetchFastAPI(), 5000),
    withTimeout(fetchSlowAPI(), 10000),
    withTimeout(fetchMediumAPI(), 5000)
  ]);

  return results.map((r, i) => ({
    data: r.status === 'fulfilled' ? r.value : null,
    error: r.status === 'rejected' ? r.reason : null
  }));
}
```

## Pitfall 4: Circuit Breaker Not Implemented
Cascading failures when downstream service is unhealthy.

```javascript
// RIGHT - implement circuit breaker
class CircuitBreaker {
  constructor(threshold = 5, timeout = 60000) {
    this.threshold = threshold;
    this.timeout = timeout;
    this.failures = 0;
    this.state = 'CLOSED'; // CLOSED | OPEN | HALF_OPEN
    this.nextAttempt = Date.now();
  }

  async execute(fn) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker OPEN - service unavailable');
      }
      this.state = 'HALF_OPEN';
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (err) {
      this.onFailure();
      throw err;
    }
  }

  onSuccess() {
    this.failures = 0;
    this.state = 'CLOSED';
  }

  onFailure() {
    this.failures++;
    if (this.failures >= this.threshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.timeout;
    }
  }
}

// Usage
const breaker = new CircuitBreaker(3, 30000);

async function fetchWithBreaker() {
  return Promise.allSettled([
    breaker.execute(() => fetchUser()),
    breaker.execute(() => fetchOrders()),
    breaker.execute(() => fetchPayments())
  ]);
}
```

## Pitfall 5: Resource Exhaustion
Too many parallel requests exhaust connection pools.

```javascript
// WRONG - no concurrency limit
async function fetchManyItems(itemIds) {
  // If itemIds has 10,000 items, creates 10,000 concurrent requests
  return Promise.all(
    itemIds.map(id => fetch(`/api/items/${id}`))
  );
}

// RIGHT - batch with concurrency limit
async function batchFetch(itemIds, batchSize = 10) {
  const batches = [];
  for (let i = 0; i < itemIds.length; i += batchSize) {
    batches.push(itemIds.slice(i, i + batchSize));
  }

  const results = [];
  for (const batch of batches) {
    const batchResults = await Promise.all(
      batch.map(id => fetch(`/api/items/${id}`))
    );
    results.push(...batchResults);
  }
  return results;
}
```
