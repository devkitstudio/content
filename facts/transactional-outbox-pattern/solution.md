## The Problem

```typescript
// DANGEROUS: Two separate operations = data loss
async function createOrder(order: Order) {
  // Write 1: Succeeds
  await db.orders.insert(order);

  try {
    // Write 2: May fail!
    await kafka.publish('order-created', order);
  } catch (err) {
    // Order exists in DB but no event published
    // Consumer never knows → inconsistency
  }
}
```

Result: Eventual consistency breaks. Other services don't know about the order.

## The Solution: Transactional Outbox

**Pattern**: Write data AND events to same database transaction. Separate service publishes events.

```
1. Application writes to orders + outbox table (SAME TRANSACTION)
2. Database commits both or neither
3. Separate publisher polls/subscribes to outbox table
4. Publisher sends to Kafka
5. Publisher marks event as published
```

### Step 1: Create Outbox Table

```sql
CREATE TABLE outbox (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  aggregate_id UUID NOT NULL,
  aggregate_type VARCHAR NOT NULL,
  event_type VARCHAR NOT NULL,
  payload JSONB NOT NULL,
  published_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_outbox_published ON outbox(published_at)
  WHERE published_at IS NULL;

CREATE INDEX idx_outbox_aggregate ON outbox(aggregate_type, aggregate_id);
```

### Step 2: Write Atomically

```typescript
async function createOrder(order: Order) {
  const client = await pool.connect();

  try {
    await client.query('BEGIN');

    // Write order
    await client.query(
      'INSERT INTO orders (id, user_id, amount) VALUES ($1, $2, $3)',
      [order.id, order.userId, order.amount]
    );

    // Write event to outbox (SAME TRANSACTION)
    await client.query(
      `INSERT INTO outbox (aggregate_id, aggregate_type, event_type, payload)
       VALUES ($1, $2, $3, $4)`,
      [
        order.id,
        'Order',
        'OrderCreated',
        JSON.stringify({
          orderId: order.id,
          userId: order.userId,
          amount: order.amount,
          createdAt: new Date()
        })
      ]
    );

    await client.query('COMMIT');
    return order;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}
```

### Step 3: Outbox Publisher Service

**Polling approach** (simpler):

```typescript
async function publishOutbox() {
  // Get unpublished events
  const events = await db.query(
    `SELECT * FROM outbox
     WHERE published_at IS NULL
     ORDER BY created_at ASC
     LIMIT 100`
  );

  for (const event of events) {
    try {
      // Publish to Kafka
      await kafka.producer.send({
        topic: `${event.aggregate_type}.${event.event_type}`,
        messages: [{
          key: event.aggregate_id,
          value: JSON.stringify(event.payload)
        }]
      });

      // Mark as published (atomic)
      await db.query(
        'UPDATE outbox SET published_at = NOW() WHERE id = $1',
        [event.id]
      );
    } catch (err) {
      // Failed to publish, retry on next poll
      console.error(`Failed to publish event ${event.id}:`, err);
      // Don't update published_at, will retry
    }
  }
}

// Poll every 5 seconds
setInterval(publishOutbox, 5000);
```

**CDC approach** (more sophisticated):

```typescript
// Using Debezium to capture changes from transaction log
const debezium = {
  topic: 'cdc.outbox',
  // Automatically captures outbox table inserts
  // Send to Kafka immediately
};

// Consumer auto-publishes
kafka.consumer.subscribe({ topic: 'cdc.outbox' });
kafka.consumer.run({
  eachMessage: async ({ message }) => {
    const event = JSON.parse(message.value.toString());
    await kafka.publish(
      `${event.aggregate_type}.${event.event_type}`,
      event.payload
    );
  }
});
```

## Guarantees

**Exactly-once semantics:**
- If order created, event published exactly once
- If publisher crashes, resumes where it left off
- Idempotent consumer required on other end

**Atomicity:**
- Order + event both written or both not
- No orphaned events or missing data

**Eventual consistency:**
- Events publish quickly (polling every 5s)
- Other services eventually consistent within seconds
