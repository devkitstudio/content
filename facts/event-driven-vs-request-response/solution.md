# Hybrid: Events + Synchronous Responses

## The Problem

```
User places order (HTTP POST)
  → OrderCreated event published
  → Async: Payment processing
  → Async: Inventory update
  → Async: Notification sent

But user needs response in 3 seconds:
"Order confirmation #12345"
```

Event-driven is great for async, but synchronous responses break the pattern.

## Solution 1: Commands vs Events (Recommended)

Separate **commands** (request-response) from **events** (fire-and-forget):

```
Command (Synchronous)
├─ PlaceOrderCommand (request/response pattern)
│  ├─ Input: user_id, items[]
│  ├─ Process: validate, reserve inventory
│  └─ Output: order_id, confirmation
│
Event (Asynchronous)
└─ OrderPlaced event published
   ├─ Subscribers: PaymentProcessor, NotificationService
   ├─ Fire-and-forget
   └─ Failures don't block user
```

**Command handler (synchronous):**
```python
def place_order(user_id, items):
    # Validation (fast, in-process)
    if not validate_items(items):
        raise ValidationError()

    if not check_inventory(items):
        raise OutOfStockError()

    # Create order (with DB transaction)
    order = Order.create(user_id=user_id, items=items)
    order.save()

    # Return immediately to user
    return {
        "order_id": order.id,
        "status": "confirmed",
        "total": order.total
    }

    # Then asynchronously (after response):
    # emit OrderPlaced(order_id) event
```

## Solution 2: Reply Queues (Request-Reply Pattern)

For async operations that need responses:

```
Client (HTTP Request)
  │
  ├─ Send: {command: "charge_payment", reply_to: "reply-queue-123"}
  │
  └─ Wait for response on reply-queue-123
     ↓
  Message Broker
     │
     ├─ Topic: commands.payment
     │
     └─ Consumer: PaymentService
        │
        ├─ Process charge
        │
        └─ Send response to: reply-queue-123
           ↓
        Client receives response
```

**RPC over messaging:**
```javascript
// Client-side
async function chargePayment(amount) {
  const replyQueueName = `reply.${uuid()}`;
  const channel = await connection.createChannel();

  // Create temporary reply queue
  await channel.assertQueue(replyQueueName, { exclusive: true });

  // Send command with reply_to header
  await channel.sendToQueue('payment.commands', Buffer.from(
    JSON.stringify({
      action: 'charge',
      amount: amount,
      reply_to: replyQueueName
    })
  ));

  // Wait for reply
  return new Promise((resolve) => {
    channel.consume(replyQueueName, (msg) => {
      const response = JSON.parse(msg.content.toString());
      resolve(response);
      channel.ack(msg);
    });
  });
}

// Server-side (PaymentService)
channel.consume('payment.commands', async (msg) => {
  const cmd = JSON.parse(msg.content.toString());

  try {
    const result = await chargeCard(cmd.amount);

    // Send response back to reply queue
    await channel.sendToQueue(cmd.reply_to, Buffer.from(
      JSON.stringify({ success: true, transactionId: result.id })
    ));
  } catch (error) {
    await channel.sendToQueue(cmd.reply_to, Buffer.from(
      JSON.stringify({ success: false, error: error.message })
    ));
  }

  channel.ack(msg);
});
```

## Solution 3: Choreography (Saga Pattern for Distributed Transactions)

When order placement involves multiple async services:

```
PlaceOrder (Command)
  │
  ├─ Create order (fast)
  ├─ Return order_id to user
  └─ Emit: OrderPlaced
     │
     ├─ PaymentService listens
     │  └─ Charge card
     │     └─ Emit: PaymentCharged or PaymentFailed
     │
     ├─ InventoryService listens
     │  └─ Reserve inventory
     │     └─ Emit: InventoryReserved or ReservationFailed
     │
     └─ NotificationService listens
        └─ Send confirmation email
```

**Choreography pattern (Saga):**
```python
# Order Service
@event_handler(OrderPlaced)
def handle_order_placed(event):
    order = Order.get(event.order_id)
    # Publish async event, don't wait
    event_bus.publish(PaymentRequested(order.id, order.total))

# Payment Service
@event_handler(PaymentRequested)
def handle_payment_requested(event):
    try:
        result = charge_card(event.order_id, event.amount)
        event_bus.publish(PaymentCharged(event.order_id, result.transaction_id))
    except Exception as e:
        event_bus.publish(PaymentFailed(event.order_id, str(e)))

# Inventory Service
@event_handler(PaymentCharged)
def handle_payment_charged(event):
    try:
        reserve_inventory(event.order_id)
        event_bus.publish(InventoryReserved(event.order_id))
    except Exception as e:
        # Compensation: refund payment
        event_bus.publish(RefundRequested(event.order_id))
        event_bus.publish(InventoryReservationFailed(event.order_id))

# Compensation
@event_handler(RefundRequested)
def handle_refund_requested(event):
    refund(event.order_id)
    event_bus.publish(PaymentRefunded(event.order_id))
```

## Recommendation Matrix

| Scenario | Pattern | Reason |
|----------|---------|--------|
| User needs immediate response | Command (sync) | Return in <100ms |
| Long operation, user waits | Reply queue (RPC) | User needs response eventually |
| Multiple services involved | Saga (choreography) | Distributed transaction handling |
| Fire-and-forget notification | Event (async) | Don't wait for delivery |
| Payment processing | Command + Event | Verify funds sync, process async |

## Best Practice

```
┌─────────────────────────────────────────┐
│  HTTP Request (PlaceOrder)              │
└──────┬──────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│ Command Handler (Synchronous)           │
├─────────────────────────────────────────┤
│ • Validate input                        │
│ • Check inventory (quick)               │
│ • Create order (DB)                     │
│ • Return: 200 OK + order_id             │
└──────┬──────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│ Emit Events (Asynchronous)              │
├─────────────────────────────────────────┤
│ • OrderPlaced event → Kafka             │
│ • PaymentProcessor consumes             │
│ • InventoryUpdater consumes             │
│ • NotificationService consumes          │
│ (All independent, no retry blocks user) │
└─────────────────────────────────────────┘
```

**The key:** Keep command handlers **fast** (< 500ms), offload heavy work to events.
