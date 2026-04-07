# Parallelizing API Calls with Dependency Management

## Strategy 1: Promise.all() for Independent Calls
When services have no dependencies, call them in parallel.

```javascript
// SEQUENTIAL (SLOW) - 3 seconds total
async function getOrderSlow(orderId) {
  const order = await fetchOrder(orderId);           // 1 second
  const customer = await fetchCustomer(order.customerId); // 1 second
  const inventory = await fetchInventory(order.itemId);   // 1 second
  return { order, customer, inventory };
}

// PARALLEL (FAST) - 1 second total
async function getOrderFast(orderId) {
  const order = await fetchOrder(orderId);

  // These don't depend on each other - run in parallel
  const [customer, inventory] = await Promise.all([
    fetchCustomer(order.customerId),
    fetchInventory(order.itemId)
  ]);

  return { order, customer, inventory };
}
```

## Strategy 2: Dependency Graph Execution
Run tasks in layers based on dependencies.

```javascript
async function getOrderWithDependencies(orderId) {
  // Layer 1: Fetch root data
  const order = await fetchOrder(orderId);

  // Layer 2: Tasks that depend on order
  const [customer, items, billing] = await Promise.all([
    fetchCustomer(order.customerId),
    fetchItems(order.itemIds),
    fetchBillingHistory(order.customerId)
  ]);

  // Layer 3: Tasks that depend on Layer 2 results
  const [reviews, recommendations] = await Promise.all([
    fetchReviews(items.map(i => i.id)),
    fetchRecommendations(customer.preferences)
  ]);

  return { order, customer, items, billing, reviews, recommendations };
}
```

## Strategy 3: Promise.allSettled for Partial Failure Tolerance
Some services can fail without breaking the entire response.

```javascript
async function getRobustOrderData(orderId) {
  const order = await fetchOrder(orderId);

  const results = await Promise.allSettled([
    fetchCustomer(order.customerId),
    fetchInventory(order.itemId),
    fetchAnalytics(orderId),      // Nice to have, not critical
    fetchRelatedProducts(order.itemId) // Nice to have
  ]);

  const [customerResult, inventoryResult, analyticsResult, relatedResult] = results;

  return {
    order,
    customer: customerResult.status === 'fulfilled' ? customerResult.value : null,
    inventory: inventoryResult.status === 'fulfilled' ? inventoryResult.value : null,
    analytics: analyticsResult.status === 'fulfilled' ? analyticsResult.value : null,
    related: relatedResult.status === 'fulfilled' ? relatedResult.value : []
  };
}
```

## Strategy 4: Conditional Parallelization
Branch execution based on data - some tasks parallel, some sequential.

```javascript
async function complexOrderFlow(orderId, includeAnalytics = false) {
  const order = await fetchOrder(orderId);
  const customer = await fetchCustomer(order.customerId);

  // Parallel branch 1: Inventory operations (independent)
  const inventory = await fetchInventory(order.itemId);
  const stock = await checkWarehouse(order.itemId);

  // Run both branches in parallel
  const [inventoryData, warehouseData] = await Promise.all([
    inventory,
    stock
  ]);

  // Optional: Analytics only if requested and not slowing down response
  let analytics = null;
  if (includeAnalytics) {
    // Fire-and-forget or timeout to avoid blocking
    analytics = await Promise.race([
      fetchAnalytics(orderId),
      new Promise((_, reject) =>
        setTimeout(() => reject(new Error('Timeout')), 1000)
      )
    ]).catch(() => null);
  }

  return { order, customer, inventoryData, warehouseData, analytics };
}
```

## Strategy 5: Worker Pool Pattern for Limited Concurrency
Control parallelism to avoid overwhelming downstream services.

```javascript
class TaskQueue {
  constructor(concurrency = 3) {
    this.concurrency = concurrency;
    this.running = 0;
    this.queue = [];
  }

  async run(fn) {
    while (this.running >= this.concurrency) {
      await new Promise(resolve => this.queue.push(resolve));
    }

    this.running++;
    try {
      return await fn();
    } finally {
      this.running--;
      const resolve = this.queue.shift();
      if (resolve) resolve();
    }
  }
}

// Usage
const taskQueue = new TaskQueue(3); // Max 3 concurrent requests

async function fetchMultipleOrders(orderIds) {
  const promises = orderIds.map(id =>
    taskQueue.run(() => fetchOrder(id))
  );
  return Promise.all(promises);
}
```
