## The Problem

```typescript
// DEEPLY NESTED (unreadable)
function processOrder(order: Order) {
  if (order) {
    if (order.items && order.items.length > 0) {
      if (order.customer) {
        if (order.customer.isActive) {
          if (order.total > 0) {
            if (order.paymentMethod) {
              // 6 levels deep!
              // Actual logic here (20 lines)
              console.log('Processing...');
              return true;
            }
          }
        }
      }
    }
  }
  return false;
}
```

**Problems:**
- 6 levels of indentation (hard to read)
- Hard to understand all conditions at once
- Buried logic makes bugs hard to find

## Solution 1: Guard Clauses (Early Return)

```typescript
// FLAT (readable)
function processOrder(order: Order): boolean {
  // Guard clauses: exit early if conditions aren't met
  if (!order) return false;
  if (!order.items || order.items.length === 0) return false;
  if (!order.customer) return false;
  if (!order.customer.isActive) return false;
  if (order.total <= 0) return false;
  if (!order.paymentMethod) return false;

  // All conditions passed, now do the work
  console.log('Processing...');
  return true;
}
```

**Benefits:**
- Single level of indentation
- Conditions at top are obvious
- Logic is isolated and clear

## Solution 2: Inverting Conditions

```typescript
// BAD (positive logic nested)
function chargeCustomer(customer: Customer) {
  if (customer.isActive) {
    if (customer.hasPaymentMethod) {
      if (customer.balance > 0) {
        charge(customer);
      }
    }
  }
}

// GOOD (invert conditions, guard clauses)
function chargeCustomer(customer: Customer) {
  if (!customer.isActive) return;
  if (!customer.hasPaymentMethod) return;
  if (customer.balance <= 0) return;

  charge(customer);
}
```

**Rule:** Guard clauses with inverted conditions flatten nesting.

## Solution 3: Extract to Functions

```typescript
// Conditions in separate functions
function validateOrder(order: Order): ValidationError | null {
  if (!order) return 'Order is required';
  if (!order.items?.length) return 'Order must have items';
  if (!order.customer) return 'Customer is required';
  if (!order.customer.isActive) return 'Customer is inactive';
  if (order.total <= 0) return 'Order total must be positive';
  if (!order.paymentMethod) return 'Payment method is required';
  return null;
}

function processOrder(order: Order) {
  const error = validateOrder(order);
  if (error) {
    console.error(error);
    return false;
  }

  // Safe to process
  charge(order);
  return true;
}
```

**Benefits:**
- Validation logic isolated
- Main function stays clean
- Easy to test validation separately

## Solution 4: Using Switch with Enums

```typescript
// NESTED (bad)
function getDiscount(userType: string, amount: number): number {
  if (userType === 'premium') {
    if (amount > 1000) {
      if (userType === 'premium_lifetime') {
        return 0.30;
      } else {
        return 0.25;
      }
    } else {
      return 0.15;
    }
  } else if (userType === 'standard') {
    if (amount > 500) {
      return 0.10;
    } else {
      return 0.05;
    }
  }
  return 0;
}

// FLAT (with switch)
type UserType = 'standard' | 'premium' | 'premium_lifetime';

function getDiscount(userType: UserType, amount: number): number {
  const tier = getTierForUser(userType);

  switch (tier) {
    case 'premium_lifetime':
      return amount > 1000 ? 0.30 : 0.25;
    case 'premium':
      return amount > 1000 ? 0.25 : 0.15;
    case 'standard':
      return amount > 500 ? 0.10 : 0.05;
    default:
      return 0;
  }
}

function getTierForUser(userType: UserType): string {
  return userType;
}
```

## Solution 5: Ternary for Simple Cases

```typescript
// Multiple levels of ternary (bad)
const status = user
  ? user.isActive
    ? user.subscription
      ? user.subscription.isValid
        ? 'active'
        : 'expired'
      : 'no_subscription'
    : 'inactive'
  : 'unknown';

// With guards (good)
function getUserStatus(user: User | null): string {
  if (!user) return 'unknown';
  if (!user.isActive) return 'inactive';
  if (!user.subscription) return 'no_subscription';
  if (!user.subscription.isValid) return 'expired';
  return 'active';
}

const status = getUserStatus(user);
```

## Complex Example: Full Refactoring

```typescript
// BEFORE: Deeply nested
function approveCredit(customer: Customer, amount: number): ApprovalResult {
  if (customer) {
    if (customer.creditScore >= 650) {
      if (customer.income >= amount / 10) {
        if (!customer.hasDelinquency) {
          if (customer.existingDebt < amount) {
            if (customer.yearsAsCustomer >= 1) {
              return {
                approved: true,
                creditLimit: calculateLimit(customer, amount)
              };
            }
          }
        }
      }
    }
  }
  return { approved: false, reason: 'Unknown' };
}

// AFTER: Extracted & flattened
interface ApprovalResult {
  approved: boolean;
  reason?: string;
  creditLimit?: number;
}

function validateCreditApplication(
  customer: Customer,
  amount: number
): string | null {
  // Guard clauses with clear error messages
  if (!customer) return 'Customer not found';
  if (customer.creditScore < 650) return 'Credit score too low';
  if (customer.income < amount / 10) return 'Income insufficient';
  if (customer.hasDelinquency) return 'Customer has delinquencies';
  if (customer.existingDebt >= amount) return 'Existing debt too high';
  if (customer.yearsAsCustomer < 1) return 'New customer';
  return null;
}

function approveCredit(customer: Customer, amount: number): ApprovalResult {
  const reason = validateCreditApplication(customer, amount);

  if (reason) {
    return { approved: false, reason };
  }

  return {
    approved: true,
    creditLimit: calculateLimit(customer, amount)
  };
}
```

**Improvements:**
- Guard clauses at top level
- Each condition has clear error message
- Logic separated from validation
- Easy to extend with new rules
