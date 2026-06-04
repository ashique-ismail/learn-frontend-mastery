# E2E Testing Strategies

## Overview

End-to-End (E2E) testing validates complete user workflows from start to finish, testing the entire application stack including the UI, backend, database, and external services. E2E tests simulate real user interactions in a browser environment, catching integration issues and ensuring critical user journeys work correctly in production-like conditions.

## What is E2E Testing?

E2E tests verify the application from the user's perspective:

```
User Action → Browser → Frontend → API → Database → Response → UI Update
     └──────────────── E2E Test Covers Everything ────────────────┘
```

**E2E tests verify:**
- Complete user workflows
- Cross-component interactions
- Frontend-backend integration
- Database persistence
- Third-party integrations
- Browser-specific behavior
- Real network conditions

## Testing Pyramid with E2E

```
        /\
       /E2E\ (Few, Slow, Expensive)
      /____\
     /      \
    /Integration\ (Some, Medium Speed)
   /__________\
  /            \
 /  Unit Tests  \ (Many, Fast, Cheap)
/________________\
```

**E2E Test Characteristics:**
- **Slow**: Takes seconds to minutes per test
- **Expensive**: Requires browser, infrastructure
- **Brittle**: Can break from UI changes
- **High Confidence**: Tests real user scenarios
- **Fewer Tests**: Only for critical paths

## E2E Testing with Playwright

### Basic Setup and First Test

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
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
  ],
  webServer: {
    command: 'npm run start',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

### Simple E2E Test

```typescript
// e2e/login.spec.ts
import { test, expect } from '@playwright/test';

test.describe('User Authentication', () => {
  test('should login successfully with valid credentials', async ({ page }) => {
    // Navigate to login page
    await page.goto('/login');

    // Fill in credentials
    await page.getByLabel('Email').fill('user@example.com');
    await page.getByLabel('Password').fill('password123');

    // Submit form
    await page.getByRole('button', { name: 'Login' }).click();

    // Verify successful login
    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByText('Welcome back')).toBeVisible();
  });

  test('should show error with invalid credentials', async ({ page }) => {
    await page.goto('/login');

    await page.getByLabel('Email').fill('wrong@example.com');
    await page.getByLabel('Password').fill('wrongpassword');
    await page.getByRole('button', { name: 'Login' }).click();

    // Should stay on login page
    await expect(page).toHaveURL('/login');
    
    // Error message displayed
    await expect(page.getByText('Invalid credentials')).toBeVisible();
  });

  test('should validate required fields', async ({ page }) => {
    await page.goto('/login');

    // Try to submit without filling fields
    await page.getByRole('button', { name: 'Login' }).click();

    // Validation errors displayed
    await expect(page.getByText('Email is required')).toBeVisible();
    await expect(page.getByText('Password is required')).toBeVisible();
  });
});
```

## Page Object Model (POM)

The Page Object Model encapsulates page interactions and locators:

```typescript
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
    this.errorMessage = page.getByRole('alert');
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.loginButton.click();
  }

  async getErrorMessage(): Promise<string | null> {
    return this.errorMessage.textContent();
  }
}

// pages/DashboardPage.ts
export class DashboardPage {
  readonly page: Page;
  readonly welcomeMessage: Locator;
  readonly logoutButton: Locator;
  readonly userMenu: Locator;

  constructor(page: Page) {
    this.page = page;
    this.welcomeMessage = page.getByText('Welcome back');
    this.logoutButton = page.getByRole('button', { name: 'Logout' });
    this.userMenu = page.getByRole('button', { name: 'User Menu' });
  }

  async goto() {
    await this.page.goto('/dashboard');
  }

  async logout() {
    await this.userMenu.click();
    await this.logoutButton.click();
  }

  async isVisible(): Promise<boolean> {
    return this.welcomeMessage.isVisible();
  }
}

// ✅ Test using Page Objects
import { test, expect } from '@playwright/test';
import { LoginPage } from './pages/LoginPage';
import { DashboardPage } from './pages/DashboardPage';

test.describe('User Authentication Flow', () => {
  let loginPage: LoginPage;
  let dashboardPage: DashboardPage;

  test.beforeEach(async ({ page }) => {
    loginPage = new LoginPage(page);
    dashboardPage = new DashboardPage(page);
  });

  test('should complete full login/logout flow', async ({ page }) => {
    // Login
    await loginPage.goto();
    await loginPage.login('user@example.com', 'password123');

    // Verify dashboard
    await expect(page).toHaveURL('/dashboard');
    expect(await dashboardPage.isVisible()).toBe(true);

    // Logout
    await dashboardPage.logout();
    await expect(page).toHaveURL('/login');
  });
});
```

## App Actions Pattern

App Actions expose application functionality through a programmatic API:

```typescript
// utils/app-actions.ts
import { Page } from '@playwright/test';

export class AppActions {
  constructor(private page: Page) {}

  /**
   * Programmatically login user (bypasses UI)
   */
  async loginAsUser(email: string, password: string) {
    await this.page.context().addCookies([
      {
        name: 'auth_token',
        value: await this.getAuthToken(email, password),
        domain: 'localhost',
        path: '/',
      },
    ]);
  }

  /**
   * Seed test data via API
   */
  async seedUser(userData: { name: string; email: string }) {
    const response = await this.page.request.post('/api/users', {
      data: userData,
    });
    return response.json();
  }

  /**
   * Reset application state
   */
  async resetDatabase() {
    await this.page.request.post('/api/test/reset');
  }

  /**
   * Get auth token for user
   */
  private async getAuthToken(email: string, password: string): Promise<string> {
    const response = await this.page.request.post('/api/auth/login', {
      data: { email, password },
    });
    const data = await response.json();
    return data.token;
  }

  /**
   * Create product via API
   */
  async createProduct(product: { name: string; price: number }) {
    const response = await this.page.request.post('/api/products', {
      data: product,
    });
    return response.json();
  }
}

// ✅ Test using App Actions
test.describe('Shopping Cart', () => {
  let actions: AppActions;

  test.beforeEach(async ({ page }) => {
    actions = new AppActions(page);
    
    // Setup: Login and create test products programmatically
    await actions.resetDatabase();
    await actions.loginAsUser('user@example.com', 'password123');
    await actions.createProduct({ name: 'Test Product', price: 29.99 });
  });

  test('should add product to cart', async ({ page }) => {
    // Now we can focus on testing the cart functionality
    // without spending time on login and product creation UI
    await page.goto('/products');
    
    await page.getByRole('button', { name: 'Add to Cart' }).first().click();
    await page.goto('/cart');
    
    await expect(page.getByText('Test Product')).toBeVisible();
    await expect(page.getByText('$29.99')).toBeVisible();
  });
});
```

## Test Data Management

### Fixtures for Test Data

```typescript
// fixtures/test-data.ts
import { test as base } from '@playwright/test';
import { AppActions } from '../utils/app-actions';

type TestUser = {
  email: string;
  password: string;
  name: string;
};

type TestFixtures = {
  appActions: AppActions;
  authenticatedUser: TestUser;
  testProducts: Array<{ id: string; name: string; price: number }>;
};

export const test = base.extend<TestFixtures>({
  appActions: async ({ page }, use) => {
    const actions = new AppActions(page);
    await use(actions);
  },

  authenticatedUser: async ({ appActions, page }, use) => {
    const user: TestUser = {
      email: 'test@example.com',
      password: 'password123',
      name: 'Test User',
    };

    // Setup: Create and login user
    await appActions.seedUser(user);
    await appActions.loginAsUser(user.email, user.password);

    await use(user);

    // Teardown: Clean up user
    await appActions.resetDatabase();
  },

  testProducts: async ({ appActions }, use) => {
    const products = [
      { name: 'Product 1', price: 10.99 },
      { name: 'Product 2', price: 20.99 },
      { name: 'Product 3', price: 30.99 },
    ];

    // Setup: Create test products
    const createdProducts = await Promise.all(
      products.map((p) => appActions.createProduct(p))
    );

    await use(createdProducts);

    // Teardown handled by authenticatedUser cleanup
  },
});

// ✅ Test using fixtures
test('should display user products', async ({ page, authenticatedUser, testProducts }) => {
  await page.goto('/products');

  // User is already authenticated and products exist
  for (const product of testProducts) {
    await expect(page.getByText(product.name)).toBeVisible();
  }
});
```

### Test Data Builders

```typescript
// builders/UserBuilder.ts
export class UserBuilder {
  private user: Partial<User> = {
    name: 'Test User',
    email: 'test@example.com',
    role: 'user',
  };

  withName(name: string): this {
    this.user.name = name;
    return this;
  }

  withEmail(email: string): this {
    this.user.email = email;
    return this;
  }

  withRole(role: 'user' | 'admin'): this {
    this.user.role = role;
    return this;
  }

  build(): User {
    return this.user as User;
  }
}

// builders/ProductBuilder.ts
export class ProductBuilder {
  private product: Partial<Product> = {
    name: 'Test Product',
    price: 10.99,
    category: 'Electronics',
    inStock: true,
  };

  withName(name: string): this {
    this.product.name = name;
    return this;
  }

  withPrice(price: number): this {
    this.product.price = price;
    return this;
  }

  outOfStock(): this {
    this.product.inStock = false;
    return this;
  }

  build(): Product {
    return this.product as Product;
  }
}

// ✅ Test using builders
test('should handle out of stock products', async ({ page, appActions }) => {
  // Create specific test data using builder
  const product = new ProductBuilder()
    .withName('Rare Item')
    .withPrice(99.99)
    .outOfStock()
    .build();

  await appActions.createProduct(product);
  await page.goto('/products');

  await expect(page.getByText('Rare Item')).toBeVisible();
  await expect(page.getByText('Out of Stock')).toBeVisible();
  await expect(page.getByRole('button', { name: 'Add to Cart' })).toBeDisabled();
});
```

## Visual Regression Testing

```typescript
// e2e/visual.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Visual Regression', () => {
  test('homepage should match screenshot', async ({ page }) => {
    await page.goto('/');
    
    // Take screenshot and compare with baseline
    await expect(page).toHaveScreenshot('homepage.png', {
      fullPage: true,
      maxDiffPixels: 100, // Allow small differences
    });
  });

  test('product card should match screenshot', async ({ page }) => {
    await page.goto('/products');
    
    const productCard = page.locator('.product-card').first();
    await expect(productCard).toHaveScreenshot('product-card.png');
  });

  test('should handle different viewports', async ({ page }) => {
    // Mobile viewport
    await page.setViewportSize({ width: 375, height: 667 });
    await page.goto('/');
    await expect(page).toHaveScreenshot('homepage-mobile.png');

    // Tablet viewport
    await page.setViewportSize({ width: 768, height: 1024 });
    await page.goto('/');
    await expect(page).toHaveScreenshot('homepage-tablet.png');

    // Desktop viewport
    await page.setViewportSize({ width: 1920, height: 1080 });
    await page.goto('/');
    await expect(page).toHaveScreenshot('homepage-desktop.png');
  });
});
```

## Testing Complex User Workflows

### E-commerce Checkout Flow

```typescript
// e2e/checkout.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from './pages/LoginPage';

test.describe('Checkout Flow', () => {
  test.beforeEach(async ({ page }) => {
    // Setup authenticated user with products
    const actions = new AppActions(page);
    await actions.resetDatabase();
    await actions.loginAsUser('user@example.com', 'password123');
    await actions.createProduct({ name: 'Test Product', price: 29.99 });
  });

  test('should complete full checkout process', async ({ page }) => {
    // Step 1: Browse products
    await page.goto('/products');
    await expect(page.getByText('Test Product')).toBeVisible();

    // Step 2: Add to cart
    await page.getByRole('button', { name: 'Add to Cart' }).first().click();
    await expect(page.getByText('Added to cart')).toBeVisible();

    // Step 3: View cart
    await page.getByRole('link', { name: 'Cart' }).click();
    await expect(page).toHaveURL('/cart');
    await expect(page.getByText('Test Product')).toBeVisible();
    await expect(page.getByText('$29.99')).toBeVisible();

    // Step 4: Proceed to checkout
    await page.getByRole('button', { name: 'Checkout' }).click();
    await expect(page).toHaveURL('/checkout');

    // Step 5: Fill shipping information
    await page.getByLabel('Full Name').fill('John Doe');
    await page.getByLabel('Address').fill('123 Main St');
    await page.getByLabel('City').fill('New York');
    await page.getByLabel('Zip Code').fill('10001');
    await page.getByRole('button', { name: 'Continue to Payment' }).click();

    // Step 6: Fill payment information
    await page.getByLabel('Card Number').fill('4111111111111111');
    await page.getByLabel('Expiry').fill('12/25');
    await page.getByLabel('CVV').fill('123');

    // Step 7: Review order
    await page.getByRole('button', { name: 'Review Order' }).click();
    await expect(page.getByText('Order Summary')).toBeVisible();
    await expect(page.getByText('Test Product')).toBeVisible();
    await expect(page.getByText('Total: $29.99')).toBeVisible();

    // Step 8: Place order
    await page.getByRole('button', { name: 'Place Order' }).click();

    // Step 9: Verify confirmation
    await expect(page).toHaveURL(/\/order\/[^/]+/);
    await expect(page.getByText('Order Confirmed')).toBeVisible();
    await expect(page.getByText(/Order #\d+/)).toBeVisible();
  });

  test('should save cart for later', async ({ page }) => {
    // Add items to cart
    await page.goto('/products');
    await page.getByRole('button', { name: 'Add to Cart' }).first().click();

    // Leave site (close browser)
    await page.context().close();

    // Come back later (new context)
    const newContext = await page.context().browser()!.newContext();
    const newPage = await newContext.newPage();

    // Login again
    const loginPage = new LoginPage(newPage);
    await loginPage.goto();
    await loginPage.login('user@example.com', 'password123');

    // Cart should still have items
    await newPage.goto('/cart');
    await expect(newPage.getByText('Test Product')).toBeVisible();

    await newContext.close();
  });
});
```

### Multi-Step Form with Validation

```typescript
// e2e/registration.spec.ts
test.describe('User Registration', () => {
  test('should complete multi-step registration', async ({ page }) => {
    await page.goto('/register');

    // Step 1: Personal Information
    await expect(page.getByText('Step 1 of 3')).toBeVisible();
    await page.getByLabel('First Name').fill('John');
    await page.getByLabel('Last Name').fill('Doe');
    await page.getByLabel('Email').fill('john@example.com');
    await page.getByRole('button', { name: 'Next' }).click();

    // Step 2: Account Details
    await expect(page.getByText('Step 2 of 3')).toBeVisible();
    await page.getByLabel('Password').fill('SecurePass123!');
    await page.getByLabel('Confirm Password').fill('SecurePass123!');
    await page.getByRole('button', { name: 'Next' }).click();

    // Step 3: Preferences
    await expect(page.getByText('Step 3 of 3')).toBeVisible();
    await page.getByLabel('Newsletter').check();
    await page.getByLabel('Terms and Conditions').check();
    await page.getByRole('button', { name: 'Complete Registration' }).click();

    // Verify success
    await expect(page).toHaveURL('/welcome');
    await expect(page.getByText('Welcome, John!')).toBeVisible();
  });

  test('should allow going back to previous steps', async ({ page }) => {
    await page.goto('/register');

    // Complete step 1
    await page.getByLabel('First Name').fill('John');
    await page.getByLabel('Last Name').fill('Doe');
    await page.getByLabel('Email').fill('john@example.com');
    await page.getByRole('button', { name: 'Next' }).click();

    // Go to step 2
    await expect(page.getByText('Step 2 of 3')).toBeVisible();

    // Go back to step 1
    await page.getByRole('button', { name: 'Back' }).click();
    await expect(page.getByText('Step 1 of 3')).toBeVisible();

    // Data should be preserved
    await expect(page.getByLabel('First Name')).toHaveValue('John');
    await expect(page.getByLabel('Last Name')).toHaveValue('Doe');
  });

  test('should validate password strength', async ({ page }) => {
    await page.goto('/register');

    // Complete step 1
    await page.getByLabel('First Name').fill('John');
    await page.getByLabel('Last Name').fill('Doe');
    await page.getByLabel('Email').fill('john@example.com');
    await page.getByRole('button', { name: 'Next' }).click();

    // Try weak password
    await page.getByLabel('Password').fill('weak');
    await expect(page.getByText('Password too weak')).toBeVisible();

    // Strong password
    await page.getByLabel('Password').fill('SecurePass123!');
    await expect(page.getByText('Strong password')).toBeVisible();
  });
});
```

## Network Mocking and API Interception

```typescript
// e2e/api-mocking.spec.ts
test.describe('API Mocking', () => {
  test('should handle slow API responses', async ({ page }) => {
    // Intercept API call and add delay
    await page.route('/api/products', async (route) => {
      await new Promise((resolve) => setTimeout(resolve, 3000));
      await route.fulfill({
        status: 200,
        body: JSON.stringify([{ id: '1', name: 'Product' }]),
      });
    });

    await page.goto('/products');

    // Loading state should be visible
    await expect(page.getByText('Loading...')).toBeVisible();

    // Eventually products load
    await expect(page.getByText('Product')).toBeVisible({ timeout: 5000 });
  });

  test('should handle API errors gracefully', async ({ page }) => {
    // Mock API error
    await page.route('/api/products', (route) => {
      route.fulfill({
        status: 500,
        body: JSON.stringify({ error: 'Internal Server Error' }),
      });
    });

    await page.goto('/products');

    // Error message displayed
    await expect(page.getByText('Failed to load products')).toBeVisible();
    await expect(page.getByRole('button', { name: 'Retry' })).toBeVisible();
  });

  test('should modify API responses', async ({ page }) => {
    // Intercept and modify response
    await page.route('/api/products', async (route) => {
      const response = await route.fetch();
      const json = await response.json();

      // Add extra field to all products
      const modified = json.map((product: any) => ({
        ...product,
        featured: true,
      }));

      await route.fulfill({
        response,
        json: modified,
      });
    });

    await page.goto('/products');
    
    // All products should show featured badge
    const badges = page.getByText('Featured');
    expect(await badges.count()).toBeGreaterThan(0);
  });
});
```

## Accessibility Testing in E2E

```typescript
// e2e/accessibility.spec.ts
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test.describe('Accessibility', () => {
  test('should not have accessibility violations', async ({ page }) => {
    await page.goto('/');

    const accessibilityScanResults = await new AxeBuilder({ page }).analyze();

    expect(accessibilityScanResults.violations).toEqual([]);
  });

  test('should be keyboard navigable', async ({ page }) => {
    await page.goto('/');

    // Tab through interactive elements
    await page.keyboard.press('Tab');
    await expect(page.getByRole('link', { name: 'Home' })).toBeFocused();

    await page.keyboard.press('Tab');
    await expect(page.getByRole('link', { name: 'Products' })).toBeFocused();

    // Activate link with Enter
    await page.keyboard.press('Enter');
    await expect(page).toHaveURL('/products');
  });

  test('should work with screen reader', async ({ page }) => {
    await page.goto('/products');

    const product = page.getByRole('article').first();

    // Check ARIA labels
    await expect(product).toHaveAttribute('aria-label');

    // Check heading hierarchy
    const heading = product.getByRole('heading', { level: 2 });
    await expect(heading).toBeVisible();
  });
});
```

## Performance Testing

```typescript
// e2e/performance.spec.ts
test.describe('Performance', () => {
  test('should load homepage within acceptable time', async ({ page }) => {
    const startTime = Date.now();
    
    await page.goto('/');
    await page.waitForLoadState('networkidle');
    
    const loadTime = Date.now() - startTime;
    expect(loadTime).toBeLessThan(3000); // 3 seconds max
  });

  test('should measure Core Web Vitals', async ({ page }) => {
    await page.goto('/');

    const metrics = await page.evaluate(() => {
      return new Promise((resolve) => {
        new PerformanceObserver((list) => {
          const entries = list.getEntries();
          const vitals = {
            LCP: 0,
            FID: 0,
            CLS: 0,
          };

          entries.forEach((entry: any) => {
            if (entry.entryType === 'largest-contentful-paint') {
              vitals.LCP = entry.renderTime || entry.loadTime;
            }
            if (entry.entryType === 'first-input') {
              vitals.FID = entry.processingStart - entry.startTime;
            }
            if (entry.entryType === 'layout-shift' && !entry.hadRecentInput) {
              vitals.CLS += entry.value;
            }
          });

          resolve(vitals);
        }).observe({ entryTypes: ['largest-contentful-paint', 'first-input', 'layout-shift'] });
      });
    });

    expect((metrics as any).LCP).toBeLessThan(2500); // Good LCP
    expect((metrics as any).FID).toBeLessThan(100);   // Good FID
    expect((metrics as any).CLS).toBeLessThan(0.1);   // Good CLS
  });
});
```

## Common Mistakes

### 1. Too Many E2E Tests

```typescript
// ❌ BAD: Testing every edge case with E2E
test('should validate email format', async ({ page }) => {
  // This should be a unit test, not E2E
  await page.goto('/register');
  await page.getByLabel('Email').fill('invalid-email');
  await expect(page.getByText('Invalid email')).toBeVisible();
});

// ✅ GOOD: E2E tests for critical user journeys only
test('should complete registration flow', async ({ page }) => {
  // Test the entire flow, not individual validations
});
```

### 2. Testing Implementation Details

```typescript
// ❌ BAD: Coupling tests to implementation
test('should call API with correct params', async ({ page }) => {
  let apiCalled = false;
  await page.route('/api/users', (route) => {
    apiCalled = true;
    route.continue();
  });
  
  await page.goto('/users');
  expect(apiCalled).toBe(true); // Testing implementation
});

// ✅ GOOD: Test user-visible behavior
test('should display users', async ({ page }) => {
  await page.goto('/users');
  await expect(page.getByText('John Doe')).toBeVisible();
});
```

### 3. No Test Data Cleanup

```typescript
// ❌ BAD: Leaving test data behind
test('should create user', async ({ page }) => {
  await page.goto('/admin/users');
  await page.getByRole('button', { name: 'Add User' }).click();
  // ... create user
  // No cleanup!
});

// ✅ GOOD: Clean up after test
test('should create user', async ({ page, appActions }) => {
  await page.goto('/admin/users');
  await page.getByRole('button', { name: 'Add User' }).click();
  // ... create user
  
  // Cleanup
  await appActions.resetDatabase();
});
```

## Best Practices

1. **Test Critical User Journeys**: Focus on happy paths and critical business flows
2. **Use Page Object Model**: Encapsulate page interactions and locators
3. **Manage Test Data**: Use fixtures, builders, and programmatic setup
4. **Run Tests in Parallel**: Speed up execution with parallel workers
5. **Use App Actions**: Bypass UI for setup when possible
6. **Handle Waiting Properly**: Use auto-waiting, avoid fixed timeouts
7. **Test Across Browsers**: Run on Chrome, Firefox, Safari, mobile
8. **Mock External Services**: Don't depend on third-party APIs
9. **Clean Up Between Tests**: Ensure test isolation
10. **Monitor Test Flakiness**: Investigate and fix flaky tests immediately

## When to Use E2E Tests

**Use E2E tests for:**
- Critical user workflows (checkout, registration)
- Cross-browser compatibility
- Complete integration scenarios
- Regression testing of key features
- Smoke tests before deployment

**Don't use E2E tests for:**
- Unit-level logic testing
- Edge case validation
- Performance-sensitive scenarios
- Every feature variation
- Frequently changing UI

## Interview Questions

1. **What's the difference between E2E and integration tests?**
   - E2E tests verify complete user workflows in a browser with the full stack. Integration tests verify multiple units work together, usually without a browser.

2. **What is the Page Object Model and why use it?**
   - POM encapsulates page interactions and locators in classes. It improves maintainability, reduces duplication, and makes tests more readable.

3. **How do you handle flaky E2E tests?**
   - Use auto-waiting, avoid fixed timeouts, properly manage test data, run tests in isolation, use retries strategically, and investigate root causes.

4. **What are App Actions and when should you use them?**
   - App Actions programmatically set up application state (login, seed data) instead of clicking through UI. Use them to speed up tests and focus on the functionality being tested.

5. **Why should E2E tests be few compared to unit tests?**
   - E2E tests are slow, expensive, brittle, and take longer to debug. Focus on critical paths; cover details with faster unit/integration tests.

## Key Takeaways

1. E2E tests verify complete user workflows from UI to database
2. Keep E2E tests few, focused on critical business flows
3. Use Page Object Model for maintainable tests
4. Manage test data with fixtures and builders
5. Use App Actions to bypass UI for test setup
6. Run tests in parallel across multiple browsers
7. Handle waiting properly with auto-waiting features
8. Mock external services to prevent dependencies
9. Clean up test data between runs
10. Monitor and fix flaky tests immediately

## Resources

- **Playwright Documentation**: https://playwright.dev/
- **Cypress Documentation**: https://docs.cypress.io/
- **Testing Library**: https://testing-library.com/
- **Axe Accessibility Testing**: https://www.deque.com/axe/
- **Page Object Model Pattern**: https://martinfowler.com/bliki/PageObject.html
- **Kent C. Dodds - Testing**: https://kentcdodds.com/blog/
- **"Test Automation Patterns"**: https://testautomationpatterns.org/
