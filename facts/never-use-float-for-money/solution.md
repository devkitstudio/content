## Why Float Breaks Money

IEEE 754 floating-point can't represent decimal fractions exactly.

```javascript
// Examples of the problem
0.1 + 0.2 === 0.3  // false!
0.1 + 0.2          // 0.30000000000000004

// Real money example
19.99 + 0.01       // 20.00... wait, let's check
console.log(19.99 + 0.01)  // 20.000000000000004

// Accumulating error
let total = 0;
for (let i = 0; i < 100; i++) {
  total += 0.1;
}
console.log(total)  // 9.99999999999998 (not 10!)
```

**Why?** Float stores as binary fractions. 0.1 in binary is infinite:
```
0.1 (decimal) = 0.0001100110011... (binary, repeating)
→ Can't store exactly
→ Rounds to nearest representable value
→ 0.1 + 0.2 ≠ 0.3
```

## Correct Solutions

### 1. DECIMAL/NUMERIC (Recommended)

```sql
-- PostgreSQL
CREATE TABLE products (
  id BIGINT PRIMARY KEY,
  name VARCHAR,
  price DECIMAL(10, 2)  -- 10 digits, 2 after decimal
);

-- MySQL
CREATE TABLE products (
  id INT PRIMARY KEY,
  name VARCHAR(255),
  price DECIMAL(10, 2)
);

-- SQL Server
CREATE TABLE products (
  id INT PRIMARY KEY,
  name VARCHAR(MAX),
  price NUMERIC(10, 2)
);

-- Insert and arithmetic
INSERT INTO products VALUES (1, 'Item', 19.99);
SELECT price + 0.01 FROM products;  -- Exact: 20.00

-- Aggregate functions
SELECT SUM(price) FROM orders;  -- Exact sum
```

**Why DECIMAL?**
- Arbitrary precision (no rounding errors)
- Exact decimal arithmetic
- Designed for financial data
- Slightly slower (negligible)

### 2. Integer Cents (Minimal Storage)

Store as integer cents instead of decimal dollars:

```typescript
// Bad: Store $19.99 as FLOAT
const price = 19.99;  // Imprecise

// Good: Store as integer cents (1999 = $19.99)
const priceCents = 1999;

// Convert for display
const displayPrice = (priceCents / 100).toFixed(2);  // "$19.99"

// Arithmetic is exact
const priceCents = 1999;
const total = priceCents + 1;  // 2000 cents = $20.00 (exact)
```

**Database:**
```sql
CREATE TABLE products (
  id BIGINT PRIMARY KEY,
  price_cents BIGINT  -- Store 1999 for $19.99
);

-- Display in application
SELECT price_cents / 100 as dollars FROM products;
```

**Advantages:**
- No rounding errors
- Smaller storage (BIGINT)
- Fast arithmetic
- Standard in financial systems

### 3. String Representation (Last Resort)

```typescript
// For unstructured data or migration
const price = "19.99";

// Convert to Decimal for calculations
const Decimal = require('decimal.js');
const price1 = new Decimal('19.99');
const price2 = new Decimal('0.01');
const total = price1.plus(price2);  // Exact: 20.00
```

## Real-World Bugs (Float for Money)

### Bug 1: Account Reconciliation

```
Customer: "I was charged $19.990000000001!"

Account A: 100.00 (FLOAT)
Account B: -100.00 (FLOAT)

SELECT (100.00::float + -100.00::float) = 0;  -- false!
Result: 9.99200000000001e-14

Balance doesn't zero out → accounting error
```

### Bug 2: Tax Calculation

```
Price: 19.99 (FLOAT)
Tax (8.875%): 19.99 * 0.08875 = ?

PostgreSQL: 19.99::float * 0.08875  = 1.77486249999999...
Charged: $1.77 (rounded)
Expected: $1.77
But: 19.99 * 0.08875 exact = 1.7748625
→ Rounding inconsistency

Over 1M transactions:
Off by ~0.0001 per transaction × 1M = $100+ error
```

### Bug 3: Totals Don't Match

```
Invoice items:
Item 1: $19.99
Item 2: $19.99
Item 3: $19.99
Manual total: $59.97

Database SUM(FLOAT): 59.97000000000001
Doesn't match → audit failure
```

## Migration Guide

```sql
-- Add new DECIMAL column
ALTER TABLE products ADD price_decimal DECIMAL(10, 2);

-- Migrate data carefully
UPDATE products
SET price_decimal = ROUND(price::numeric, 2)
WHERE price IS NOT NULL;

-- Verify no data loss
SELECT COUNT(*) FROM products
WHERE ABS(price_decimal - price) > 0.01;  -- Should be 0

-- Drop old column
ALTER TABLE products DROP price;
ALTER TABLE products RENAME price_decimal TO price;
```

## Application Layer

```typescript
// Always use Decimal library
import Decimal from 'decimal.js';

function calculateTotal(items: Item[]): Decimal {
  return items.reduce(
    (sum, item) => sum.plus(new Decimal(item.price)),
    new Decimal(0)
  );
}

// Store in DB as DECIMAL
async function savePrice(price: Decimal) {
  await db.products.update({
    price: price.toString()  // "19.99"
  });
}
```
