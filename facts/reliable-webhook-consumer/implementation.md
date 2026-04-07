# Complete Webhook Implementation with Signature Verification

## Verify Webhook Signature (Stripe Example)

```javascript
const crypto = require('crypto');
const express = require('express');

// CRITICAL: Use raw body, not parsed JSON
app.post('/webhooks/stripe',
  express.raw({ type: 'application/json' }),
  async (req, res) => {
    const sig = req.headers['stripe-signature'];
    const webhookSecret = process.env.STRIPE_WEBHOOK_SECRET;

    // Verify signature
    try {
      const event = stripe.webhooks.constructEvent(
        req.body,
        sig,
        webhookSecret
      );
      await handleEvent(event);
      res.json({ received: true });
    } catch (err) {
      console.error('Webhook signature verification failed:', err);
      res.status(400).json({ error: 'Invalid signature' });
    }
  }
);

// Manual signature verification (PayPal example)
function verifyPayPalSignature(req) {
  const transmissionId = req.headers['paypal-transmission-id'];
  const transmissionTime = req.headers['paypal-transmission-time'];
  const certUrl = req.headers['paypal-cert-url'];
  const authAlgo = req.headers['paypal-auth-algo'];
  const transmissionSig = req.headers['paypal-transmission-sig'];
  const webhookId = process.env.PAYPAL_WEBHOOK_ID;

  // Construct the expected signature
  const expectedSig = crypto
    .createHash('sha256')
    .update(`${transmissionId}|${transmissionTime}|${webhookId}|${getBodyHash(req.body)}`)
    .digest('hex');

  return expectedSig === transmissionSig;
}

function getBodyHash(body) {
  return crypto
    .createHash('sha256')
    .update(JSON.stringify(body))
    .digest('base64');
}
```

## Full Webhook Handler with Error Handling

```javascript
async function handleEvent(event) {
  const startTime = Date.now();

  try {
    // Idempotency check
    const existing = await WebhookEvent.findOne({ eventId: event.id });
    if (existing) {
      logger.info(`Duplicate event: ${event.id}`);
      return { status: 'duplicate' };
    }

    // Process based on type
    let result;
    switch (event.type) {
      case 'charge.succeeded':
        result = await handleChargeSucceeded(event.data.object);
        break;
      case 'charge.failed':
        result = await handleChargeFailed(event.data.object);
        break;
      case 'customer.subscription.deleted':
        result = await handleSubscriptionCancelled(event.data.object);
        break;
      default:
        logger.warn(`Unknown event type: ${event.type}`);
        return { status: 'ignored' };
    }

    // Record successful processing
    await WebhookEvent.create({
      eventId: event.id,
      type: event.type,
      providerId: event.id.split('_')[0], // stripe, paypal, etc
      status: 'success',
      result,
      processingTimeMs: Date.now() - startTime,
      processedAt: new Date()
    });

    return { status: 'success', result };
  } catch (error) {
    // Record failure
    await WebhookEvent.create({
      eventId: event.id,
      type: event.type,
      status: 'failed',
      error: error.message,
      stack: error.stack,
      processingTimeMs: Date.now() - startTime,
      processedAt: new Date()
    });

    // Rethrow to trigger retry
    throw error;
  }
}

async function handleChargeSucceeded(charge) {
  const customerId = charge.customer;
  const amount = charge.amount / 100; // Convert from cents

  // Update customer balance in our system
  const customer = await Customer.findOne({ stripeId: customerId });
  if (!customer) {
    throw new Error(`Customer not found: ${customerId}`);
  }

  // Update payment record
  const payment = await Payment.findOneAndUpdate(
    { stripeChargeId: charge.id },
    {
      status: 'completed',
      completedAt: new Date(),
      amount
    },
    { new: true }
  );

  // Trigger fulfillment
  if (payment.orderId) {
    await triggerOrderFulfillment(payment.orderId);
  }

  return { customerId, amount, paymentId: payment._id };
}

async function handleChargeFailed(charge) {
  const customerId = charge.customer;
  const failureReason = charge.failure_message;

  // Update payment status
  const payment = await Payment.findOneAndUpdate(
    { stripeChargeId: charge.id },
    {
      status: 'failed',
      failedAt: new Date(),
      failureReason
    },
    { new: true }
  );

  // Notify customer
  await sendFailureEmail(customerId, failureReason);

  // Retry logic (optional)
  if (charge.failure_code === 'card_declined') {
    await scheduleRetry(payment._id, delayMs = 86400000); // Retry in 24 hours
  }

  return { customerId, failureReason };
}

async function handleSubscriptionCancelled(subscription) {
  const customerId = subscription.customer;

  // Update subscription in database
  await Subscription.findOneAndUpdate(
    { stripeId: subscription.id },
    {
      status: 'cancelled',
      cancelledAt: new Date(),
      cancelledReason: subscription.cancellation_details?.reason
    }
  );

  // Cleanup: revoke access if applicable
  await revokeCustomerAccess(customerId);

  // Send confirmation email
  await sendCancellationEmail(customerId);

  return { customerId, subscriptionId: subscription.id };
}
```

## Webhook Retry & Monitoring

```javascript
// Express middleware for webhook health
app.get('/health/webhooks', async (req, res) => {
  const stats = await WebhookEvent.aggregate([
    {
      $group: {
        _id: '$status',
        count: { $sum: 1 }
      }
    },
    {
      $group: {
        _id: null,
        total: { $sum: '$count' },
        statuses: { $push: { status: '$_id', count: '$count' } }
      }
    }
  ]);

  const failedRecent = await WebhookEvent.countDocuments({
    status: 'failed',
    processedAt: { $gte: new Date(Date.now() - 3600000) } // Last hour
  });

  res.json({
    status: failedRecent === 0 ? 'healthy' : 'degraded',
    stats: stats[0],
    recentFailures: failedRecent
  });
});

// Retry failed webhooks
async function retryFailedWebhooks() {
  const failed = await WebhookEvent.find({
    status: 'failed',
    retryCount: { $lt: 3 },
    processedAt: { $lt: new Date(Date.now() - 300000) } // 5 min ago
  });

  for (const event of failed) {
    try {
      await handleEvent({ id: event.eventId, type: event.type });
      await WebhookEvent.updateOne(
        { _id: event._id },
        { status: 'success', retryCount: event.retryCount + 1 }
      );
    } catch (err) {
      await WebhookEvent.updateOne(
        { _id: event._id },
        { retryCount: event.retryCount + 1 }
      );
    }
  }
}

// Run every 5 minutes
setInterval(retryFailedWebhooks, 300000);
```
