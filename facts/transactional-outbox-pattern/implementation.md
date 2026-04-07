## Prisma Implementation

```typescript
import { PrismaClient } from '@prisma/client';
import { Kafka } from 'kafkajs';

const prisma = new PrismaClient();
const kafka = new Kafka({
  clientId: 'outbox-publisher',
  brokers: ['localhost:9092']
});

const producer = kafka.producer();

// Database schema
// prisma/schema.prisma:
// model Order {
//   id            String   @id @default(cuid())
//   userId        String
//   amount        Float
//   createdAt     DateTime @default(now())
//   outboxEvents  OutboxEvent[]
// }
//
// model OutboxEvent {
//   id              String   @id @default(cuid())
//   aggregateId     String
//   aggregateType   String
//   eventType       String
//   payload         Json
//   publishedAt     DateTime?
//   createdAt       DateTime @default(now())
//   @@index([publishedAt, createdAt])
// }

// Create order with transactional outbox
async function createOrder(data: {
  userId: string;
  amount: number;
}) {
  return prisma.$transaction(async (tx) => {
    // Create order
    const order = await tx.order.create({
      data: {
        userId: data.userId,
        amount: data.amount
      }
    });

    // Add event to outbox
    await tx.outboxEvent.create({
      data: {
        aggregateId: order.id,
        aggregateType: 'Order',
        eventType: 'OrderCreated',
        payload: {
          orderId: order.id,
          userId: order.userId,
          amount: order.amount,
          createdAt: order.createdAt
        }
      }
    });

    return order;
  });
}

// Outbox publisher
async function publishOutboxEvents() {
  const events = await prisma.outboxEvent.findMany({
    where: { publishedAt: null },
    orderBy: { createdAt: 'asc' },
    take: 100
  });

  for (const event of events) {
    try {
      await producer.send({
        topic: `${event.aggregateType}.${event.eventType}`,
        messages: [{
          key: event.aggregateId,
          value: JSON.stringify(event.payload),
          headers: {
            'event-id': event.id,
            'event-type': event.eventType
          }
        }]
      });

      // Mark as published
      await prisma.outboxEvent.update({
        where: { id: event.id },
        data: { publishedAt: new Date() }
      });

      console.log(`Published event ${event.id}`);
    } catch (error) {
      console.error(`Failed to publish event ${event.id}:`, error);
      // Will retry on next poll
    }
  }
}

// Start publisher
async function startPublisher() {
  await producer.connect();

  setInterval(publishOutboxEvents, 5000);

  process.on('SIGTERM', async () => {
    await producer.disconnect();
    process.exit(0);
  });
}

startPublisher();

// Usage in Express
import express from 'express';

const app = express();
app.use(express.json());

app.post('/api/orders', async (req, res, next) => {
  try {
    const order = await createOrder(req.body);
    res.status(201).json(order);
  } catch (error) {
    next(error);
  }
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

## Multiple Outbox Events

```typescript
async function transferMoney(data: {
  fromAccountId: string;
  toAccountId: string;
  amount: number;
}) {
  return prisma.$transaction(async (tx) => {
    // Update from account
    await tx.account.update({
      where: { id: data.fromAccountId },
      data: { balance: { decrement: data.amount } }
    });

    // Update to account
    await tx.account.update({
      where: { id: data.toAccountId },
      data: { balance: { increment: data.amount } }
    });

    // Create two events atomically
    await tx.outboxEvent.createMany({
      data: [
        {
          aggregateId: data.fromAccountId,
          aggregateType: 'Account',
          eventType: 'MoneyDebited',
          payload: {
            accountId: data.fromAccountId,
            amount: data.amount
          }
        },
        {
          aggregateId: data.toAccountId,
          aggregateType: 'Account',
          eventType: 'MoneyCredited',
          payload: {
            accountId: data.toAccountId,
            amount: data.amount
          }
        }
      ]
    });
  });
}
```

## Error Handling & Cleanup

```typescript
// Mark failed events for dead letter queue
async function handlePublishFailures() {
  const failedEvents = await prisma.outboxEvent.findMany({
    where: {
      publishedAt: null,
      createdAt: {
        lt: new Date(Date.now() - 1000 * 60 * 60) // Older than 1 hour
      }
    }
  });

  for (const event of failedEvents) {
    console.error(`Event ${event.id} failed to publish for 1+ hour`);
    // Send to dead letter queue
    await kafka.producer.send({
      topic: 'dead-letter-queue',
      messages: [{
        key: event.id,
        value: JSON.stringify(event)
      }]
    });

    // Mark as published to stop retrying
    await prisma.outboxEvent.update({
      where: { id: event.id },
      data: { publishedAt: new Date() }
    });
  }
}

// Cleanup old published events
async function cleanupOutbox() {
  await prisma.outboxEvent.deleteMany({
    where: {
      publishedAt: {
        lt: new Date(Date.now() - 1000 * 60 * 60 * 24 * 7) // Older than 7 days
      }
    }
  });
}
```
