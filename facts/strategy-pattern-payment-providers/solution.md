## The Anti-Pattern (Before)

```typescript
class PaymentProcessor {
  async charge(amount: number, provider: string) {
    if (provider === 'stripe') {
      // Stripe logic (200 lines)
      const stripeClient = new Stripe(config.stripeKey);
      const charge = await stripeClient.charges.create({ amount });
      return { id: charge.id, status: 'success' };
    }

    if (provider === 'paypal') {
      // PayPal logic (200 lines)
      const paypalClient = new PayPal(config.paypalKey);
      const payment = await paypalClient.createPayment({ amount });
      return { id: payment.id, status: 'pending' };
    }

    if (provider === 'momo') {
      // MoMo logic (200 lines)
      const momoClient = new MoMo(config.momoKey);
      const response = await momoClient.pay({ amount });
      return { id: response.transactionId, status: 'pending' };
    }

    throw new Error('Unknown provider');
  }
}

// Adding a new provider requires:
// 1. Modify PaymentProcessor (add if statement)
// 2. Modify tests
// 3. Update documentation
// 4. Change service that uses it
// = 4 files changed, violation of Open/Closed principle
```

## The Strategy Pattern (After)

```typescript
// 1. Define the strategy interface
interface PaymentStrategy {
  charge(amount: number): Promise<PaymentResult>;
  refund(transactionId: string, amount: number): Promise<RefundResult>;
  getStatus(transactionId: string): Promise<string>;
}

interface PaymentResult {
  id: string;
  status: 'success' | 'pending' | 'failed';
  amount: number;
}

// 2. Implement each provider
class StripeStrategy implements PaymentStrategy {
  private client: StripeClient;

  constructor(apiKey: string) {
    this.client = new Stripe({ apiKey });
  }

  async charge(amount: number): Promise<PaymentResult> {
    const charge = await this.client.charges.create({
      amount: Math.round(amount * 100), // cents
    });

    return {
      id: charge.id,
      status: charge.paid ? 'success' : 'failed',
      amount,
    };
  }

  async refund(transactionId: string, amount: number) {
    const refund = await this.client.refunds.create({
      charge: transactionId,
      amount: Math.round(amount * 100),
    });
    return { id: refund.id, status: 'success' };
  }

  async getStatus(transactionId: string) {
    const charge = await this.client.charges.retrieve(transactionId);
    return charge.paid ? 'success' : 'pending';
  }
}

class PayPalStrategy implements PaymentStrategy {
  private client: PayPalClient;

  constructor(clientId: string, clientSecret: string) {
    this.client = new PayPal({ clientId, clientSecret });
  }

  async charge(amount: number): Promise<PaymentResult> {
    const payment = await this.client.createPayment({
      amount,
      currency: 'USD',
    });

    return {
      id: payment.id,
      status: 'pending', // PayPal is async
      amount,
    };
  }

  async refund(transactionId: string, amount: number) {
    const refund = await this.client.refundPayment({
      paymentId: transactionId,
      amount,
    });
    return { id: refund.id, status: 'pending' };
  }

  async getStatus(transactionId: string) {
    const payment = await this.client.getPayment(transactionId);
    return payment.state;
  }
}

class MoMoStrategy implements PaymentStrategy {
  private client: MoMoClient;

  constructor(partnerCode: string, accessKey: string) {
    this.client = new MoMo({ partnerCode, accessKey });
  }

  async charge(amount: number): Promise<PaymentResult> {
    const response = await this.client.requestPayment({
      amount,
      orderId: generateOrderId(),
    });

    return {
      id: response.requestId,
      status: response.resultCode === 0 ? 'success' : 'failed',
      amount,
    };
  }

  async refund(transactionId: string, amount: number) {
    const refund = await this.client.refundPayment({
      transactionId,
      amount,
    });
    return { id: refund.transId, status: 'success' };
  }

  async getStatus(transactionId: string) {
    const status = await this.client.queryTransaction(transactionId);
    return status.state;
  }
}

// 3. Use strategy in processor (simple!)
class PaymentProcessor {
  private strategy: PaymentStrategy;

  constructor(provider: string, config: any) {
    this.strategy = this.createStrategy(provider, config);
  }

  private createStrategy(provider: string, config: any): PaymentStrategy {
    switch (provider) {
      case 'stripe':
        return new StripeStrategy(config.stripeKey);
      case 'paypal':
        return new PayPalStrategy(config.paypalClientId, config.paypalClientSecret);
      case 'momo':
        return new MoMoStrategy(config.momoPartnerCode, config.momoAccessKey);
      default:
        throw new Error(`Unknown provider: ${provider}`);
    }
  }

  async charge(amount: number): Promise<PaymentResult> {
    return this.strategy.charge(amount);
  }

  async refund(transactionId: string, amount: number) {
    return this.strategy.refund(transactionId, amount);
  }

  async getStatus(transactionId: string) {
    return this.strategy.getStatus(transactionId);
  }
}

// Usage
const processor = new PaymentProcessor('stripe', config);
const result = await processor.charge(100);
console.log(result); // { id: 'ch_123', status: 'success', amount: 100 }
```

## Adding a New Provider (Easy!)

```typescript
// Only add one new file: AliPayStrategy.ts

class AliPayStrategy implements PaymentStrategy {
  private client: AliPayClient;

  constructor(appId: string, privateKey: string) {
    this.client = new AliPay({ appId, privateKey });
  }

  async charge(amount: number): Promise<PaymentResult> {
    // AliPay-specific logic
    const payment = await this.client.pay({ amount });
    return {
      id: payment.tradeNo,
      status: 'pending',
      amount,
    };
  }

  async refund(transactionId: string, amount: number) {
    const refund = await this.client.refund(transactionId, amount);
    return { id: refund.refundNo, status: 'success' };
  }

  async getStatus(transactionId: string) {
    const payment = await this.client.query(transactionId);
    return payment.tradeStatus;
  }
}

// Then update PaymentProcessor constructor:
private createStrategy(provider: string, config: any): PaymentStrategy {
  switch (provider) {
    case 'stripe':
      return new StripeStrategy(config.stripeKey);
    case 'paypal':
      return new PayPalStrategy(config.paypalClientId, config.paypalClientSecret);
    case 'momo':
      return new MoMoStrategy(config.momoPartnerCode, config.momoAccessKey);
    case 'alipay':  // ← ONE LINE ADDED
      return new AliPayStrategy(config.aliPayAppId, config.aliPayPrivateKey);
    default:
      throw new Error(`Unknown provider: ${provider}`);
  }
}

// That's it! No other files need changes.
// All the logic for each provider is isolated.
```

## Benefits

| Before | After |
|--------|-------|
| 10 files changed per provider | 1 file added |
| 2000-line PaymentProcessor | 100-line PaymentProcessor |
| Tight coupling | Loose coupling |
| Hard to test | Easy to test |
| Add provider = 3 days | Add provider = 30 mins |

## Testing

```typescript
describe('PaymentProcessor with different strategies', () => {
  it('should work with Stripe', async () => {
    const processor = new PaymentProcessor('stripe', { stripeKey: 'test' });
    const result = await processor.charge(100);
    expect(result.amount).toBe(100);
  });

  it('should work with PayPal', async () => {
    const processor = new PaymentProcessor('paypal', {
      paypalClientId: 'test',
      paypalClientSecret: 'test'
    });
    const result = await processor.charge(100);
    expect(result.amount).toBe(100);
  });

  // Easy to mock for testing
  it('should use correct strategy', async () => {
    const mockStrategy = mock<PaymentStrategy>();
    mockStrategy.charge.mockResolvedValue({ id: '123', status: 'success', amount: 100 });

    const processor = new PaymentProcessor('stripe', config);
    (processor as any).strategy = mockStrategy; // Inject mock

    await processor.charge(100);
    expect(mockStrategy.charge).toHaveBeenCalledWith(100);
  });
});
```
