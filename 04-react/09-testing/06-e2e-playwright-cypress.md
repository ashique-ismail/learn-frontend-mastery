# E2E Testing with Playwright and Cypress

## Overview

End-to-end (E2E) testing validates complete user workflows in a real browser environment. Playwright and Cypress are modern E2E testing frameworks that provide reliable, fast, and developer-friendly testing experiences. This guide covers both tools, their differences, and best practices for E2E testing.

## Table of Contents

1. [Playwright Basics](#playwright-basics)
2. [Cypress Basics](#cypress-basics)
3. [Playwright vs Cypress](#playwright-vs-cypress)
4. [Page Object Model](#page-object-model)
5. [Network Mocking](#network-mocking)
6. [Test Isolation](#test-isolation)
7. [Visual Testing](#visual-testing)
8. [CI Integration](#ci-integration)
9. [Common Mistakes](#common-mistakes)
10. [Best Practices](#best-practices)
11. [Interview Questions](#interview-questions)

## Playwright Basics

Playwright supports multiple browsers with a unified API, auto-waiting, and parallel execution.

### Getting Started with Playwright

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  use: {
    baseURL: 'http://localhost:3000',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    { name: 'chromium', use: { browserName: 'chromium' } },
    { name: 'firefox', use: { browserName: 'firefox' } },
    { name: 'webkit', use: { browserName: 'webkit' } },
  ],
});
```

### Basic Playwright Test

```typescript
import { test, expect } from '@playwright/test';

test('user can log in', async ({ page }) => {
  // Navigate to page
  await page.goto('/login');
  
  // Fill form
  await page.fill('[name="email"]', 'user@example.com');
  await page.fill('[name="password"]', 'password123');
  
  // Click button
  await page.click('button[type="submit"]');
  
  // Wait for navigation
  await page.waitForURL('/dashboard');
  
  // Assert
  await expect(page.locator('h1')).toHaveText('Dashboard');
});
```

### Playwright Locators

```typescript
test('locator strategies', async ({ page }) => {
  await page.goto('/products');
  
  // By role (recommended)
  await page.getByRole('button', { name: 'Add to Cart' }).click();
  
  // By text
  await page.getByText('Product Name').click();
  
  // By label
  await page.getByLabel('Search').fill('laptop');
  
  // By placeholder
  await page.getByPlaceholder('Enter email').fill('user@example.com');
  
  // By test ID
  await page.getByTestId('product-card').click();
  
  // CSS selector
  await page.locator('.product-title').click();
  
  // XPath
  await page.locator('//button[contains(text(), "Buy")]').click();
});
```

### Auto-Waiting in Playwright

```typescript
test('auto-waiting', async ({ page }) => {
  await page.goto('/');
  
  // Playwright automatically waits for:
  
  // 1. Element to be attached to DOM
  await page.click('button');
  
  // 2. Element to be visible
  await page.click('.hidden-then-visible');
  
  // 3. Element to be stable (not animating)
  await page.click('.animated-button');
  
  // 4. Element to be enabled
  await page.click('button:not(:disabled)');
  
  // 5. Element to receive events
  await page.fill('input', 'text');
});
```

### Playwright Actions

```typescript
test('user interactions', async ({ page }) => {
  await page.goto('/form');
  
  // Click
  await page.click('button');
  await page.dblclick('.item');
  
  // Type
  await page.fill('input[name="username"]', 'alice');
  await page.type('textarea', 'Long text...', { delay: 100 });
  
  // Select
  await page.selectOption('select[name="country"]', 'US');
  
  // Check/Uncheck
  await page.check('input[type="checkbox"]');
  await page.uncheck('input[type="checkbox"]');
  
  // Upload file
  await page.setInputFiles('input[type="file"]', 'path/to/file.pdf');
  
  // Hover
  await page.hover('.dropdown-trigger');
  
  // Press key
  await page.press('input', 'Enter');
  await page.keyboard.press('Control+A');
});
```

### Playwright Assertions

```typescript
test('assertions', async ({ page }) => {
  await page.goto('/');
  
  // Text content
  await expect(page.locator('h1')).toHaveText('Welcome');
  await expect(page.locator('h1')).toContainText('Wel');
  
  // Visibility
  await expect(page.locator('.alert')).toBeVisible();
  await expect(page.locator('.hidden')).toBeHidden();
  
  // Enabled/Disabled
  await expect(page.locator('button')).toBeEnabled();
  await expect(page.locator('button')).toBeDisabled();
  
  // Value
  await expect(page.locator('input')).toHaveValue('initial');
  
  // Attribute
  await expect(page.locator('a')).toHaveAttribute('href', '/about');
  
  // Count
  await expect(page.locator('.item')).toHaveCount(5);
  
  // URL
  await expect(page).toHaveURL('/dashboard');
  await expect(page).toHaveTitle('Dashboard - MyApp');
});
```

### Playwright Test Fixtures

```typescript
import { test as base } from '@playwright/test';

// Extend base test with custom fixtures
type MyFixtures = {
  authenticatedPage: Page;
};

const test = base.extend<MyFixtures>({
  authenticatedPage: async ({ page }, use) => {
    // Login before each test
    await page.goto('/login');
    await page.fill('[name="email"]', 'user@example.com');
    await page.fill('[name="password"]', 'password123');
    await page.click('button[type="submit"]');
    await page.waitForURL('/dashboard');
    
    // Use the authenticated page
    await use(page);
    
    // Cleanup after test
    await page.goto('/logout');
  },
});

test('view profile', async ({ authenticatedPage }) => {
  await authenticatedPage.goto('/profile');
  await expect(authenticatedPage.locator('h1')).toHaveText('My Profile');
});
```

### Parallel Testing

```typescript
// playwright.config.ts
export default defineConfig({
  workers: 4, // Run 4 tests in parallel
  fullyParallel: true,
});

// Tests run in parallel by default
test('test 1', async ({ page }) => {
  // Each test gets its own browser context
});

test('test 2', async ({ page }) => {
  // Isolated from test 1
});

// Serial mode for dependent tests
test.describe.serial('checkout flow', () => {
  test('add to cart', async ({ page }) => {});
  test('proceed to checkout', async ({ page }) => {});
  test('complete purchase', async ({ page }) => {});
});
```

## Cypress Basics

Cypress provides time-travel debugging, automatic retries, and real-time reloads.

### Getting Started with Cypress

```typescript
// cypress.config.ts
import { defineConfig } from 'cypress';

export default defineConfig({
  e2e: {
    baseUrl: 'http://localhost:3000',
    viewportWidth: 1280,
    viewportHeight: 720,
    video: false,
    screenshotOnRunFailure: true,
  },
});
```

### Basic Cypress Test

```typescript
describe('Login', () => {
  it('allows user to log in', () => {
    cy.visit('/login');
    
    cy.get('[name="email"]').type('user@example.com');
    cy.get('[name="password"]').type('password123');
    
    cy.get('button[type="submit"]').click();
    
    cy.url().should('include', '/dashboard');
    cy.get('h1').should('contain', 'Dashboard');
  });
});
```

### Cypress Selectors

```typescript
describe('Selectors', () => {
  it('demonstrates selector strategies', () => {
    cy.visit('/');
    
    // CSS selector
    cy.get('.product-card').click();
    
    // Attribute selector
    cy.get('[data-testid="add-to-cart"]').click();
    
    // Contains text
    cy.contains('Add to Cart').click();
    
    // Find within
    cy.get('.product-card').find('.price').should('exist');
    
    // Filter
    cy.get('.product-card').filter('.featured').should('have.length', 3);
    
    // First/Last
    cy.get('.product-card').first().click();
    cy.get('.product-card').last().click();
    
    // Eq (index)
    cy.get('.product-card').eq(2).click();
  });
});
```

### Cypress Commands

```typescript
describe('User Interactions', () => {
  it('performs various actions', () => {
    cy.visit('/form');
    
    // Type
    cy.get('input[name="username"]').type('alice');
    cy.get('textarea').type('Hello world');
    
    // Clear
    cy.get('input').clear();
    
    // Click
    cy.get('button').click();
    cy.get('.item').dblclick();
    cy.get('.item').rightclick();
    
    // Check/Uncheck
    cy.get('input[type="checkbox"]').check();
    cy.get('input[type="checkbox"]').uncheck();
    
    // Select
    cy.get('select').select('Option 1');
    cy.get('select').select(['Option 1', 'Option 2']);
    
    // Upload file
    cy.get('input[type="file"]').selectFile('cypress/fixtures/file.pdf');
    
    // Hover (requires plugin)
    cy.get('.dropdown').trigger('mouseenter');
  });
});
```

### Cypress Assertions

```typescript
describe('Assertions', () => {
  it('makes various assertions', () => {
    cy.visit('/');
    
    // Existence
    cy.get('.alert').should('exist');
    cy.get('.hidden').should('not.exist');
    
    // Visibility
    cy.get('.visible').should('be.visible');
    cy.get('.hidden').should('not.be.visible');
    
    // Text content
    cy.get('h1').should('have.text', 'Welcome');
    cy.get('h1').should('contain', 'Wel');
    
    // Value
    cy.get('input').should('have.value', 'initial');
    
    // Class
    cy.get('button').should('have.class', 'active');
    
    // Attribute
    cy.get('a').should('have.attr', 'href', '/about');
    
    // Disabled/Enabled
    cy.get('button').should('be.disabled');
    cy.get('button').should('be.enabled');
    
    // Length
    cy.get('.item').should('have.length', 5);
    
    // URL
    cy.url().should('include', '/dashboard');
    cy.url().should('eq', 'http://localhost:3000/dashboard');
  });
});
```

### Cypress Automatic Retries

```typescript
describe('Automatic Retries', () => {
  it('retries until assertion passes', () => {
    cy.visit('/');
    
    cy.get('button').click();
    
    // Cypress retries this assertion until it passes (default: 4s)
    cy.get('.loading').should('not.exist');
    cy.get('.data').should('be.visible');
  });
  
  it('custom timeout', () => {
    cy.visit('/slow-page');
    
    // Wait up to 10 seconds
    cy.get('.data', { timeout: 10000 }).should('exist');
  });
});
```

### Cypress Time-Travel Debugging

```typescript
describe('Time Travel', () => {
  it('allows debugging with snapshots', () => {
    cy.visit('/');
    
    // Each command creates a snapshot
    cy.get('input').type('search term');
    cy.get('button').click();
    cy.get('.results').should('exist');
    
    // In Cypress UI, you can:
    // 1. Hover over each command to see DOM at that moment
    // 2. Click to pin and inspect
    // 3. See before/after state
  });
});
```

### Cypress Custom Commands

```typescript
// cypress/support/commands.ts
declare global {
  namespace Cypress {
    interface Chainable {
      login(email: string, password: string): Chainable<void>;
      dataCy(value: string): Chainable<JQuery<HTMLElement>>;
    }
  }
}

Cypress.Commands.add('login', (email, password) => {
  cy.visit('/login');
  cy.get('[name="email"]').type(email);
  cy.get('[name="password"]').type(password);
  cy.get('button[type="submit"]').click();
  cy.url().should('include', '/dashboard');
});

Cypress.Commands.add('dataCy', (value) => {
  return cy.get(`[data-cy="${value}"]`);
});

// Usage
describe('Custom Commands', () => {
  it('uses custom login command', () => {
    cy.login('user@example.com', 'password123');
    cy.dataCy('welcome-message').should('be.visible');
  });
});
```

## Playwright vs Cypress

### Key Differences

```typescript
// Playwright: Multiple browser contexts
test('playwright contexts', async ({ browser }) => {
  // Create multiple contexts (incognito windows)
  const context1 = await browser.newContext();
  const context2 = await browser.newContext();
  
  const page1 = await context1.newPage();
  const page2 = await context2.newPage();
  
  // Completely isolated
  await page1.goto('/');
  await page2.goto('/');
});

// Cypress: Single browser per spec
describe('cypress isolation', () => {
  it('test 1', () => {
    cy.visit('/');
    // Fresh browser state
  });
  
  it('test 2', () => {
    cy.visit('/');
    // Fresh browser state (cookies cleared automatically)
  });
});
```

### Browser Support

```typescript
// Playwright: Chromium, Firefox, WebKit (Safari)
test('cross-browser test', async ({ browserName, page }) => {
  console.log(`Running on: ${browserName}`);
  await page.goto('/');
});

// Cypress: Chrome, Edge, Firefox (no Safari)
describe('browser test', () => {
  it('works in Cypress', () => {
    cy.visit('/');
  });
});
```

### Language Support

```typescript
// Playwright: TypeScript/JavaScript, Python, .NET, Java
// Full TypeScript support out of the box

// Cypress: TypeScript/JavaScript only
// Requires separate TypeScript configuration
```

### Test Execution

```typescript
// Playwright: True parallelization
// Each worker runs independent tests simultaneously

// Cypress: Sequential by default, parallel with paid plan
// Tests run one after another in open source version
```

## Page Object Model

POM pattern encapsulates page interactions for maintainability.

### Playwright Page Objects

```typescript
// pages/LoginPage.ts
import { Page } from '@playwright/test';

export class LoginPage {
  constructor(private page: Page) {}
  
  async goto() {
    await this.page.goto('/login');
  }
  
  async login(email: string, password: string) {
    await this.page.fill('[name="email"]', email);
    await this.page.fill('[name="password"]', password);
    await this.page.click('button[type="submit"]');
  }
  
  async getErrorMessage() {
    return this.page.locator('.error-message').textContent();
  }
}

// pages/DashboardPage.ts
export class DashboardPage {
  constructor(private page: Page) {}
  
  async goto() {
    await this.page.goto('/dashboard');
  }
  
  async getWelcomeMessage() {
    return this.page.locator('h1').textContent();
  }
  
  async clickProfile() {
    await this.page.click('[data-testid="profile-link"]');
  }
}

// tests/login.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';
import { DashboardPage } from '../pages/DashboardPage';

test('successful login', async ({ page }) => {
  const loginPage = new LoginPage(page);
  const dashboardPage = new DashboardPage(page);
  
  await loginPage.goto();
  await loginPage.login('user@example.com', 'password123');
  
  await expect(page).toHaveURL('/dashboard');
  await expect(dashboardPage.getWelcomeMessage()).resolves.toContain('Welcome');
});
```

### Cypress Page Objects

```typescript
// cypress/pages/LoginPage.ts
export class LoginPage {
  visit() {
    cy.visit('/login');
  }
  
  fillEmail(email: string) {
    cy.get('[name="email"]').type(email);
    return this;
  }
  
  fillPassword(password: string) {
    cy.get('[name="password"]').type(password);
    return this;
  }
  
  submit() {
    cy.get('button[type="submit"]').click();
    return this;
  }
  
  getErrorMessage() {
    return cy.get('.error-message');
  }
}

// cypress/e2e/login.cy.ts
import { LoginPage } from '../pages/LoginPage';

describe('Login', () => {
  const loginPage = new LoginPage();
  
  it('successful login', () => {
    loginPage
      .visit()
      .fillEmail('user@example.com')
      .fillPassword('password123')
      .submit();
    
    cy.url().should('include', '/dashboard');
    cy.get('h1').should('contain', 'Welcome');
  });
  
  it('shows error for invalid credentials', () => {
    loginPage
      .visit()
      .fillEmail('invalid@example.com')
      .fillPassword('wrong')
      .submit();
    
    loginPage.getErrorMessage().should('contain', 'Invalid credentials');
  });
});
```

### Component-Based Page Objects

```typescript
// components/Navigation.ts
export class Navigation {
  constructor(private page: Page) {}
  
  async clickHome() {
    await this.page.click('[data-testid="nav-home"]');
  }
  
  async clickProfile() {
    await this.page.click('[data-testid="nav-profile"]');
  }
  
  async search(query: string) {
    await this.page.fill('[data-testid="search-input"]', query);
    await this.page.press('[data-testid="search-input"]', 'Enter');
  }
}

// pages/ProductPage.ts
export class ProductPage {
  private nav: Navigation;
  
  constructor(private page: Page) {
    this.nav = new Navigation(page);
  }
  
  async goto(id: string) {
    await this.page.goto(`/products/${id}`);
  }
  
  async addToCart() {
    await this.page.click('[data-testid="add-to-cart"]');
  }
  
  navigation() {
    return this.nav;
  }
}
```

## Network Mocking

Control network requests for reliable tests.

### Playwright Route Interception

```typescript
test('mock API response', async ({ page }) => {
  // Mock GET request
  await page.route('**/api/users', async (route) => {
    await route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify([
        { id: 1, name: 'Alice' },
        { id: 2, name: 'Bob' },
      ]),
    });
  });
  
  await page.goto('/users');
  
  await expect(page.locator('.user-item')).toHaveCount(2);
});

test('mock API error', async ({ page }) => {
  await page.route('**/api/users', async (route) => {
    await route.fulfill({
      status: 500,
      contentType: 'application/json',
      body: JSON.stringify({ error: 'Server error' }),
    });
  });
  
  await page.goto('/users');
  
  await expect(page.locator('.error-message')).toHaveText('Failed to load users');
});

test('modify request', async ({ page }) => {
  await page.route('**/api/search', async (route) => {
    const request = route.request();
    const url = new URL(request.url());
    
    // Modify query parameter
    url.searchParams.set('limit', '10');
    
    await route.continue({ url: url.toString() });
  });
  
  await page.goto('/search?q=test');
});

test('abort specific requests', async ({ page }) => {
  // Block analytics
  await page.route('**/analytics/**', (route) => route.abort());
  
  // Block images
  await page.route('**/*.{png,jpg,jpeg}', (route) => route.abort());
  
  await page.goto('/');
});
```

### Cypress Intercept

```typescript
describe('Network Mocking', () => {
  it('mocks API response', () => {
    cy.intercept('GET', '/api/users', {
      statusCode: 200,
      body: [
        { id: 1, name: 'Alice' },
        { id: 2, name: 'Bob' },
      ],
    }).as('getUsers');
    
    cy.visit('/users');
    
    cy.wait('@getUsers');
    cy.get('.user-item').should('have.length', 2);
  });
  
  it('mocks API error', () => {
    cy.intercept('GET', '/api/users', {
      statusCode: 500,
      body: { error: 'Server error' },
    });
    
    cy.visit('/users');
    
    cy.get('.error-message').should('contain', 'Failed to load users');
  });
  
  it('waits for multiple requests', () => {
    cy.intercept('GET', '/api/users').as('getUsers');
    cy.intercept('GET', '/api/posts').as('getPosts');
    
    cy.visit('/dashboard');
    
    cy.wait(['@getUsers', '@getPosts']);
    
    cy.get('.dashboard').should('be.visible');
  });
  
  it('modifies request', () => {
    cy.intercept('GET', '/api/search', (req) => {
      req.url = req.url + '&limit=10';
      req.continue();
    });
    
    cy.visit('/search?q=test');
  });
  
  it('uses fixture data', () => {
    cy.intercept('GET', '/api/users', { fixture: 'users.json' });
    
    cy.visit('/users');
  });
});
```

## Test Isolation

Ensure tests are independent and don't affect each other.

### Playwright Test Isolation

```typescript
// Each test gets fresh browser context
test('test 1', async ({ page }) => {
  await page.goto('/');
  await page.evaluate(() => localStorage.setItem('key', 'value1'));
});

test('test 2', async ({ page }) => {
  await page.goto('/');
  const value = await page.evaluate(() => localStorage.getItem('key'));
  expect(value).toBeNull(); // Fresh context, no data from test 1
});

// Database seeding
test.beforeEach(async ({ request }) => {
  // Reset database
  await request.post('/api/test/reset-db');
  
  // Seed test data
  await request.post('/api/test/seed', {
    data: {
      users: [{ email: 'test@example.com' }],
    },
  });
});
```

### Cypress Test Isolation

```typescript
describe('User Management', () => {
  beforeEach(() => {
    // Reset state before each test
    cy.clearCookies();
    cy.clearLocalStorage();
    
    // Seed database
    cy.request('POST', '/api/test/reset-db');
    cy.request('POST', '/api/test/seed', {
      users: [{ email: 'test@example.com' }],
    });
  });
  
  it('test 1', () => {
    cy.visit('/');
    cy.window().then((win) => {
      win.localStorage.setItem('key', 'value1');
    });
  });
  
  it('test 2', () => {
    cy.visit('/');
    cy.window().then((win) => {
      expect(win.localStorage.getItem('key')).to.be.null;
    });
  });
});
```

### Independent Test Data

```typescript
// ❌ BAD: Shared state between tests
let userId: string;

test('create user', async ({ page }) => {
  userId = await createUser();
  // ...
});

test('update user', async ({ page }) => {
  await updateUser(userId); // Depends on previous test!
  // ...
});

// ✅ GOOD: Each test creates its own data
test('create user', async ({ page }) => {
  const userId = await createUser();
  await page.goto(`/users/${userId}`);
  // ...
});

test('update user', async ({ page }) => {
  const userId = await createUser(); // Independent
  await page.goto(`/users/${userId}`);
  // ...
});
```

## Visual Testing

Catch visual regressions with screenshot comparison.

### Playwright Screenshots

```typescript
test('homepage visual test', async ({ page }) => {
  await page.goto('/');
  
  // Take screenshot and compare
  await expect(page).toHaveScreenshot('homepage.png');
});

test('responsive screenshots', async ({ page }) => {
  await page.goto('/');
  
  // Desktop
  await page.setViewportSize({ width: 1920, height: 1080 });
  await expect(page).toHaveScreenshot('desktop.png');
  
  // Mobile
  await page.setViewportSize({ width: 375, height: 667 });
  await expect(page).toHaveScreenshot('mobile.png');
});

test('element screenshot', async ({ page }) => {
  await page.goto('/');
  
  const element = page.locator('.hero-section');
  await expect(element).toHaveScreenshot('hero.png');
});

test('hide dynamic content', async ({ page }) => {
  await page.goto('/dashboard');
  
  // Hide elements with timestamps
  await expect(page).toHaveScreenshot({
    mask: [page.locator('.timestamp')],
  });
});
```

### Cypress Screenshots

```typescript
describe('Visual Tests', () => {
  it('takes screenshot', () => {
    cy.visit('/');
    cy.screenshot('homepage');
  });
  
  it('takes element screenshot', () => {
    cy.visit('/');
    cy.get('.hero-section').screenshot('hero');
  });
  
  it('takes full page screenshot', () => {
    cy.visit('/');
    cy.screenshot('full-page', { capture: 'fullPage' });
  });
});
```

### Percy Visual Testing

```typescript
// Playwright + Percy
import percySnapshot from '@percy/playwright';

test('visual regression with Percy', async ({ page }) => {
  await page.goto('/');
  await percySnapshot(page, 'Homepage');
});

// Cypress + Percy
import '@percy/cypress';

describe('Visual Tests', () => {
  it('captures homepage', () => {
    cy.visit('/');
    cy.percySnapshot('Homepage');
  });
  
  it('captures responsive views', () => {
    cy.visit('/');
    
    cy.percySnapshot('Homepage Desktop', { widths: [1280] });
    cy.percySnapshot('Homepage Mobile', { widths: [375] });
  });
});
```

## CI Integration

Run E2E tests in continuous integration pipelines.

### GitHub Actions with Playwright

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Install Playwright browsers
        run: npx playwright install --with-deps
      
      - name: Run E2E tests
        run: npm run test:e2e
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: playwright-report
          path: playwright-report/
```

### GitHub Actions with Cypress

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Cypress run
        uses: cypress-io/github-action@v5
        with:
          build: npm run build
          start: npm start
          wait-on: 'http://localhost:3000'
      
      - name: Upload screenshots
        if: failure()
        uses: actions/upload-artifact@v3
        with:
          name: cypress-screenshots
          path: cypress/screenshots
```

### Docker for E2E Tests

```dockerfile
# Dockerfile.e2e
FROM mcr.microsoft.com/playwright:v1.40.0-jammy

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

CMD ["npm", "run", "test:e2e"]
```

```yaml
# docker-compose.e2e.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=test
  
  e2e:
    build:
      context: .
      dockerfile: Dockerfile.e2e
    depends_on:
      - app
    environment:
      - BASE_URL=http://app:3000
```

### Test Sharding

```typescript
// playwright.config.ts
export default defineConfig({
  // Split tests across 4 CI machines
  shard: process.env.CI ? {
    current: parseInt(process.env.SHARD_INDEX || '1'),
    total: parseInt(process.env.SHARD_TOTAL || '4'),
  } : undefined,
});
```

```yaml
# .github/workflows/e2e.yml
jobs:
  test:
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - name: Run tests
        run: npm run test:e2e
        env:
          SHARD_INDEX: ${{ matrix.shard }}
          SHARD_TOTAL: 4
```

## Common Mistakes

### 1. Not Waiting for Elements

```typescript
// ❌ WRONG: Not waiting
test('bad wait', async ({ page }) => {
  await page.goto('/');
  const text = await page.locator('.async-content').textContent();
  expect(text).toBe('Loaded'); // May fail if not loaded yet
});

// ✅ CORRECT: Wait for element
test('good wait', async ({ page }) => {
  await page.goto('/');
  await page.waitForSelector('.async-content');
  const text = await page.locator('.async-content').textContent();
  expect(text).toBe('Loaded');
});

// ✅ BETTER: Use assertions that auto-wait
test('better wait', async ({ page }) => {
  await page.goto('/');
  await expect(page.locator('.async-content')).toHaveText('Loaded');
});
```

### 2. Brittle Selectors

```typescript
// ❌ WRONG: Fragile selectors
cy.get('div > div > span.class-name').click();
cy.get(':nth-child(3)').click();

// ✅ CORRECT: Stable selectors
cy.get('[data-testid="submit-button"]').click();
cy.getByRole('button', { name: 'Submit' }).click();
```

### 3. Test Dependencies

```typescript
// ❌ WRONG: Tests depend on each other
test('create item', async ({ page }) => {
  // Creates item with ID stored globally
});

test('edit item', async ({ page }) => {
  // Uses ID from previous test - will fail if run alone
});

// ✅ CORRECT: Independent tests
test('create item', async ({ page }) => {
  const id = await createItem();
  await page.goto(`/items/${id}`);
});

test('edit item', async ({ page }) => {
  const id = await createItem();
  await page.goto(`/items/${id}`);
});
```

## Best Practices

### 1. Use Data Attributes for Testing

```typescript
// ✅ Add data-testid attributes
<button data-testid="submit-button">Submit</button>

// Easy to select
await page.getByTestId('submit-button').click();
cy.get('[data-testid="submit-button"]').click();
```

### 2. Test User Journeys

```typescript
// ✅ Test complete workflows
test('complete checkout flow', async ({ page }) => {
  // 1. Browse products
  await page.goto('/products');
  await page.click('[data-testid="product-1"]');
  
  // 2. Add to cart
  await page.click('[data-testid="add-to-cart"]');
  
  // 3. Checkout
  await page.click('[data-testid="cart"]');
  await page.click('[data-testid="checkout"]');
  
  // 4. Fill shipping
  await page.fill('[name="address"]', '123 Main St');
  await page.click('[data-testid="continue"]');
  
  // 5. Complete purchase
  await page.fill('[name="cardNumber"]', '4242424242424242');
  await page.click('[data-testid="place-order"]');
  
  // 6. Verify confirmation
  await expect(page.locator('.confirmation')).toBeVisible();
});
```

### 3. Keep Tests Fast

```typescript
// ✅ Run tests in parallel
// ✅ Mock external APIs
// ✅ Use database snapshots
// ✅ Skip animations

test.beforeEach(async ({ page }) => {
  // Disable animations for faster tests
  await page.addInitScript(() => {
    document.body.style.transition = 'none !important';
    document.body.style.animation = 'none !important';
  });
});
```

### 4. Handle Flaky Tests

```typescript
// ✅ Use retries for flaky tests
test('flaky test', async ({ page }) => {
  test.fixme(); // Mark as known flaky
  // or
  test.skip(); // Skip temporarily
});

// playwright.config.ts
export default defineConfig({
  retries: process.env.CI ? 2 : 0,
});
```

## Interview Questions

### Q1: What's the difference between Playwright and Cypress?

**Answer:** Playwright supports multiple browsers (Chromium, Firefox, WebKit) with true parallel execution and multiple language support. Cypress only supports Chromium-based browsers and Firefox, runs tests sequentially in open source, and only supports JavaScript/TypeScript. Playwright has better debugging with DevTools protocol, while Cypress has time-travel debugging in its UI.

### Q2: What is the Page Object Model pattern?

**Answer:** POM is a design pattern that encapsulates page interactions into reusable classes. Each page/component has a corresponding class with methods for user actions and element selectors. This improves maintainability by centralizing selector management and reducing code duplication.

### Q3: How do you handle flaky tests in E2E testing?

**Answer:** Use automatic retries, improve selectors to be more resilient, add explicit waits, mock unreliable external APIs, ensure test isolation, disable animations, and use tools like Playwright's auto-waiting. If a test is consistently flaky, investigate the root cause rather than just increasing retries.

### Q4: What's the difference between route mocking and API mocking?

**Answer:** Route mocking intercepts network requests at the browser level, allowing you to control responses, modify requests, or block specific calls. API mocking sets up a mock server that the application calls normally. Route mocking is better for E2E tests as it doesn't require code changes and tests the full stack except network calls.

### Q5: How do you ensure test isolation?

**Answer:** Each test should create its own data, use fresh browser contexts, clear cookies/localStorage between tests, reset database state, and avoid sharing state between tests. Use beforeEach hooks to set up clean state and afterEach to clean up resources.

## Key Takeaways

1. Playwright supports multiple browsers with parallel execution
2. Cypress provides time-travel debugging and automatic retries
3. Use Page Object Model for maintainable tests
4. Mock network requests for reliable tests
5. Ensure test isolation with fresh data and state
6. Visual testing catches CSS/layout regressions
7. Run E2E tests in CI with proper artifact collection
8. Use stable selectors (data-testid, roles) over brittle ones
9. Test complete user journeys, not isolated features
10. Keep tests fast with parallelization and mocking
11. Handle flaky tests with retries and improved waits
12. Choose the right tool based on browser support and team needs

## Resources

- [Playwright Documentation](https://playwright.dev/)
- [Cypress Documentation](https://docs.cypress.io/)
- [Percy Visual Testing](https://percy.io/)
- [Playwright Best Practices](https://playwright.dev/docs/best-practices)
- [Cypress Best Practices](https://docs.cypress.io/guides/references/best-practices)
- [Testing Library Guiding Principles](https://testing-library.com/docs/guiding-principles/)
