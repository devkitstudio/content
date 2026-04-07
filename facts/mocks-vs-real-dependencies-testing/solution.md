## The Testing Pyramid

```
        E2E Tests (Real browser, real backend)
           ↑ Few, Slow, $$$
           |
   Integration Tests (Real DB, real HTTP)
           ↑ Some, Medium, $$
           |
    Unit Tests (All mocked)
           ↑ Many, Fast, $
           |
    Code
```

**Balance:** 10% E2E, 20% Integration, 70% Unit

## When to Mock

Use mocks for:
- **External APIs** (payment processors, weather APIs)
- **Expensive operations** (AWS S3 uploads, email sending)
- **Unreliable services** (third-party APIs that go down)
- **Non-deterministic behavior** (random.Random(), Date.now())

```typescript
// Correct: Mock external dependency
describe('PaymentService', () => {
  it('should charge customer', async () => {
    const stripe = mock<StripeClient>();
    stripe.charge.mockResolvedValue({ id: 'ch_123' });

    const service = new PaymentService(stripe);
    const result = await service.processPayment(100);

    expect(result.id).toBe('ch_123');
    expect(stripe.charge).toHaveBeenCalledWith(100);
  });
});
```

## When to Use Real Dependencies

Use real dependencies for:
- **Database operations** (integration tests)
- **Serialization logic** (JSON, XML parsing)
- **Internal business logic** (core algorithms)
- **Integration points** (database + repository pattern)

```typescript
// Correct: Use real database
describe('UserRepository', () => {
  let db: Database;

  beforeEach(() => {
    db = new SQLite(':memory:'); // Real in-memory DB
    db.exec(CREATE_SCHEMA);
  });

  it('should save and retrieve user', async () => {
    const repo = new UserRepository(db);
    await repo.save({ id: 1, name: 'Alice' });

    const user = await repo.getById(1);
    expect(user.name).toBe('Alice');
  });
});
```

## Anti-Patterns

```typescript
// WRONG: Mocking the thing you want to test
describe('ProductRepository', () => {
  it('should return products', () => {
    const mockDb = mock<Database>();
    mockDb.query.mockResolvedValue([{ id: 1, name: 'Widget' }]);

    const repo = new ProductRepository(mockDb);
    // ^^^ Mocking the database for a DB test is useless!
    // The mock will always return what you tell it to

    expect(repo.getAll()).resolves.toEqual([...]);
  });
});
```

**Better:**

```typescript
// CORRECT: Real database
describe('ProductRepository', () => {
  it('should return products', async () => {
    const db = new SQLite(':memory:');
    db.exec(`CREATE TABLE products (id INT, name TEXT)`);
    db.exec(`INSERT INTO products VALUES (1, 'Widget')`);

    const repo = new ProductRepository(db);
    const products = await repo.getAll();

    expect(products[0].name).toBe('Widget');
  });
});
```

## Practical Framework

```typescript
// Layer 1: Unit tests (all mocked)
describe('DiscountCalculator', () => {
  it('should apply 10% discount', () => {
    const calculator = new DiscountCalculator();
    expect(calculator.apply(100, 0.1)).toBe(90);
  });
});

// Layer 2: Integration tests (real DB, mocked external APIs)
describe('OrderService', () => {
  let db: Database;
  let stripe: MockedStripeClient;

  beforeEach(() => {
    db = new SQLite(':memory:');
    stripe = mock<StripeClient>();
  });

  it('should create order and charge customer', async () => {
    stripe.charge.mockResolvedValue({ id: 'ch_123' });

    const service = new OrderService(db, stripe);
    const order = await service.createAndCharge({
      items: [{ id: 1, qty: 2 }],
      amount: 100
    });

    expect(order.id).toBeDefined();
    expect(stripe.charge).toHaveBeenCalled();
    expect(await db.getOrder(order.id)).toBeDefined();
  });
});

// Layer 3: E2E tests (everything real)
describe('Checkout Flow', () => {
  it('should complete purchase', async () => {
    const browser = await puppeteer.launch();
    const page = await browser.newPage();

    await page.goto('http://localhost:3000/checkout');
    await page.fill('[name="email"]', 'test@example.com');
    await page.click('button:text("Buy now")');

    await page.waitForNavigation();
    expect(page.url()).toContain('/success');
  });
});
```

## Determining Mock Boundaries

| Component | Type | Dependencies | Mock? |
|-----------|------|--------------|-------|
| UserValidator | Logic | None | No mocks needed |
| UserRepository | DB Access | Database | Use real DB |
| PaymentProcessor | External API | Stripe API | Mock API |
| OrderService | Orchestration | UserRepository, PaymentProcessor | Real repo, mock API |
| HttpClient | Infrastructure | Network | Mock for unit, real for integration |

## Why All-Mocking Fails

```typescript
// All mocked = false confidence
describe('CheckoutFlow', () => {
  const mockDb = mock<Database>();
  const mockStripe = mock<StripeClient>();
  const mockEmailService = mock<EmailService>();

  mockDb.saveOrder.mockResolvedValue({ id: 1 });
  mockStripe.charge.mockResolvedValue({ success: true });
  mockEmailService.send.mockResolvedValue(true);

  it('should checkout', async () => {
    const checkout = new Checkout(mockDb, mockStripe, mockEmailService);
    await checkout.process({ items: [...], amount: 100 });

    expect(mockDb.saveOrder).toHaveBeenCalled();
    expect(mockStripe.charge).toHaveBeenCalled();
  });
  // ^^^ Test passes. In production: Database schema changed,
  //     Stripe API changed format, or email service is down.
  //     None of these are caught!
});
```

**The Fix:** Have at least one test with real database and real Stripe sandbox.
