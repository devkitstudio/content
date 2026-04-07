## Test Double Comparison

### 1. Mock

Verifies calls were made correctly.

```typescript
const mockPayment = mock<PaymentGateway>();
mockPayment.charge.mockResolvedValue({ id: 'ch_123' });

await service.pay(100);

expect(mockPayment.charge).toHaveBeenCalledWith(100);
// ^^^ Asserts on behavior (was the method called?)
```

**Use:** External dependencies, verifying API contracts

### 2. Stub

Provides canned responses, no behavior verification.

```typescript
const stubPayment: PaymentGateway = {
  charge: () => Promise.resolve({ id: 'ch_123' })
};

const result = await service.pay(100);
expect(result.success).toBe(true);
// ^^^ Only asserts result, not calls
```

**Use:** Simple return values, no call verification needed

### 3. Fake

Real implementation but with simplified behavior.

```typescript
class FakeDatabase implements Database {
  private store = new Map();

  async save(key: string, value: any) {
    this.store.set(key, value);
  }

  async get(key: string) {
    return this.store.get(key);
  }
}

// Works like real database but stores in memory
const db = new FakeDatabase();
await db.save('user:1', { name: 'Alice' });
expect(await db.get('user:1')).toEqual({ name: 'Alice' });
```

**Use:** When real implementation is too heavy (real DB vs fake), but behavior matters

### 4. Real

Actual production code/infrastructure.

```typescript
const db = new PostgreSQL('postgresql://localhost/test_db');
await db.save('user:1', { name: 'Alice' });
expect(await db.get('user:1')).toEqual({ name: 'Alice' });
```

**Use:** Integration tests, critical paths

## Comparison Table

| Type | Speed | Accuracy | Complexity | When to Use |
|------|-------|----------|-----------|------------|
| Mock | Fast | Medium | High | External APIs |
| Stub | Fast | Medium | Low | Simple returns |
| Fake | Fast | High | Medium | Complex but testable |
| Real | Slow | High | Low | Integration tests |

## Cost-Benefit Analysis

```
Cost to maintain + Caught bugs + Confidence = Total value

Mock:   Low cost + Medium bugs + Low confidence = OK
Fake:   High cost + High bugs + High confidence = Good
Real:   Slowest + All bugs + Highest confidence = Best (but use sparingly)

Optimal mix: 70% Mock, 20% Fake/Integration, 10% Real
```
