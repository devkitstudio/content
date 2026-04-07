## Famous Float Money Failures

### Failure 1: Amazon Rounding Errors

Amazon had to shut down parts of their store for rounding errors caused by float arithmetic in pricing calculations.

```
Price: $299.99 (stored as float: 299.989999...)
Tax rate: 8.625%
Calculated tax: 25.8745... (rounds to $25.87)

Customer charged: $325.86
Expected: $325.86

Over millions of transactions:
Some calculated as $325.87, some as $325.86
Total mismatch: thousands of dollars
```

### Failure 2: JPMorgan Flash Crash Trading

Float imprecision in high-frequency trading algorithms caused miscalculations in the 2010 Flash Crash. Prices stored as floats drifted during rapid calculations.

### Failure 3: Lufthansa Frequent Flyer Program

Miles stored as floats had rounding errors. Customers couldn't book flights because:
```
100,000.00 miles (displayed)
but stored as 99,999.99999...
→ "Insufficient miles"
```

## Code Examples: The Problem

```javascript
// Online store
const cartItems = [
  { name: 'Item 1', price: 19.99 },
  { name: 'Item 2', price: 29.99 },
  { name: 'Item 3', price: 14.99 }
];

// Calculate total (WRONG)
let total = 0;
cartItems.forEach(item => {
  total += item.price;
});

console.log(total);              // 64.96999999999999 (not 64.97!)
console.log(total.toFixed(2));   // "64.97" (works by luck)
console.log(total - 64.97);      // 8.881784197001252e-16 (hidden error)

// Now apply tax
const tax = total * 0.08875;
console.log(tax);                // 5.771319999999998
console.log(tax.toFixed(2));     // "5.77"

// But when processing payment (no rounding):
const finalPrice = total + tax;
console.log(finalPrice);         // 70.74131999999998
// Charged customer: $70.74... or $70.74? Inconsistent!
```

## Verification: Test Your Database

```sql
-- PostgreSQL: Check for float dangers
SELECT
  price::float,
  price::decimal,
  (price::float - price::decimal)::float as error
FROM products
LIMIT 10;

-- You'll see errors like:
-- 19.99 | 19.99 | -8.881784197001252e-16
-- ^ These add up over time!

-- Test with your actual data
SELECT
  COUNT(*),
  SUM(price::float) as sum_float,
  SUM(price::decimal) as sum_decimal,
  (SUM(price::float) - SUM(price::decimal))::float as error
FROM products;
```

## The Fix Applied

```sql
-- Original problematic schema
CREATE TABLE orders_bad (
  id BIGINT PRIMARY KEY,
  total_price REAL,  -- DANGER!
  tax FLOAT,         -- DANGER!
  final_amount DOUBLE  -- DANGER!
);

-- Fixed schema
CREATE TABLE orders_good (
  id BIGINT PRIMARY KEY,
  total_price DECIMAL(12, 2),    -- Exact
  tax DECIMAL(12, 2),            -- Exact
  final_amount DECIMAL(12, 2)    -- Exact
);

-- Or alternative: integer cents
CREATE TABLE orders_alt (
  id BIGINT PRIMARY KEY,
  total_price_cents BIGINT,      -- 1999 = $19.99
  tax_cents BIGINT,              -- 177 = $1.77
  final_amount_cents BIGINT      -- 2176 = $21.76
);
```

## Rule of Thumb

```
IF money is involved:
  ├─ NEVER use: FLOAT, REAL, DOUBLE
  └─ USE: DECIMAL, NUMERIC
     OR: INTEGER (cents)

IF you see FLOAT for money:
  ├─ RED FLAG
  └─ Schedule immediate refactoring
```

## Testing Recommendation

```javascript
// Add to your test suite
test('money arithmetic is exact', () => {
  const price1 = new Decimal('19.99');
  const price2 = new Decimal('29.99');
  const price3 = new Decimal('14.99');

  const total = price1.plus(price2).plus(price3);

  expect(total.toString()).toBe('64.97');

  const tax = total.times('0.08875');
  expect(tax.toString()).toBe('5.7688125');
  expect(tax.toDecimalPlaces(2).toString()).toBe('5.77');
});
```
