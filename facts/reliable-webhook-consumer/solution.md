# Idempotent Webhook Processing with Event Ordering

## Core Principles: Idempotency + Ordering

### Idempotency: Same Event, Same Outcome
Process the same event multiple times safely.

```javascript
// WRONG - duplicate events cause duplicate charges
app.post('/webhooks/payment', async (req, res) => {
  const event = req.body;

  if (event.type === 'payment.completed') {
    // What happens if webhook is retried? Charges twice!
    await chargeCustomer(event.data.customerId, event.data.amount);
  }

  res.json({ received: true });
});

// RIGHT - idempotent processing
app.post('/webhooks/payment', async (req, res) => {
  const event = req.body;
  const eventId = event.id; // Unique event identifier from Stripe

  // Check if we've already processed this event
  const existing = await db.collection('webhook_events').findOne({ eventId });
  if (existing) {
    // Event already processed - return success without re-processing
    return res.json({ received: true, duplicate: true });
  }

  // Process event only once
  if (event.type === 'payment.completed') {
    await chargeCustomer(event.data.customerId, event.data.amount);
  }

  // Record that we processed this event
  await db.collection('webhook_events').insertOne({
    eventId: event.id,
    type: event.type,
    processedAt: new Date(),
    status: 'completed'
  });

  res.json({ received: true });
});
```

### Event Ordering: Process Events in Correct Sequence
Out-of-order events can create inconsistent state.

```javascript
// WRONG - no ordering guarantee
// Event A: user.created (id: 123)
// Event B: user.updated (id: 123, subscription: premium)
// Problem: If B arrives before A, update fails because user doesn't exist

// RIGHT - maintain event sequence numbers
app.post('/webhooks/user-events', async (req, res) => {
  const event = req.body;
  const { userId, sequenceNumber, type, data } = event;

  // Find expected next sequence for this user
  const lastEvent = await db.collection('user_events').findOne(
    { userId },
    { sort: { sequenceNumber: -1 } }
  );

  const expectedNext = (lastEvent?.sequenceNumber ?? -1) + 1;

  if (sequenceNumber < expectedNext) {
    // Duplicate or old event - already processed
    return res.json({ received: true, duplicate: true });
  }

  if (sequenceNumber > expectedNext) {
    // Out of order - queue for later
    await db.collection('webhook_queue').insertOne({
      userId,
      sequenceNumber,
      event,
      receivedAt: new Date()
    });
    return res.json({ received: true, queued: true });
  }

  // Process event in correct order
  await processUserEvent(userId, type, data);

  // Record processed event
  await db.collection('user_events').insertOne({
    userId,
    sequenceNumber,
    type,
    processedAt: new Date()
  });

  // Check if there are queued events we can now process
  const nextQueued = await db.collection('webhook_queue').findOne(
    { userId, sequenceNumber: expectedNext + 1 },
    { sort: { sequenceNumber: 1 } }
  );

  if (nextQueued) {
    // Process queued event recursively
    await processUserEvent(nextQueued.event.userId, nextQueued.event.type, nextQueued.event.data);
    await db.collection('webhook_queue').deleteOne({ _id: nextQueued._id });
  }

  res.json({ received: true });
});
```

## Combined Approach: Transaction-Safe Processing

```javascript
app.post('/webhooks/stripe', async (req, res) => {
  const event = req.body;
  const { id: eventId, type, created, data } = event;

  // Use database transaction for atomicity
  const session = db.startSession();
  await session.withTransaction(async () => {
    // Step 1: Check idempotency
    const existing = await db.collection('webhook_events').findOne(
      { eventId },
      { session }
    );

    if (existing) {
      throw new Error('DUPLICATE_EVENT');
    }

    // Step 2: Verify webhook signature (see implementation.md)
    if (!verifyStripeSignature(req)) {
      throw new Error('INVALID_SIGNATURE');
    }

    // Step 3: Process based on event type
    switch (type) {
      case 'charge.succeeded':
        await handleChargeSucceeded(data.object, session);
        break;
      case 'charge.failed':
        await handleChargeFailed(data.object, session);
        break;
      case 'customer.subscription.updated':
        await handleSubscriptionUpdated(data.object, session);
        break;
      default:
        logger.info(`Unhandled event type: ${type}`);
    }

    // Step 4: Record event as processed
    await db.collection('webhook_events').insertOne(
      {
        eventId,
        type,
        createdAt: new Date(created * 1000),
        processedAt: new Date(),
        status: 'completed'
      },
      { session }
    );
  });

  res.json({ received: true });
});
```

## Idempotency Key Pattern (For Outbound Requests)
When your webhook needs to call external services, use idempotency keys.

```javascript
async function processPaymentEvent(paymentData) {
  const idempotencyKey = `payment_${paymentData.id}`;

  // External service (like Stripe) won't process same idempotency key twice
  const result = await externalAPI.chargeCard({
    amount: paymentData.amount,
    customerId: paymentData.customerId,
    idempotencyKey // Prevents duplicate charges if request retries
  });

  return result;
}
```
