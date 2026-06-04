# Playwright - End-to-End Testing

## Overview

Playwright is a modern end-to-end testing framework developed by Microsoft that enables reliable testing across all major browsers (Chromium, Firefox, WebKit). It provides a powerful API for automating browser interactions, supports multiple languages, includes built-in test runner, and offers advanced features like auto-waiting, network interception, and parallel execution.

## Installation and Setup

```bash
# Install Playwright
npm init playwright@latest

# Or add to existing project
npm install -D @playwright/test

# Install browsers
npx playwright install

# Install specific browsers
npx playwright install chromium firefox webkit

# With TypeScript
npm install -D @playwright/test typescript
```

### Basic Configuration

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ['html'],
    ['junit', { outputFile: 'test-results/junit.xml' }],
    ['json', { outputFile: 'test-results/results.json' }],
  ],
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
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
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] },
    },
    {
      name: 'Mobile Safari',
      use: { ...devices['iPhone 12'] },
    },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

## Core Testing Features

### Basic Test Structure

```typescript
import { test, expect } from '@playwright/test';

// Basic test
test('homepage loads correctly', async ({ page }) => {
  await page.goto('/');
  await expect(page).toHaveTitle(/Home/);
  await expect(page.getByRole('heading', { name: 'Welcome' })).toBeVisible();
});

// Test with describe blocks
test.describe('User authentication', () => {
  test('successful login', async ({ page }) => {
    await page.goto('/login');
    await page.getByLabel('Email').fill('user@example.com');
    await page.getByLabel('Password').fill('password123');
    await page.getByRole('button', { name: 'Login' }).click();
    
    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByText('Welcome back')).toBeVisible();
  });

  test('failed login shows error', async ({ page }) => {
    await page.goto('/login');
    await page.getByLabel('Email').fill('wrong@example.com');
    await page.getByLabel('Password').fill('wrongpass');
    await page.getByRole('button', { name: 'Login' }).click();
    
    await expect(page.getByText('Invalid credentials')).toBeVisible();
  });
});

// Nested describe blocks
test.describe('Shopping cart', () => {
  test.describe('Adding items', () => {
    test('adds single item', async ({ page }) => {
      await page.goto('/products');
      await page.getByTestId('product-1').click();
      await page.getByRole('button', { name: 'Add to Cart' }).click();
      
      await expect(page.getByTestId('cart-count')).toHaveText('1');
    });

    test('adds multiple items', async ({ page }) => {
      await page.goto('/products');
      await page.getByTestId('product-1').click();
      await page.getByRole('button', { name: 'Add to Cart' }).click();
      await page.goto('/products');
      await page.getByTestId('product-2').click();
      await page.getByRole('button', { name: 'Add to Cart' }).click();
      
      await expect(page.getByTestId('cart-count')).toHaveText('2');
    });
  });
});

// Parameterized tests
const credentials = [
  { email: 'admin@test.com', password: 'admin123', role: 'admin' },
  { email: 'user@test.com', password: 'user123', role: 'user' },
];

for (const { email, password, role } of credentials) {
  test(`login as ${role}`, async ({ page }) => {
    await page.goto('/login');
    await page.getByLabel('Email').fill(email);
    await page.getByLabel('Password').fill(password);
    await page.getByRole('button', { name: 'Login' }).click();
    
    await expect(page.getByTestId('user-role')).toHaveText(role);
  });
}
```

### Locators and Selectors

```typescript
import { test, expect } from '@playwright/test';

test('various locator strategies', async ({ page }) => {
  await page.goto('/');

  // Role-based locators (recommended)
  await page.getByRole('button', { name: 'Submit' }).click();
  await page.getByRole('heading', { name: 'Welcome' }).isVisible();
  await page.getByRole('textbox', { name: 'Email' }).fill('test@example.com');
  
  // Text locators
  await page.getByText('Click here').click();
  await page.getByText(/case insensitive/i).click();
  
  // Label locators
  await page.getByLabel('Username').fill('john');
  await page.getByLabel(/email/i).fill('john@example.com');
  
  // Placeholder locators
  await page.getByPlaceholder('Enter your email').fill('test@example.com');
  
  // Test ID locators
  await page.getByTestId('submit-button').click();
  await page.getByTestId('user-profile').isVisible();
  
  // CSS selectors
  await page.locator('.submit-btn').click();
  await page.locator('#email-input').fill('test@example.com');
  await page.locator('[data-testid="header"]').isVisible();
  
  // XPath selectors
  await page.locator('xpath=//button[text()="Submit"]').click();
  
  // Chaining locators
  await page
    .locator('.modal')
    .getByRole('button', { name: 'Close' })
    .click();
  
  // Filtering locators
  await page
    .getByRole('listitem')
    .filter({ hasText: 'Important' })
    .click();
  
  // Nth element
  await page.getByRole('button').nth(2).click();
  await page.getByRole('button').first().click();
  await page.getByRole('button').last().click();
});

// Locator chaining
test('complex locator chains', async ({ page }) => {
  await page.goto('/products');

  // Find product card, then button within it
  const productCard = page.getByTestId('product-card-1');
  await productCard.getByRole('button', { name: 'Add to Cart' }).click();
  
  // Filter by text content
  const activeItems = page.getByRole('listitem').filter({ hasText: 'Active' });
  await expect(activeItems).toHaveCount(3);
  
  // Has selector
  const cardsWithButtons = page.locator('.card').filter({
    has: page.locator('button'),
  });
  await expect(cardsWithButtons).toHaveCount(5);
});
```

### Assertions

```typescript
import { test, expect } from '@playwright/test';

test('various assertions', async ({ page }) => {
  await page.goto('/');

  // Visibility assertions
  await expect(page.getByText('Welcome')).toBeVisible();
  await expect(page.getByText('Hidden')).toBeHidden();
  
  // Text assertions
  await expect(page.getByRole('heading')).toHaveText('Welcome');
  await expect(page.getByRole('heading')).toContainText('Wel');
  
  // Value assertions
  await expect(page.getByLabel('Email')).toHaveValue('test@example.com');
  await expect(page.getByLabel('Email')).not.toBeEmpty();
  
  // Attribute assertions
  await expect(page.getByRole('link')).toHaveAttribute('href', '/about');
  await expect(page.getByRole('button')).toHaveClass(/btn-primary/);
  await expect(page.getByTestId('logo')).toHaveAttribute('alt', 'Company Logo');
  
  // State assertions
  await expect(page.getByRole('button')).toBeEnabled();
  await expect(page.getByRole('button')).toBeDisabled();
  await expect(page.getByRole('checkbox')).toBeChecked();
  await expect(page.getByRole('checkbox')).not.toBeChecked();
  
  // Count assertions
  await expect(page.getByRole('listitem')).toHaveCount(5);
  
  // URL assertions
  await expect(page).toHaveURL('/dashboard');
  await expect(page).toHaveURL(/\/dashboard/);
  
  // Title assertions
  await expect(page).toHaveTitle('Dashboard');
  await expect(page).toHaveTitle(/Dashboard/);
  
  // Screenshot assertions
  await expect(page).toHaveScreenshot('homepage.png');
  await expect(page.getByTestId('modal')).toHaveScreenshot();
  
  // Custom timeout
  await expect(page.getByText('Loading...')).toBeVisible({ timeout: 10000 });
});
```

### Setup and Teardown

```typescript
import { test, expect } from '@playwright/test';

test.describe('User management', () => {
  // Runs before all tests
  test.beforeAll(async ({ browser }) => {
    const context = await browser.newContext();
    const page = await context.newPage();
    await page.goto('/setup');
    await page.getByRole('button', { name: 'Initialize' }).click();
    await context.close();
  });

  // Runs before each test
  test.beforeEach(async ({ page }) => {
    await page.goto('/login');
    await page.getByLabel('Email').fill('admin@test.com');
    await page.getByLabel('Password').fill('admin123');
    await page.getByRole('button', { name: 'Login' }).click();
    await expect(page).toHaveURL('/dashboard');
  });

  // Runs after each test
  test.afterEach(async ({ page }) => {
    await page.getByRole('button', { name: 'Logout' }).click();
  });

  // Runs after all tests
  test.afterAll(async ({ browser }) => {
    const context = await browser.newContext();
    const page = await context.newPage();
    await page.goto('/cleanup');
    await page.getByRole('button', { name: 'Clean' }).click();
    await context.close();
  });

  test('creates user', async ({ page }) => {
    await page.goto('/users/new');
    await page.getByLabel('Name').fill('John Doe');
    await page.getByLabel('Email').fill('john@test.com');
    await page.getByRole('button', { name: 'Create' }).click();
    
    await expect(page.getByText('User created')).toBeVisible();
  });

  test('edits user', async ({ page }) => {
    await page.goto('/users/1');
    await page.getByRole('button', { name: 'Edit' }).click();
    await page.getByLabel('Name').fill('Jane Doe');
    await page.getByRole('button', { name: 'Save' }).click();
    
    await expect(page.getByText('User updated')).toBeVisible();
  });
});
```

## Advanced Features

### Fixtures and Page Objects

```typescript
// fixtures/auth.fixture.ts
import { test as base } from '@playwright/test';

type AuthFixtures = {
  authenticatedPage: Page;
};

export const test = base.extend<AuthFixtures>({
  authenticatedPage: async ({ page }, use) => {
    // Login before test
    await page.goto('/login');
    await page.getByLabel('Email').fill('test@example.com');
    await page.getByLabel('Password').fill('password123');
    await page.getByRole('button', { name: 'Login' }).click();
    await page.waitForURL('/dashboard');
    
    // Use the authenticated page in test
    await use(page);
    
    // Cleanup after test
    await page.goto('/logout');
  },
});

// pages/LoginPage.ts
import { Page, Locator } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly loginButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel('Email');
    this.passwordInput = page.getByLabel('Password');
    this.loginButton = page.getByRole('button', { name: 'Login' });
    this.errorMessage = page.getByTestId('error-message');
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.loginButton.click();
  }

  async hasError(message: string) {
    await this.errorMessage.waitFor({ state: 'visible' });
    return await this.errorMessage.textContent() === message;
  }
}

// Usage
import { test, expect } from './fixtures/auth.fixture';
import { LoginPage } from './pages/LoginPage';

test('login with page object', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login('test@example.com', 'password123');
  
  await expect(page).toHaveURL('/dashboard');
});
```

### Network Interception

```typescript
import { test, expect } from '@playwright/test';

test('mock API responses', async ({ page }) => {
  // Mock successful response
  await page.route('/api/users', async (route) => {
    await route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify([
        { id: 1, name: 'John Doe', email: 'john@test.com' },
        { id: 2, name: 'Jane Doe', email: 'jane@test.com' },
      ]),
    });
  });

  await page.goto('/users');
  await expect(page.getByText('John Doe')).toBeVisible();
  await expect(page.getByText('Jane Doe')).toBeVisible();
});

test('mock API error', async ({ page }) => {
  await page.route('/api/users', async (route) => {
    await route.fulfill({
      status: 500,
      contentType: 'application/json',
      body: JSON.stringify({ error: 'Internal server error' }),
    });
  });

  await page.goto('/users');
  await expect(page.getByText('Failed to load users')).toBeVisible();
});

test('modify API response', async ({ page }) => {
  await page.route('/api/users', async (route) => {
    const response = await route.fetch();
    const json = await response.json();
    
    // Modify response data
    json.forEach((user: any) => {
      user.email = `modified_${user.email}`;
    });
    
    await route.fulfill({ response, json });
  });

  await page.goto('/users');
  await expect(page.getByText(/modified_/)).toBeVisible();
});

test('wait for API request', async ({ page }) => {
  const responsePromise = page.waitForResponse('/api/users');
  
  await page.goto('/users');
  
  const response = await responsePromise;
  expect(response.status()).toBe(200);
  
  const data = await response.json();
  expect(data).toHaveLength(2);
});

test('intercept and log requests', async ({ page }) => {
  const requests: string[] = [];
  
  page.on('request', (request) => {
    requests.push(request.url());
  });
  
  await page.goto('/');
  
  expect(requests).toContain('https://example.com/api/data');
});
```

### Browser Contexts and Parallel Testing

```typescript
import { test, expect } from '@playwright/test';

// Each test gets isolated context
test('isolated contexts', async ({ browser }) => {
  const context1 = await browser.newContext();
  const page1 = await context1.newPage();
  await page1.goto('/');
  
  const context2 = await browser.newContext();
  const page2 = await context2.newPage();
  await page2.goto('/');
  
  // Contexts are completely isolated
  await page1.evaluate(() => localStorage.setItem('user', 'user1'));
  await page2.evaluate(() => localStorage.setItem('user', 'user2'));
  
  const user1 = await page1.evaluate(() => localStorage.getItem('user'));
  const user2 = await page2.evaluate(() => localStorage.getItem('user'));
  
  expect(user1).toBe('user1');
  expect(user2).toBe('user2');
  
  await context1.close();
  await context2.close();
});

// Persistent context (saves cookies, localStorage)
test('persistent context', async ({ browser }) => {
  const context = await browser.newContext({
    storageState: 'state.json', // Load saved state
  });
  
  const page = await context.newPage();
  await page.goto('/dashboard'); // Already logged in
  
  await expect(page.getByText('Welcome back')).toBeVisible();
  
  await context.close();
});

// Save auth state
test('save authentication', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('test@example.com');
  await page.getByLabel('Password').fill('password123');
  await page.getByRole('button', { name: 'Login' }).click();
  
  await page.context().storageState({ path: 'auth.json' });
});

// Run tests in parallel
test.describe.configure({ mode: 'parallel' });

test.describe('Parallel tests', () => {
  test('test 1', async ({ page }) => {
    await page.goto('/test1');
    // Test logic
  });

  test('test 2', async ({ page }) => {
    await page.goto('/test2');
    // Test logic
  });

  test('test 3', async ({ page }) => {
    await page.goto('/test3');
    // Test logic
  });
});

// Serial tests (one after another)
test.describe.configure({ mode: 'serial' });

test.describe('Serial tests', () => {
  let sharedData: any;

  test('step 1', async ({ page }) => {
    sharedData = await page.evaluate(() => /* ... */);
  });

  test('step 2', async ({ page }) => {
    // Use sharedData from previous test
  });
});
```

### Trace Viewer and Debugging

```typescript
import { test, expect } from '@playwright/test';

// Enable tracing
test.use({
  trace: 'on', // 'on' | 'off' | 'on-first-retry' | 'retain-on-failure'
});

test('traced test', async ({ page }) => {
  await page.goto('/');
  await page.getByRole('button', { name: 'Click' }).click();
  await expect(page.getByText('Clicked')).toBeVisible();
});

// Debug mode
test('debugging', async ({ page }) => {
  // Pause execution
  await page.pause();
  
  // Step through actions
  await page.getByRole('button').click();
  
  // Continue execution
});

// Screenshots
test('take screenshots', async ({ page }) => {
  await page.goto('/');
  
  // Full page screenshot
  await page.screenshot({ path: 'screenshot.png', fullPage: true });
  
  // Element screenshot
  await page.getByTestId('modal').screenshot({ path: 'modal.png' });
});

// Video recording (configured in playwright.config.ts)
test.use({
  video: 'on', // 'on' | 'off' | 'retain-on-failure' | 'on-first-retry'
});

test('recorded test', async ({ page }) => {
  await page.goto('/');
  // Test actions are recorded
});
```

### Mobile and Device Emulation

```typescript
import { test, expect, devices } from '@playwright/test';

// Emulate mobile device
test.use({
  ...devices['iPhone 12'],
});

test('mobile test', async ({ page }) => {
  await page.goto('/');
  
  // Test mobile-specific UI
  await page.getByRole('button', { name: 'Menu' }).click();
  await expect(page.getByRole('navigation')).toBeVisible();
});

// Custom viewport
test.use({
  viewport: { width: 375, height: 667 },
  deviceScaleFactor: 2,
  isMobile: true,
  hasTouch: true,
});

test('custom viewport', async ({ page }) => {
  await page.goto('/');
  await expect(page).toHaveScreenshot('mobile-view.png');
});

// Geolocation
test('geolocation', async ({ context, page }) => {
  await context.setGeolocation({ latitude: 37.7749, longitude: -122.4194 });
  await context.grantPermissions(['geolocation']);
  
  await page.goto('/location');
  await expect(page.getByText('San Francisco')).toBeVisible();
});

// Orientation
test('landscape mode', async ({ page }) => {
  await page.setViewportSize({ width: 667, height: 375 });
  
  await page.goto('/');
  await expect(page.getByTestId('landscape-view')).toBeVisible();
});
```

## Common Mistakes

1. **Not using auto-waiting**: Adding unnecessary `waitFor` when Playwright auto-waits.

2. **Poor locator strategies**: Using CSS selectors instead of semantic locators.

3. **Not isolating tests**: Tests affecting each other through shared state.

4. **Synchronous thinking**: Not awaiting async operations properly.

5. **Missing error handling**: Not checking for error states and edge cases.

6. **Over-reliance on timeouts**: Using fixed timeouts instead of waiting for specific conditions.

7. **Not using fixtures**: Duplicating setup code instead of creating reusable fixtures.

8. **Ignoring trace/debug tools**: Not leveraging Playwright's debugging capabilities.

9. **Testing implementation details**: Coupling tests to implementation instead of behavior.

10. **Not cleaning up contexts**: Memory leaks from unclosed browser contexts.

## Best Practices

1. **Use semantic locators**: Prefer role-based and label locators over CSS selectors.

2. **Leverage auto-waiting**: Trust Playwright's auto-waiting instead of manual waits.

3. **Create page objects**: Organize selectors and actions into page object models.

4. **Use fixtures**: Extract common setup into reusable fixtures.

5. **Isolate tests**: Ensure tests can run independently in any order.

6. **Enable tracing**: Use trace viewer for debugging test failures.

7. **Mock external dependencies**: Intercept and mock API calls for reliability.

8. **Test mobile viewports**: Include mobile device testing in your suite.

9. **Parallelize wisely**: Use parallel execution but be aware of resource constraints.

10. **Document test flows**: Add descriptive test names and comments.

## When to Use Playwright

**Use Playwright when:**
- Need cross-browser E2E testing
- Want modern, reliable test automation
- Need mobile/device emulation
- Want built-in debugging tools
- Need network interception
- Building modern web applications
- Want parallel test execution

**Consider alternatives when:**
- Only need Chrome testing (use Puppeteer)
- Need GUI test recorder (use Cypress)
- Building component tests (use Vitest/Jest)
- Need visual regression testing only (use Percy)

## Interview Questions

### Q1: What makes Playwright more reliable than Selenium?

**Answer**: Playwright achieves better reliability through: 1) Auto-waiting for elements to be actionable, 2) Web-first assertions with automatic retries, 3) Browser context isolation preventing test pollution, 4) Built-in network interception and mocking, 5) Tracing and debugging tools, 6) Modern architecture using CDP/WDP protocols, 7) No flaky waits required. This results in more stable, maintainable tests.

### Q2: How does Playwright handle element waiting differently than Selenium?

**Answer**: Playwright automatically waits for elements to be actionable (visible, enabled, stable) before performing actions. It has built-in auto-waiting with web-first assertions that retry until conditions are met. Selenium requires explicit waits. Example: `await page.click('button')` in Playwright automatically waits for the button to be clickable, while Selenium needs `WebDriverWait`.

### Q3: What are Playwright fixtures and when should you use them?

**Answer**: Fixtures are a way to set up test environment and provide dependencies to tests. They handle setup/teardown automatically and can be composed. Use fixtures for: authentication state, page objects, database connections, or any reusable test setup. They're superior to beforeEach because they're lazy-loaded and have better composition.

### Q4: How do you implement Page Object Model in Playwright?

**Answer**: Create classes that encapsulate page elements and actions:

```typescript
class LoginPage {
  constructor(private page: Page) {}
  
  async login(email: string, password: string) {
    await this.page.getByLabel('Email').fill(email);
    await this.page.getByLabel('Password').fill(password);
    await this.page.getByRole('button', { name: 'Login' }).click();
  }
}
```

This centralizes selectors and actions for maintainability.

### Q5: How does network interception work in Playwright?

**Answer**: Use `page.route()` to intercept requests and control responses:

```typescript
await page.route('/api/users', (route) => {
  route.fulfill({
    status: 200,
    body: JSON.stringify([{ id: 1, name: 'Test' }]),
  });
});
```

You can mock responses, modify real responses, or abort requests. This enables testing without backend dependencies.

### Q6: What's the difference between browser contexts and pages in Playwright?

**Answer**: A browser context is an isolated browser session with its own cookies, localStorage, and cache - like an incognito window. Pages are tabs within a context. Contexts provide test isolation without the overhead of launching new browsers. Use separate contexts for testing different user sessions in parallel.

### Q7: How do you debug failing Playwright tests?

**Answer**: Playwright offers multiple debugging approaches: 1) `await page.pause()` to pause execution, 2) `--debug` flag to run in headed mode with inspector, 3) Trace viewer with `trace: 'on'` to see timeline of actions, 4) Screenshots with `screenshot: 'only-on-failure'`, 5) Video recording with `video: 'retain-on-failure'`, 6) VS Code extension with breakpoint debugging.

## Key Takeaways

1. Playwright provides cross-browser testing for Chromium, Firefox, and WebKit with a unified API.

2. Auto-waiting automatically waits for elements to be actionable, eliminating flaky tests from timing issues.

3. Browser contexts provide isolated sessions enabling parallel test execution without interference.

4. Page Object Model pattern organizes selectors and actions into maintainable, reusable classes.

5. Fixtures provide powerful dependency injection and setup/teardown management superior to hooks.

6. Network interception enables mocking API responses and testing without backend dependencies.

7. Trace viewer provides visual debugging with timeline of actions, network requests, and screenshots.

8. Device emulation supports testing mobile viewports and touch interactions with built-in device profiles.

9. Locator strategies prefer semantic selectors (role, label, text) over fragile CSS selectors.

10. Parallel execution with worker isolation enables fast test suites while maintaining test reliability.

## Resources

- [Official Documentation](https://playwright.dev/)
- [API Reference](https://playwright.dev/docs/api/class-playwright)
- [GitHub Repository](https://github.com/microsoft/playwright)
- [Trace Viewer Guide](https://playwright.dev/docs/trace-viewer)
- [Best Practices](https://playwright.dev/docs/best-practices)
- [VS Code Extension](https://marketplace.visualstudio.com/items?itemName=ms-playwright.playwright)
