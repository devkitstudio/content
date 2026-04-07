## Saga Implementation Patterns

### Orchestration Saga (Centralized Coordinator)

A central "saga orchestrator" tells each service what to do next:

```typescript
// Saga Orchestrator
class OrderSaga {
  private steps: SagaStep[] = [];

  constructor(private order: Order) {
    this.steps = [
      {
        name: "reserve_inventory",
        execute: () => inventoryService.reserve(order.items),
        compensate: () => inventoryService.release(order.items),
      },
      {
        name: "charge_payment",
        execute: () => paymentService.charge(order.userId, order.total),
        compensate: () => paymentService.refund(order.userId, order.total),
      },
      {
        name: "create_shipment",
        execute: () => shippingService.schedule(order),
        compensate: () => shippingService.cancel(order.id),
      },
      {
        name: "send_confirmation",
        execute: () => notificationService.sendOrderConfirmed(order),
        compensate: () => notificationService.sendOrderCancelled(order),
      },
    ];
  }

  async run(): Promise<SagaResult> {
    const completed: SagaStep[] = [];

    for (const step of this.steps) {
      try {
        await step.execute();
        completed.push(step);
      } catch (error) {
        // Compensate all completed steps in REVERSE order
        console.error(`Saga failed at ${step.name}:`, error);

        for (const done of completed.reverse()) {
          try {
            await done.compensate();
          } catch (compError) {
            // Compensation failed — log for manual intervention
            console.error(
              `CRITICAL: Compensation failed for ${done.name}`,
              compError,
            );
            await alertOps(done.name, compError);
          }
        }

        return { success: false, failedAt: step.name, error };
      }
    }

    return { success: true };
  }
}

// Usage
const saga = new OrderSaga(order);
const result = await saga.run();
```

### Choreography Saga (Event-Driven, No Coordinator)

Each service listens for events and reacts independently:

```typescript
// Order Service
class OrderService {
  async createOrder(data: OrderData) {
    const order = await db.orders.create({ data, status: "PENDING" });
    await eventBus.publish("order.created", {
      orderId: order.id,
      items: data.items,
    });
  }

  @Subscribe("payment.failed")
  async onPaymentFailed(event: PaymentFailedEvent) {
    await db.orders.update({ id: event.orderId, status: "CANCELLED" });
    await eventBus.publish("order.cancelled", { orderId: event.orderId });
  }
}

// Inventory Service
class InventoryService {
  @Subscribe("order.created")
  async onOrderCreated(event: OrderCreatedEvent) {
    try {
      await this.reserveStock(event.items);
      await eventBus.publish("inventory.reserved", { orderId: event.orderId });
    } catch {
      await eventBus.publish("inventory.failed", { orderId: event.orderId });
    }
  }

  @Subscribe("order.cancelled")
  async onOrderCancelled(event: OrderCancelledEvent) {
    await this.releaseStock(event.orderId);
  }
}

// Payment Service
class PaymentService {
  @Subscribe("inventory.reserved")
  async onInventoryReserved(event: InventoryReservedEvent) {
    try {
      await this.charge(event.orderId);
      await eventBus.publish("payment.completed", { orderId: event.orderId });
    } catch {
      await eventBus.publish("payment.failed", { orderId: event.orderId });
    }
  }
}
```

### Orchestration vs Choreography

| Aspect                      | Orchestration                           | Choreography                       |
| --------------------------- | --------------------------------------- | ---------------------------------- |
| **Complexity**              | Centralized logic, easier to understand | Distributed logic, harder to trace |
| **Coupling**                | Orchestrator knows all services         | Services only know events          |
| **Single point of failure** | Orchestrator is SPOF                    | No SPOF                            |
| **Adding steps**            | Modify orchestrator only                | Add new service + events           |
| **Debugging**               | Follow orchestrator logs                | Need distributed tracing           |
| **Best for**                | Complex flows (5+ steps)                | Simple flows (2-3 steps)           |

## Production Frameworks

### Temporal (Recommended for Node.js/Go/Java)

```typescript
import { proxyActivities, defineSignal } from '@temporalio/workflow';

// Activities = the actual service calls
const { reserveInventory, chargePayment, scheduleShipping, sendNotification,
        releaseInventory, refundPayment, cancelShipping } = proxyActivities({
  startToCloseTimeout: '30s',
  retry: { maximumAttempts: 3 },
});

// Workflow = the saga orchestration
export async function orderWorkflow(order: Order): Promise<OrderResult> {
  // Step 1: Reserve inventory
  await reserveInventory(order.items);

  try {
    // Step 2: Charge payment
    await chargePayment(order.userId, order.total);
  } catch {
    await releaseInventory(order.items); // compensate step 1
    throw;
  }

  try {
    // Step 3: Schedule shipping
    await scheduleShipping(order);
  } catch {
    await refundPayment(order.userId, order.total); // compensate step 2
    await releaseInventory(order.items);             // compensate step 1
    throw;
  }

  // Step 4: Send confirmation (non-critical, no compensation needed)
  await sendNotification(order);

  return { status: 'completed', orderId: order.id };
}
```

**Why Temporal?** It handles retries, timeouts, persistence, and recovery automatically. If your service crashes mid-saga, Temporal resumes exactly where it left off.

### AWS Step Functions (Serverless Saga)

```json
{
  "StartAt": "ReserveInventory",
  "States": {
    "ReserveInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:reserve-inventory",
      "Next": "ChargePayment",
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "OrderFailed"
        }
      ]
    },
    "ChargePayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:charge-payment",
      "Next": "OrderComplete",
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "ReleaseInventory"
        }
      ]
    },
    "ReleaseInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:release-inventory",
      "Next": "OrderFailed"
    },
    "OrderComplete": { "Type": "Succeed" },
    "OrderFailed": { "Type": "Fail" }
  }
}
```

## When NOT to Use Saga

| Situation                         | Better Alternative                      |
| --------------------------------- | --------------------------------------- |
| All data in one database          | **Database transaction** (ACID)         |
| 2 services, simple flow           | **Two-phase commit** or direct API call |
| Read-heavy, no writes             | No transaction needed                   |
| Can tolerate eventual consistency | **Outbox pattern** + async processing   |
| All services on same platform     | **Distributed transaction** (XA)        |
