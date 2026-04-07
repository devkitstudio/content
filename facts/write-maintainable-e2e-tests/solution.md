## Anti-Pattern: Brittle Tests

```typescript
// WRONG: Relies on implementation details
it('should create user', async () => {
  await page.click('div.container > section > button.btn-primary');
  // ^^^ CSS changes → test breaks

  await page.fill('input[type="email"][placeholder="Enter email"]', 'user@example.com');
  // ^^^ Placeholder changes → test breaks

  await page.waitForNavigation({ timeout: 5000 });
  // ^^^ Hard-coded timeout → flaky
});
```

## Pattern 1: Page Object Model (POM)

Centralize selectors and actions.

```typescript
// pages/LoginPage.ts
export class LoginPage {
  constructor(private page: Page) {}

  async goto() {
    await this.page.goto('/login');
  }

  async fillEmail(email: string) {
    await this.page.fill('[data-testid="email-input"]', email);
  }

  async fillPassword(password: string) {
    await this.page.fill('[data-testid="password-input"]', password);
  }

  async clickSubmit() {
    await this.page.click('[data-testid="login-submit"]');
  }

  async getErrorMessage(): Promise<string> {
    return this.page.locator('[data-testid="error-message"]').textContent();
  }
}

// pages/DashboardPage.ts
export class DashboardPage {
  constructor(private page: Page) {}

  async isLoaded() {
    return this.page.locator('[data-testid="dashboard-header"]').isVisible();
  }

  async getUserName(): Promise<string> {
    return this.page.locator('[data-testid="user-name"]').textContent();
  }
}
```

**Test using POM:**

```typescript
import { LoginPage } from './pages/LoginPage';
import { DashboardPage } from './pages/DashboardPage';

test('should login successfully', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.fillEmail('user@example.com');
  await loginPage.fillPassword('password123');
  await loginPage.clickSubmit();

  const dashboardPage = new DashboardPage(page);
  expect(await dashboardPage.isLoaded()).toBeTruthy();
  expect(await dashboardPage.getUserName()).toBe('User');
});
```

**Benefit:** If CSS changes, update only the page object, not 50 tests.

## Pattern 2: data-testid Attributes

Use semantic identifiers instead of CSS.

```html
<!-- HTML -->
<form>
  <input data-testid="email-input" />
  <input data-testid="password-input" />
  <button data-testid="login-submit">Sign In</button>
  <span data-testid="error-message" role="alert"></span>
</form>
```

```typescript
// Selectors are explicit and stable
page.fill('[data-testid="email-input"]', 'user@example.com');
page.click('[data-testid="login-submit"]');
```

**Advantages:**
- Separates test selectors from CSS
- CSS changes don't break tests
- Easy to audit which elements are tested

## Pattern 3: Semantic Queries

Use accessible selectors (role-based).

```typescript
// BAD: CSS-brittle
await page.click('.btn.btn-primary');

// GOOD: Role-based (accessible)
await page.click('button:has-text("Submit")');

// BETTER: Using getByRole (most resilient)
await page.click(page.getByRole('button', { name: /submit/i }));
```

**Why:** `getByRole` queries how users actually interact with the page.

## Pattern 4: Smart Waits

Avoid hard-coded timeouts.

```typescript
// WRONG: Flaky
await page.waitForTimeout(2000);
await page.click('button');

// GOOD: Wait for element
await page.waitForSelector('[data-testid="submit-btn"]', { timeout: 5000 });
await page.click('[data-testid="submit-btn"]');

// BETTER: Wait and navigate
await Promise.all([
  page.waitForNavigation(),
  page.click('[data-testid="submit-btn"]')
]);

// BEST: Wait for actual state
await page.locator('[data-testid="success-message"]').waitFor();
expect(await page.locator('[data-testid="success-message"]').isVisible()).toBeTruthy();
```

## Pattern 5: Test Fixtures & Helpers

```typescript
// fixtures/auth.ts
export const authFixture = {
  async login(page: Page, email = 'user@example.com', password = 'password123') {
    await page.goto('/login');
    await page.fill('[data-testid="email-input"]', email);
    await page.fill('[data-testid="password-input"]', password);
    await page.click('[data-testid="login-submit"]');
    await page.waitForURL('/dashboard');
  }
};

// tests/checkout.spec.ts
test('should complete checkout', async ({ page, context }) => {
  await authFixture.login(page);
  // User is logged in, proceed with test
});
```

## Complete Example

```typescript
// pages/ProductPage.ts
export class ProductPage {
  constructor(private page: Page) {}

  async addToCart(productName: string) {
    const product = this.page.getByRole('heading', { name: productName });
    const addBtn = product.locator('..').getByRole('button', { name: /add to cart/i });
    await addBtn.click();
  }

  async goToCart() {
    await this.page.getByRole('link', { name: /cart/i }).click();
  }

  async getCartCount(): Promise<number> {
    const badge = this.page.locator('[data-testid="cart-badge"]');
    const text = await badge.textContent();
    return parseInt(text || '0');
  }
}

// pages/CartPage.ts
export class CartPage {
  constructor(private page: Page) {}

  async checkout() {
    await this.page.getByRole('button', { name: /checkout/i }).click();
  }

  async getTotal(): Promise<string> {
    return this.page.locator('[data-testid="cart-total"]').textContent();
  }
}

// tests/shopping.spec.ts
test('should add item to cart', async ({ page }) => {
  const productPage = new ProductPage(page);
  await page.goto('/products');

  await productPage.addToCart('Laptop');
  expect(await productPage.getCartCount()).toBe(1);

  const cartPage = new CartPage(page);
  await productPage.goToCart();
  expect(await cartPage.getTotal()).toContain('$999');
});
```
