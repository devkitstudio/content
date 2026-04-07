## Playwright Configuration

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',

  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },

  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },

  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
  ],
});
```

## Helper Functions

```typescript
// helpers/test-utils.ts
import { Page, expect } from '@playwright/test';

export async function fillForm(
  page: Page,
  fields: Record<string, string>
) {
  for (const [testId, value] of Object.entries(fields)) {
    await page.fill(`[data-testid="${testId}"]`, value);
  }
}

export async function submitForm(page: Page, submitButtonTestId = 'submit') {
  await Promise.all([
    page.waitForNavigation(),
    page.click(`[data-testid="${submitButtonTestId}"]`)
  ]);
}

export async function expectSuccess(page: Page, message = /success/i) {
  await expect(
    page.locator('[data-testid="success-message"]')
  ).toContainText(message);
}

export async function expectError(page: Page, message: RegExp) {
  await expect(
    page.locator('[data-testid="error-message"]')
  ).toContainText(message);
}
```

## Base Test Fixture

```typescript
// fixtures/index.ts
import { test as base, Page } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';
import { DashboardPage } from '../pages/DashboardPage';

type Fixtures = {
  authenticatedPage: Page;
  loginPage: LoginPage;
  dashboardPage: DashboardPage;
};

export const test = base.extend<Fixtures>({
  authenticatedPage: async ({ page }, use) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.fillEmail('test@example.com');
    await loginPage.fillPassword('password123');
    await loginPage.clickSubmit();

    await use(page);
  },

  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page));
  },

  dashboardPage: async ({ page }, use) => {
    await use(new DashboardPage(page));
  },
});

export { expect } from '@playwright/test';
```

## Usage in Tests

```typescript
// tests/auth.spec.ts
import { test, expect } from '../fixtures';

test.describe('Authentication', () => {
  test('should login', async ({ loginPage, dashboardPage }) => {
    await loginPage.goto();
    await loginPage.fillEmail('user@example.com');
    await loginPage.fillPassword('password');
    await loginPage.clickSubmit();

    expect(await dashboardPage.isLoaded()).toBeTruthy();
  });

  test('should show error on invalid credentials', async ({ loginPage }) => {
    await loginPage.goto();
    await loginPage.fillEmail('user@example.com');
    await loginPage.fillPassword('wrong');
    await loginPage.clickSubmit();

    expect(await loginPage.getErrorMessage()).toContain('Invalid');
  });
});
```

## Visual Regression Testing

```typescript
// tests/visual.spec.ts
import { test, expect } from '@playwright/test';

test('homepage should match snapshot', async ({ page }) => {
  await page.goto('/');
  await page.waitForLoadState('networkidle');

  await expect(page).toHaveScreenshot('homepage.png');
});

test('dashboard should match snapshot', async ({ authenticatedPage }) => {
  await authenticatedPage.goto('/dashboard');
  await expect(authenticatedPage).toHaveScreenshot('dashboard.png');
});
```

## Debugging

```typescript
// Run with debug UI
// npx playwright test --debug

// Run single test file
// npx playwright test tests/auth.spec.ts

// Run with headed browser
// npx playwright test --headed

// Generate trace for debugging
test('should work', async ({ page }) => {
  await page.context().tracing.start({ screenshots: true, snapshots: true });

  // Test code...

  await page.context().tracing.stop({ path: 'trace.zip' });
});
```
