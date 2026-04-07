## Factory Pattern for Strategy Creation

```typescript
// PaymentStrategyFactory.ts
class PaymentStrategyFactory {
  static create(provider: string, config: Record<string, any>): PaymentStrategy {
    const strategies: Record<string, new (...args: any[]) => PaymentStrategy> = {
      stripe: StripeStrategy,
      paypal: PayPalStrategy,
      momo: MoMoStrategy,
      alipay: AliPayStrategy,
    };

    const StrategyClass = strategies[provider];
    if (!StrategyClass) {
      throw new Error(`Unknown payment provider: ${provider}`);
    }

    return this.instantiate(StrategyClass, provider, config);
  }

  private static instantiate(
    StrategyClass: new (...args: any[]) => PaymentStrategy,
    provider: string,
    config: Record<string, any>
  ): PaymentStrategy {
    const providerConfig = config[provider];
    if (!providerConfig) {
      throw new Error(`No configuration found for provider: ${provider}`);
    }

    return new StrategyClass(providerConfig);
  }

  static getAvailableProviders(): string[] {
    return ['stripe', 'paypal', 'momo', 'alipay'];
  }
}

// Usage
const processor = PaymentStrategyFactory.create('stripe', config);
const available = PaymentStrategyFactory.getAvailableProviders();
```

## Async Strategy with Webhooks

```typescript
interface PaymentStrategy {
  charge(amount: number, orderId: string): Promise<PaymentResult>;
  handleWebhook(event: any): Promise<WebhookResult>;
}

class StripeStrategy implements PaymentStrategy {
  async handleWebhook(event: any): Promise<WebhookResult> {
    if (event.type === 'charge.succeeded') {
      return {
        transactionId: event.data.object.id,
        status: 'success',
        amount: event.data.object.amount / 100,
      };
    }

    if (event.type === 'charge.failed') {
      return {
        transactionId: event.data.object.id,
        status: 'failed',
      };
    }

    return { transactionId: '', status: 'unknown' };
  }
}

// API route
app.post('/webhooks/stripe', async (req, res) => {
  const strategy = PaymentStrategyFactory.create('stripe', config);
  const result = await strategy.handleWebhook(req.body);
  // Update database based on result
  res.json({ received: true });
});
```

## Composing with Other Patterns

```typescript
// Combine with Decorator pattern for logging/metrics
class LoggingPaymentStrategy implements PaymentStrategy {
  constructor(private innerStrategy: PaymentStrategy) {}

  async charge(amount: number): Promise<PaymentResult> {
    console.log(`Charging $${amount} with ${this.innerStrategy.constructor.name}`);
    const result = await this.innerStrategy.charge(amount);
    console.log(`Result: ${result.status}`);
    return result;
  }

  async refund(transactionId: string, amount: number) {
    console.log(`Refunding $${amount} for transaction ${transactionId}`);
    return this.innerStrategy.refund(transactionId, amount);
  }

  async getStatus(transactionId: string) {
    return this.innerStrategy.getStatus(transactionId);
  }
}

// Usage
let strategy: PaymentStrategy = new StripeStrategy(config.stripeKey);
if (process.env.DEBUG) {
  strategy = new LoggingPaymentStrategy(strategy);
}

const processor = new PaymentProcessor(strategy);
```

## Dependency Injection

```typescript
// AppContainer.ts
class AppContainer {
  private strategies: Map<string, PaymentStrategy> = new Map();

  registerPaymentStrategy(name: string, strategy: PaymentStrategy) {
    this.strategies.set(name, strategy);
  }

  getPaymentStrategy(name: string): PaymentStrategy {
    const strategy = this.strategies.get(name);
    if (!strategy) {
      throw new Error(`Payment strategy not registered: ${name}`);
    }
    return strategy;
  }

  getPaymentProcessor(provider: string): PaymentProcessor {
    const strategy = this.getPaymentStrategy(provider);
    return new PaymentProcessor(strategy);
  }
}

// Setup (in main.ts)
const container = new AppContainer();
container.registerPaymentStrategy('stripe', new StripeStrategy(config.stripeKey));
container.registerPaymentStrategy('paypal', new PayPalStrategy(config.paypalId, config.paypalSecret));
container.registerPaymentStrategy('momo', new MoMoStrategy(config.momoCode, config.momoKey));

// Usage in services
class OrderService {
  constructor(private container: AppContainer) {}

  async processPayment(orderId: string, provider: string, amount: number) {
    const processor = this.container.getPaymentProcessor(provider);
    return processor.charge(amount);
  }
}
```
