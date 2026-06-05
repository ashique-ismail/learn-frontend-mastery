# End-to-End Testing in Angular with Cypress and Playwright

## The Idea

**In plain English:** End-to-end (E2E) testing means running your entire app as a real user would — clicking buttons, filling forms, and checking what appears on screen — to make sure every piece works together from start to finish. Unlike tests that check one small function in isolation, E2E tests follow a whole journey through the app, just like a person using it would.

**Real-world analogy:** Imagine a restaurant inspector who visits your diner and orders a full meal from start to finish — they walk in, read the menu, place an order, watch it get cooked, receive the plate, taste the food, pay the bill, and leave. They are not just checking the stove in isolation; they are verifying the whole experience works end-to-end.
- The inspector = the E2E test runner (Cypress or Playwright)
- The diner customer journey (sit, order, eat, pay) = the user flow being tested
- The meal arriving correctly and tasting right = the assertions that check what appears on screen

---

## Table of Contents
- [Introduction](#introduction)
- [Cypress Fundamentals](#cypress-fundamentals)
- [Playwright Fundamentals](#playwright-fundamentals)
- [Setting Up E2E Testing](#setting-up-e2e-testing)
- [Writing Your First E2E Tests](#writing-your-first-e2e-tests)
- [Page Object Pattern](#page-object-pattern)
- [Testing User Flows](#testing-user-flows)
- [Network Stubbing and Mocking](#network-stubbing-and-mocking)
- [Component Testing Mode](#component-testing-mode)
- [Best Practices for Selectors](#best-practices-for-selectors)
- [Advanced E2E Patterns](#advanced-e2e-patterns)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

End-to-end (E2E) testing validates your entire Angular application from the user's perspective, testing the full stack including UI, business logic, APIs, and integrations. Unlike unit tests that verify isolated components, E2E tests ensure all parts work together correctly.

Modern E2E testing has evolved significantly with tools like Cypress and Playwright, which offer better developer experience, faster execution, and more reliable tests compared to traditional Protractor. These tools provide automatic waiting, time-travel debugging, and powerful APIs for simulating real user interactions.

## Cypress Fundamentals

### Installation

```bash
npm install cypress --save-dev
# or
yarn add cypress --dev

# Initialize Cypress
npx cypress open
```

### Basic Cypress Configuration

```typescript
// cypress.config.ts
import { defineConfig } from 'cypress';

export default defineConfig({
  e2e: {
    baseUrl: 'http://localhost:4200',
    viewportWidth: 1280,
    viewportHeight: 720,
    video: false,
    screenshotOnRunFailure: true,
    setupNodeEvents(on, config) {
      // implement node event listeners here
    },
  },
  component: {
    devServer: {
      framework: 'angular',
      bundler: 'webpack',
    },
    specPattern: '**/*.cy.ts',
  },
});
```

### Basic Cypress Test Structure

```typescript
// cypress/e2e/home.cy.ts
describe('Home Page', () => {
  beforeEach(() => {
    cy.visit('/');
  });

  it('should display the home page', () => {
    cy.get('[data-testid="home-title"]').should('contain', 'Welcome');
  });

  it('should navigate to about page', () => {
    cy.get('[data-testid="nav-about"]').click();
    cy.url().should('include', '/about');
  });
});
```

### Cypress Commands

```typescript
describe('Cypress Commands', () => {
  it('demonstrates common commands', () => {
    // Visit a page
    cy.visit('/login');

    // Get elements
    cy.get('[data-testid="username"]');
    cy.get('.submit-button');
    cy.get('#email-input');

    // Type into inputs
    cy.get('[data-testid="username"]').type('testuser');
    cy.get('[data-testid="password"]').type('password123');

    // Click elements
    cy.get('[data-testid="login-button"]').click();

    // Assertions
    cy.get('[data-testid="welcome-message"]')
      .should('be.visible')
      .and('contain', 'Welcome back');

    // Check URL
    cy.url().should('include', '/dashboard');

    // Wait for element
    cy.get('[data-testid="user-profile"]', { timeout: 10000 })
      .should('exist');

    // Multiple elements
    cy.get('[data-testid="todo-item"]').should('have.length', 5);

    // Find within
    cy.get('[data-testid="user-card"]')
      .find('[data-testid="user-name"]')
      .should('contain', 'John Doe');
  });
});
```

## Playwright Fundamentals

### Installation

```bash
npm install @playwright/test --save-dev
# or
yarn add @playwright/test --dev

# Install browsers
npx playwright install
```

### Basic Playwright Configuration

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
    baseURL: 'http://localhost:4200',
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
  ],

  webServer: {
    command: 'npm run start',
    url: 'http://localhost:4200',
    reuseExistingServer: !process.env.CI,
  },
});
```

### Basic Playwright Test Structure

```typescript
// e2e/home.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Home Page', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/');
  });

  test('should display the home page', async ({ page }) => {
    await expect(page.getByTestId('home-title')).toContainText('Welcome');
  });

  test('should navigate to about page', async ({ page }) => {
    await page.getByTestId('nav-about').click();
    await expect(page).toHaveURL(/.*about/);
  });
});
```

### Playwright Commands

```typescript
import { test, expect } from '@playwright/test';

test('demonstrates Playwright commands', async ({ page }) => {
  // Navigate
  await page.goto('/login');

  // Locators
  await page.getByTestId('username');
  await page.locator('.submit-button');
  await page.locator('#email-input');

  // Fill inputs
  await page.getByTestId('username').fill('testuser');
  await page.getByTestId('password').fill('password123');

  // Click
  await page.getByTestId('login-button').click();

  // Assertions
  await expect(page.getByTestId('welcome-message'))
    .toBeVisible();
  await expect(page.getByTestId('welcome-message'))
    .toContainText('Welcome back');

  // URL assertions
  await expect(page).toHaveURL(/.*dashboard/);

  // Wait for element
  await page.waitForSelector('[data-testid="user-profile"]', {
    timeout: 10000
  });

  // Multiple elements
  const items = page.getByTestId('todo-item');
  await expect(items).toHaveCount(5);

  // Find within
  const userCard = page.getByTestId('user-card');
  await expect(userCard.getByTestId('user-name'))
    .toContainText('John Doe');
});
```

## Setting Up E2E Testing

### Project Structure

```
my-angular-app/
├── cypress/
│   ├── e2e/
│   │   ├── auth/
│   │   │   ├── login.cy.ts
│   │   │   └── register.cy.ts
│   │   ├── dashboard/
│   │   │   └── dashboard.cy.ts
│   │   └── home.cy.ts
│   ├── fixtures/
│   │   ├── users.json
│   │   └── todos.json
│   ├── support/
│   │   ├── commands.ts
│   │   ├── e2e.ts
│   │   └── page-objects/
│   │       ├── login.page.ts
│   │       └── dashboard.page.ts
│   └── cypress.config.ts
│
├── e2e/ (Playwright)
│   ├── pages/
│   │   ├── login.page.ts
│   │   └── dashboard.page.ts
│   ├── fixtures/
│   │   └── test-data.ts
│   ├── home.spec.ts
│   └── auth.spec.ts
└── playwright.config.ts
```

### Adding Test IDs to Components

```typescript
// user-card.component.ts
@Component({
  selector: 'app-user-card',
  template: `
    <div class="user-card" [attr.data-testid]="'user-card-' + user.id">
      <h3 [attr.data-testid]="'user-name-' + user.id">{{ user.name }}</h3>
      <p [attr.data-testid]="'user-email-' + user.id">{{ user.email }}</p>
      <button
        [attr.data-testid]="'delete-user-' + user.id"
        (click)="onDelete()">
        Delete
      </button>
    </div>
  `
})
export class UserCardComponent {
  @Input() user!: User;
  @Output() delete = new EventEmitter<void>();

  onDelete(): void {
    this.delete.emit();
  }
}
```

## Writing Your First E2E Tests

### Testing a Login Flow (Cypress)

```typescript
// cypress/e2e/auth/login.cy.ts
describe('Login Flow', () => {
  beforeEach(() => {
    cy.visit('/login');
  });

  it('should display login form', () => {
    cy.get('[data-testid="login-form"]').should('be.visible');
    cy.get('[data-testid="username-input"]').should('be.visible');
    cy.get('[data-testid="password-input"]').should('be.visible');
    cy.get('[data-testid="login-button"]').should('be.visible');
  });

  it('should show validation errors for empty fields', () => {
    cy.get('[data-testid="login-button"]').click();

    cy.get('[data-testid="username-error"]')
      .should('be.visible')
      .and('contain', 'Username is required');

    cy.get('[data-testid="password-error"]')
      .should('be.visible')
      .and('contain', 'Password is required');
  });

  it('should login successfully with valid credentials', () => {
    cy.get('[data-testid="username-input"]').type('testuser');
    cy.get('[data-testid="password-input"]').type('password123');
    cy.get('[data-testid="login-button"]').click();

    // Should redirect to dashboard
    cy.url().should('include', '/dashboard');

    // Should display welcome message
    cy.get('[data-testid="welcome-message"]')
      .should('contain', 'Welcome, testuser');
  });

  it('should show error for invalid credentials', () => {
    cy.get('[data-testid="username-input"]').type('wronguser');
    cy.get('[data-testid="password-input"]').type('wrongpass');
    cy.get('[data-testid="login-button"]').click();

    cy.get('[data-testid="error-message"]')
      .should('be.visible')
      .and('contain', 'Invalid username or password');

    // Should stay on login page
    cy.url().should('include', '/login');
  });

  it('should toggle password visibility', () => {
    cy.get('[data-testid="password-input"]')
      .should('have.attr', 'type', 'password');

    cy.get('[data-testid="toggle-password"]').click();

    cy.get('[data-testid="password-input"]')
      .should('have.attr', 'type', 'text');
  });
});
```

### Testing a Login Flow (Playwright)

```typescript
// e2e/auth.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Login Flow', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/login');
  });

  test('should display login form', async ({ page }) => {
    await expect(page.getByTestId('login-form')).toBeVisible();
    await expect(page.getByTestId('username-input')).toBeVisible();
    await expect(page.getByTestId('password-input')).toBeVisible();
    await expect(page.getByTestId('login-button')).toBeVisible();
  });

  test('should show validation errors for empty fields', async ({ page }) => {
    await page.getByTestId('login-button').click();

    await expect(page.getByTestId('username-error'))
      .toContainText('Username is required');

    await expect(page.getByTestId('password-error'))
      .toContainText('Password is required');
  });

  test('should login successfully', async ({ page }) => {
    await page.getByTestId('username-input').fill('testuser');
    await page.getByTestId('password-input').fill('password123');
    await page.getByTestId('login-button').click();

    await expect(page).toHaveURL(/.*dashboard/);
    await expect(page.getByTestId('welcome-message'))
      .toContainText('Welcome, testuser');
  });

  test('should show error for invalid credentials', async ({ page }) => {
    await page.getByTestId('username-input').fill('wronguser');
    await page.getByTestId('password-input').fill('wrongpass');
    await page.getByTestId('login-button').click();

    await expect(page.getByTestId('error-message'))
      .toContainText('Invalid username or password');

    await expect(page).toHaveURL(/.*login/);
  });
});
```

## Page Object Pattern

### Page Object with Cypress

```typescript
// cypress/support/page-objects/login.page.ts
export class LoginPage {
  // Selectors
  private usernameInput = '[data-testid="username-input"]';
  private passwordInput = '[data-testid="password-input"]';
  private loginButton = '[data-testid="login-button"]';
  private errorMessage = '[data-testid="error-message"]';
  private welcomeMessage = '[data-testid="welcome-message"]';

  visit(): void {
    cy.visit('/login');
  }

  fillUsername(username: string): void {
    cy.get(this.usernameInput).type(username);
  }

  fillPassword(password: string): void {
    cy.get(this.passwordInput).type(password);
  }

  submit(): void {
    cy.get(this.loginButton).click();
  }

  login(username: string, password: string): void {
    this.fillUsername(username);
    this.fillPassword(password);
    this.submit();
  }

  getErrorMessage(): Cypress.Chainable {
    return cy.get(this.errorMessage);
  }

  getWelcomeMessage(): Cypress.Chainable {
    return cy.get(this.welcomeMessage);
  }

  shouldShowError(message: string): void {
    this.getErrorMessage()
      .should('be.visible')
      .and('contain', message);
  }

  shouldBeOnDashboard(): void {
    cy.url().should('include', '/dashboard');
  }
}

// Usage in test
import { LoginPage } from '../support/page-objects/login.page';

describe('Login with Page Object', () => {
  const loginPage = new LoginPage();

  beforeEach(() => {
    loginPage.visit();
  });

  it('should login successfully', () => {
    loginPage.login('testuser', 'password123');
    loginPage.shouldBeOnDashboard();
    loginPage.getWelcomeMessage().should('contain', 'Welcome');
  });

  it('should show error for invalid credentials', () => {
    loginPage.login('wronguser', 'wrongpass');
    loginPage.shouldShowError('Invalid username or password');
  });
});
```

### Page Object with Playwright

```typescript
// e2e/pages/login.page.ts
import { Page, Locator } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  readonly usernameInput: Locator;
  readonly passwordInput: Locator;
  readonly loginButton: Locator;
  readonly errorMessage: Locator;
  readonly welcomeMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.usernameInput = page.getByTestId('username-input');
    this.passwordInput = page.getByTestId('password-input');
    this.loginButton = page.getByTestId('login-button');
    this.errorMessage = page.getByTestId('error-message');
    this.welcomeMessage = page.getByTestId('welcome-message');
  }

  async goto(): Promise<void> {
    await this.page.goto('/login');
  }

  async login(username: string, password: string): Promise<void> {
    await this.usernameInput.fill(username);
    await this.passwordInput.fill(password);
    await this.loginButton.click();
  }

  async getErrorText(): Promise<string | null> {
    return await this.errorMessage.textContent();
  }
}

// Usage in test
import { test, expect } from '@playwright/test';
import { LoginPage } from './pages/login.page';

test.describe('Login with Page Object', () => {
  test('should login successfully', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login('testuser', 'password123');

    await expect(page).toHaveURL(/.*dashboard/);
    await expect(loginPage.welcomeMessage).toContainText('Welcome');
  });

  test('should show error for invalid credentials', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login('wronguser', 'wrongpass');

    await expect(loginPage.errorMessage)
      .toContainText('Invalid username or password');
  });
});
```

## Testing User Flows

### Complex User Journey (Cypress)

```typescript
describe('Todo Application User Flow', () => {
  beforeEach(() => {
    cy.visit('/');
  });

  it('should complete full todo workflow', () => {
    // Add new todos
    cy.get('[data-testid="new-todo-input"]').type('Buy groceries');
    cy.get('[data-testid="add-todo-button"]').click();

    cy.get('[data-testid="new-todo-input"]').type('Walk the dog');
    cy.get('[data-testid="add-todo-button"]').click();

    cy.get('[data-testid="new-todo-input"]').type('Read a book');
    cy.get('[data-testid="add-todo-button"]').click();

    // Verify todos added
    cy.get('[data-testid="todo-item"]').should('have.length', 3);

    // Complete first todo
    cy.get('[data-testid="todo-item"]')
      .first()
      .find('[data-testid="todo-checkbox"]')
      .check();

    cy.get('[data-testid="todo-item"]')
      .first()
      .should('have.class', 'completed');

    // Edit second todo
    cy.get('[data-testid="todo-item"]')
      .eq(1)
      .find('[data-testid="edit-button"]')
      .click();

    cy.get('[data-testid="edit-input"]')
      .clear()
      .type('Walk the dog in the park');

    cy.get('[data-testid="save-button"]').click();

    cy.get('[data-testid="todo-item"]')
      .eq(1)
      .should('contain', 'Walk the dog in the park');

    // Delete third todo
    cy.get('[data-testid="todo-item"]')
      .eq(2)
      .find('[data-testid="delete-button"]')
      .click();

    cy.get('[data-testid="confirm-delete"]').click();

    // Verify final state
    cy.get('[data-testid="todo-item"]').should('have.length', 2);
    cy.get('[data-testid="completed-count"]').should('contain', '1');
    cy.get('[data-testid="active-count"]').should('contain', '1');
  });

  it('should filter todos by status', () => {
    // Setup: Add and complete some todos
    ['Todo 1', 'Todo 2', 'Todo 3'].forEach(todo => {
      cy.get('[data-testid="new-todo-input"]').type(todo);
      cy.get('[data-testid="add-todo-button"]').click();
    });

    cy.get('[data-testid="todo-item"]')
      .first()
      .find('[data-testid="todo-checkbox"]')
      .check();

    // Test filters
    cy.get('[data-testid="filter-all"]').click();
    cy.get('[data-testid="todo-item"]').should('have.length', 3);

    cy.get('[data-testid="filter-active"]').click();
    cy.get('[data-testid="todo-item"]').should('have.length', 2);

    cy.get('[data-testid="filter-completed"]').click();
    cy.get('[data-testid="todo-item"]').should('have.length', 1);
  });
});
```

### Complex User Journey (Playwright)

```typescript
import { test, expect } from '@playwright/test';

test.describe('Todo Application User Flow', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/');
  });

  test('should complete full todo workflow', async ({ page }) => {
    // Add new todos
    const todoInput = page.getByTestId('new-todo-input');
    const addButton = page.getByTestId('add-todo-button');

    await todoInput.fill('Buy groceries');
    await addButton.click();

    await todoInput.fill('Walk the dog');
    await addButton.click();

    await todoInput.fill('Read a book');
    await addButton.click();

    // Verify todos added
    await expect(page.getByTestId('todo-item')).toHaveCount(3);

    // Complete first todo
    await page.getByTestId('todo-item')
      .first()
      .getByTestId('todo-checkbox')
      .check();

    await expect(page.getByTestId('todo-item').first())
      .toHaveClass(/completed/);

    // Edit second todo
    await page.getByTestId('todo-item')
      .nth(1)
      .getByTestId('edit-button')
      .click();

    await page.getByTestId('edit-input')
      .fill('Walk the dog in the park');

    await page.getByTestId('save-button').click();

    await expect(page.getByTestId('todo-item').nth(1))
      .toContainText('Walk the dog in the park');

    // Delete third todo
    await page.getByTestId('todo-item')
      .nth(2)
      .getByTestId('delete-button')
      .click();

    await page.getByTestId('confirm-delete').click();

    // Verify final state
    await expect(page.getByTestId('todo-item')).toHaveCount(2);
    await expect(page.getByTestId('completed-count')).toContainText('1');
    await expect(page.getByTestId('active-count')).toContainText('1');
  });
});
```

## Network Stubbing and Mocking

### Stubbing Network Requests (Cypress)

```typescript
describe('Network Stubbing', () => {
  beforeEach(() => {
    // Intercept API calls
    cy.intercept('GET', '/api/users', {
      statusCode: 200,
      body: [
        { id: 1, name: 'John Doe', email: 'john@example.com' },
        { id: 2, name: 'Jane Smith', email: 'jane@example.com' }
      ]
    }).as('getUsers');

    cy.visit('/users');
  });

  it('should display users from API', () => {
    cy.wait('@getUsers');

    cy.get('[data-testid="user-item"]').should('have.length', 2);
    cy.get('[data-testid="user-item"]').first()
      .should('contain', 'John Doe');
  });

  it('should handle API errors', () => {
    cy.intercept('GET', '/api/users', {
      statusCode: 500,
      body: { error: 'Internal Server Error' }
    }).as('getUsersError');

    cy.reload();
    cy.wait('@getUsersError');

    cy.get('[data-testid="error-message"]')
      .should('contain', 'Failed to load users');
  });

  it('should test POST request', () => {
    cy.intercept('POST', '/api/users', (req) => {
      expect(req.body).to.have.property('name', 'New User');

      req.reply({
        statusCode: 201,
        body: { id: 3, name: 'New User', email: 'new@example.com' }
      });
    }).as('createUser');

    cy.get('[data-testid="add-user-button"]').click();
    cy.get('[data-testid="name-input"]').type('New User');
    cy.get('[data-testid="email-input"]').type('new@example.com');
    cy.get('[data-testid="submit-button"]').click();

    cy.wait('@createUser');

    cy.get('[data-testid="user-item"]').should('have.length', 3);
  });

  it('should simulate slow network', () => {
    cy.intercept('GET', '/api/users', (req) => {
      req.reply({
        delay: 2000, // 2 second delay
        body: []
      });
    }).as('slowRequest');

    cy.reload();

    cy.get('[data-testid="loading-spinner"]').should('be.visible');
    cy.wait('@slowRequest');
    cy.get('[data-testid="loading-spinner"]').should('not.exist');
  });
});
```

### Mocking Network Requests (Playwright)

```typescript
import { test, expect } from '@playwright/test';

test.describe('Network Mocking', () => {
  test('should display users from mocked API', async ({ page }) => {
    await page.route('/api/users', async (route) => {
      await route.fulfill({
        status: 200,
        contentType: 'application/json',
        body: JSON.stringify([
          { id: 1, name: 'John Doe', email: 'john@example.com' },
          { id: 2, name: 'Jane Smith', email: 'jane@example.com' }
        ])
      });
    });

    await page.goto('/users');

    await expect(page.getByTestId('user-item')).toHaveCount(2);
    await expect(page.getByTestId('user-item').first())
      .toContainText('John Doe');
  });

  test('should handle API errors', async ({ page }) => {
    await page.route('/api/users', async (route) => {
      await route.fulfill({
        status: 500,
        body: JSON.stringify({ error: 'Internal Server Error' })
      });
    });

    await page.goto('/users');

    await expect(page.getByTestId('error-message'))
      .toContainText('Failed to load users');
  });

  test('should test POST request', async ({ page }) => {
    await page.route('/api/users', async (route) => {
      if (route.request().method() === 'POST') {
        const postData = route.request().postDataJSON();
        expect(postData.name).toBe('New User');

        await route.fulfill({
          status: 201,
          body: JSON.stringify({
            id: 3,
            name: 'New User',
            email: 'new@example.com'
          })
        });
      }
    });

    await page.goto('/users');
    await page.getByTestId('add-user-button').click();
    await page.getByTestId('name-input').fill('New User');
    await page.getByTestId('email-input').fill('new@example.com');
    await page.getByTestId('submit-button').click();

    await expect(page.getByTestId('user-item')).toHaveCount(3);
  });

  test('should abort requests', async ({ page }) => {
    // Block analytics requests
    await page.route(/analytics/, route => route.abort());

    await page.goto('/');
    // Page loads without analytics calls
  });
});
```

## Component Testing Mode

### Cypress Component Testing

```typescript
// cypress/component/counter.cy.ts
import { CounterComponent } from '../../src/app/counter/counter.component';

describe('CounterComponent', () => {
  it('should mount', () => {
    cy.mount(CounterComponent);
    cy.get('[data-testid="count"]').should('contain', '0');
  });

  it('should increment count', () => {
    cy.mount(CounterComponent);

    cy.get('[data-testid="increment"]').click();
    cy.get('[data-testid="count"]').should('contain', '1');

    cy.get('[data-testid="increment"]').click();
    cy.get('[data-testid="count"]').should('contain', '2');
  });

  it('should mount with props', () => {
    cy.mount(CounterComponent, {
      componentProperties: {
        initialCount: 10
      }
    });

    cy.get('[data-testid="count"]').should('contain', '10');
  });

  it('should emit events', () => {
    const onChangeSpy = cy.spy().as('onChangeSpy');

    cy.mount(CounterComponent, {
      componentProperties: {
        change: { emit: onChangeSpy } as any
      }
    });

    cy.get('[data-testid="increment"]').click();
    cy.get('@onChangeSpy').should('have.been.calledWith', 1);
  });
});
```

### Playwright Component Testing

```typescript
// e2e/components/counter.spec.ts
import { test, expect } from '@playwright/experimental-ct-angular';
import { CounterComponent } from '../../src/app/counter/counter.component';

test('should increment count', async ({ mount }) => {
  const component = await mount(CounterComponent);

  await expect(component.getByTestId('count')).toContainText('0');

  await component.getByTestId('increment').click();
  await expect(component.getByTestId('count')).toContainText('1');
});

test('should mount with props', async ({ mount }) => {
  const component = await mount(CounterComponent, {
    props: { initialCount: 10 }
  });

  await expect(component.getByTestId('count')).toContainText('10');
});
```

## Best Practices for Selectors

### Using data-testid Attributes

```typescript
// GOOD: Use data-testid
<button data-testid="submit-button">Submit</button>
cy.get('[data-testid="submit-button"]');
page.getByTestId('submit-button');

// BAD: Using CSS classes (fragile)
<button class="btn btn-primary">Submit</button>
cy.get('.btn.btn-primary'); // May break with style changes

// BAD: Using text content (fragile for i18n)
cy.contains('Submit'); // Breaks with translations
```

### Selector Best Practices

```typescript
// Component with good test selectors
@Component({
  template: `
    <form [attr.data-testid]="'user-form-' + userId">
      <input
        [attr.data-testid]="'username-input-' + userId"
        [(ngModel)]="username">

      <button
        [attr.data-testid]="'submit-button-' + userId"
        (click)="onSubmit()">
        Submit
      </button>
    </form>
  `
})
export class UserFormComponent {
  @Input() userId!: string;
  username = '';

  onSubmit(): void {
    // Handle submit
  }
}

// Test
it('should submit form', () => {
  cy.get('[data-testid="user-form-123"]')
    .find('[data-testid="username-input-123"]')
    .type('testuser');

  cy.get('[data-testid="submit-button-123"]').click();
});
```

## Advanced E2E Patterns

### Custom Cypress Commands

```typescript
// cypress/support/commands.ts
declare global {
  namespace Cypress {
    interface Chainable {
      login(username: string, password: string): Chainable<void>;
      logout(): Chainable<void>;
      getBySel(selector: string): Chainable<JQuery<HTMLElement>>;
    }
  }
}

Cypress.Commands.add('login', (username: string, password: string) => {
  cy.session([username, password], () => {
    cy.visit('/login');
    cy.get('[data-testid="username-input"]').type(username);
    cy.get('[data-testid="password-input"]').type(password);
    cy.get('[data-testid="login-button"]').click();
    cy.url().should('include', '/dashboard');
  });
});

Cypress.Commands.add('logout', () => {
  cy.get('[data-testid="user-menu"]').click();
  cy.get('[data-testid="logout-button"]').click();
});

Cypress.Commands.add('getBySel', (selector: string) => {
  return cy.get(`[data-testid="${selector}"]`);
});

// Usage
describe('Dashboard', () => {
  beforeEach(() => {
    cy.login('testuser', 'password123');
    cy.visit('/dashboard');
  });

  it('should display user data', () => {
    cy.getBySel('user-name').should('contain', 'testuser');
  });

  it('should logout', () => {
    cy.logout();
    cy.url().should('include', '/login');
  });
});
```

### Playwright Fixtures

```typescript
// e2e/fixtures/auth.fixture.ts
import { test as base } from '@playwright/test';
import { LoginPage } from '../pages/login.page';

type AuthFixtures = {
  authenticatedPage: Page;
  loginPage: LoginPage;
};

export const test = base.extend<AuthFixtures>({
  loginPage: async ({ page }, use) => {
    const loginPage = new LoginPage(page);
    await use(loginPage);
  },

  authenticatedPage: async ({ page }, use) => {
    // Login before each test
    await page.goto('/login');
    await page.getByTestId('username-input').fill('testuser');
    await page.getByTestId('password-input').fill('password123');
    await page.getByTestId('login-button').click();
    await page.waitForURL('**/dashboard');

    await use(page);

    // Logout after each test
    await page.getByTestId('user-menu').click();
    await page.getByTestId('logout-button').click();
  },
});

// Usage
import { test, expect } from './fixtures/auth.fixture';

test('should access protected page', async ({ authenticatedPage }) => {
  await authenticatedPage.goto('/profile');
  await expect(authenticatedPage.getByTestId('profile-data')).toBeVisible();
});
```

### Testing File Uploads

```typescript
// Cypress
it('should upload file', () => {
  cy.get('[data-testid="file-input"]').selectFile('cypress/fixtures/test-image.png');

  cy.get('[data-testid="upload-button"]').click();

  cy.get('[data-testid="success-message"]')
    .should('contain', 'File uploaded successfully');
});

// Playwright
test('should upload file', async ({ page }) => {
  await page.goto('/upload');

  await page.getByTestId('file-input').setInputFiles('e2e/fixtures/test-image.png');

  await page.getByTestId('upload-button').click();

  await expect(page.getByTestId('success-message'))
    .toContainText('File uploaded successfully');
});
```

### Testing Drag and Drop

```typescript
// Cypress
it('should drag and drop', () => {
  cy.get('[data-testid="draggable"]')
    .drag('[data-testid="dropzone"]');

  cy.get('[data-testid="dropzone"]')
    .should('contain', 'Item dropped');
});

// Playwright
test('should drag and drop', async ({ page }) => {
  await page.goto('/drag-drop');

  const draggable = page.getByTestId('draggable');
  const dropzone = page.getByTestId('dropzone');

  await draggable.dragTo(dropzone);

  await expect(dropzone).toContainText('Item dropped');
});
```

## Common Mistakes

### 1. Not Using data-testid Selectors

```typescript
// BAD: Fragile selectors
cy.get('.btn.btn-primary.submit-btn');
cy.contains('Submit');
cy.get('button').eq(2);

// GOOD: Stable test selectors
cy.get('[data-testid="submit-button"]');
```

### 2. Not Waiting for Async Operations

```typescript
// BAD: Not waiting for API
cy.visit('/users');
cy.get('[data-testid="user-item"]').should('have.length', 5); // Might fail

// GOOD: Wait for API
cy.intercept('GET', '/api/users').as('getUsers');
cy.visit('/users');
cy.wait('@getUsers');
cy.get('[data-testid="user-item"]').should('have.length', 5);
```

### 3. Testing Implementation Details

```typescript
// BAD: Testing internal state
cy.window().then(win => {
  expect(win.componentInstance.internalCounter).to.equal(5);
});

// GOOD: Testing user-visible behavior
cy.get('[data-testid="counter-display"]').should('contain', '5');
```

### 4. Overly Complex Tests

```typescript
// BAD: Testing too much in one test
it('should do everything', () => {
  // Login
  // Create item
  // Edit item
  // Delete item
  // Logout
  // ... 100 more lines
});

// GOOD: Focused tests
it('should login successfully', () => { /* ... */ });
it('should create item', () => { /* ... */ });
it('should edit item', () => { /* ... */ });
```

### 5. Not Cleaning Up State

```typescript
// BAD: Tests affect each other
it('test 1', () => {
  cy.visit('/');
  // Creates data that affects test 2
});

it('test 2', () => {
  cy.visit('/');
  // Assumes clean state but has data from test 1
});

// GOOD: Clean state between tests
beforeEach(() => {
  cy.clearLocalStorage();
  cy.clearCookies();
  // Reset database state
});
```

## Best Practices

### 1. Use Page Objects for Reusability

```typescript
// Encapsulate page interactions
export class DashboardPage {
  async navigateToSettings() { /* ... */ }
  async getUserName() { /* ... */ }
}

// Reuse across tests
const dashboard = new DashboardPage(page);
await dashboard.navigateToSettings();
```

### 2. Test User Flows, Not Implementation

```typescript
// GOOD: Test from user perspective
test('user can purchase item', async ({ page }) => {
  await page.goto('/products');
  await page.getByTestId('product-1').click();
  await page.getByTestId('add-to-cart').click();
  await page.getByTestId('checkout').click();
  await page.getByTestId('confirm-purchase').click();
  await expect(page.getByTestId('success-message')).toBeVisible();
});
```

### 3. Use Fixtures for Test Data

```typescript
// cypress/fixtures/users.json
{
  "validUser": {
    "username": "testuser",
    "password": "password123"
  },
  "adminUser": {
    "username": "admin",
    "password": "admin123"
  }
}

// Test
cy.fixture('users').then((users) => {
  cy.login(users.validUser.username, users.validUser.password);
});
```

### 4. Parallelize Tests

```typescript
// playwright.config.ts
export default defineConfig({
  workers: process.env.CI ? 2 : 4,
  fullyParallel: true,
});
```

### 5. Use Visual Testing Cautiously

```typescript
// Cypress with percy
import '@percy/cypress';

it('should match snapshot', () => {
  cy.visit('/dashboard');
  cy.percySnapshot('Dashboard');
});

// Playwright screenshots
test('visual test', async ({ page }) => {
  await page.goto('/dashboard');
  await expect(page).toHaveScreenshot('dashboard.png');
});
```

## Interview Questions

### 1. What is E2E testing and how does it differ from unit testing?

**Answer:** E2E testing validates the entire application from the user's perspective, testing all layers (UI, business logic, APIs, database) working together. Unit testing tests individual components in isolation.

**Key differences:**
- **Scope**: E2E tests full user flows; unit tests single functions/components
- **Speed**: E2E slower (seconds); unit tests fast (milliseconds)
- **Reliability**: E2E more flaky; unit tests very stable
- **Coverage**: E2E tests integration; unit tests logic
- **Cost**: E2E expensive to maintain; unit tests cheap

### 2. Why use Cypress or Playwright over Protractor?

**Answer:**
- **Modern architecture**: Run in browser (Cypress) or browser automation (Playwright) vs WebDriver
- **Better DX**: Time-travel debugging, automatic waiting, better error messages
- **Faster execution**: Direct browser control vs WebDriver protocol
- **Active development**: Protractor deprecated; Cypress/Playwright actively maintained
- **Better API**: More intuitive commands and assertions
- **Cross-browser**: Playwright supports Chromium, Firefox, WebKit natively

### 3. What are data-testid attributes and why use them?

**Answer:** `data-testid` attributes are custom data attributes specifically for testing selectors:

```html
<button data-testid="submit-button">Submit</button>
```

**Benefits:**
- **Stability**: Don't break with CSS/styling changes
- **Clarity**: Clear intent for testing
- **Separation**: Tests don't couple to implementation
- **i18n-friendly**: Don't rely on text content
- **Specificity**: Target exact elements reliably

### 4. Explain the Page Object Pattern in E2E testing.

**Answer:** Page Object Pattern encapsulates page interactions in reusable classes:

```typescript
class LoginPage {
  async login(user, pass) {
    await this.usernameInput.fill(user);
    await this.passwordInput.fill(pass);
    await this.submitButton.click();
  }
}
```

**Benefits:**
- **Reusability**: Share interactions across tests
- **Maintainability**: Update selectors in one place
- **Readability**: Tests read like user stories
- **Abstraction**: Hide implementation details

### 5. How do you handle flaky E2E tests?

**Answer:** Strategies to reduce flakiness:

- **Automatic waiting**: Use framework wait mechanisms
- **Explicit waits**: Wait for specific conditions
- **Network stubbing**: Mock API responses for consistency
- **Deterministic data**: Use fixtures instead of random data
- **Retry logic**: Configure retries for transient failures
- **Isolation**: Clean state between tests
- **Stable selectors**: Use data-testid, avoid fragile selectors
- **Avoid timing dependencies**: Don't use arbitrary waits

### 6. What is network stubbing and when should you use it?

**Answer:** Network stubbing intercepts HTTP requests and provides mock responses:

```typescript
cy.intercept('GET', '/api/users', mockUsers).as('getUsers');
```

**Use when:**
- Testing UI logic independent of backend
- Backend not ready
- Testing error scenarios
- Ensuring consistent test data
- Avoiding external dependencies
- Speeding up tests

**Don't use when:**
- Testing API integration
- Validating request/response formats
- Testing real backend behavior

### 7. How do you test authentication flows?

**Answer:** Strategies:

**Session management:**
```typescript
cy.session('user1', () => {
  cy.login('user', 'pass');
});
```

**Token injection:**
```typescript
beforeEach(() => {
  localStorage.setItem('token', 'test-token');
});
```

**Custom commands:**
```typescript
Cypress.Commands.add('login', (user, pass) => {
  // Login logic
});
```

Test login/logout, protected routes, session expiry, and role-based access.

### 8. What's the difference between Cypress and Playwright?

**Answer:**

**Cypress:**
- JavaScript/TypeScript only
- Runs in browser alongside app
- Excellent DX with time-travel debugging
- Limited cross-browser (Chrome, Firefox, Edge)
- Great for component testing

**Playwright:**
- Multiple language bindings
- Browser automation via DevTools Protocol
- True cross-browser (Chromium, Firefox, WebKit)
- Better mobile testing
- Parallel execution
- More powerful for complex scenarios

### 9. How do you organize E2E tests?

**Answer:**

```
e2e/
├── fixtures/        # Test data
├── pages/           # Page objects
├── support/         # Helpers, commands
├── specs/
│   ├── auth/       # Feature-based organization
│   ├── dashboard/
│   └── checkout/
└── config/         # Configuration
```

**Best practices:**
- Group by feature/module
- Use descriptive file names
- Shared page objects
- Centralized test data
- Separate critical user paths
- Tag tests (smoke, regression, etc.)

### 10. What are the best practices for E2E test selectors?

**Answer:**

**Preferred order:**
1. `data-testid` attributes (best)
2. `aria-label` / role attributes
3. Semantic locators (`getByRole`, `getByLabel`)
4. Element IDs (if stable)

**Avoid:**
- CSS classes (change frequently)
- Element positions (`eq(2)`, `first()`)
- Text content (i18n issues)
- Complex CSS selectors

**Example:**
```typescript
// GOOD
await page.getByTestId('submit-button');
await page.getByRole('button', { name: 'Submit' });

// BAD
await page.locator('.btn.btn-primary');
await page.locator('button').nth(3);
```

## Key Takeaways

1. **E2E tests validate complete user flows** from UI through backend, ensuring all parts work together
2. **Use data-testid attributes** for stable, implementation-independent selectors
3. **Page Object Pattern** encapsulates page interactions for reusability and maintainability
4. **Network stubbing/mocking** provides consistent, fast tests independent of backend
5. **Cypress excels in DX** with time-travel debugging; Playwright offers better cross-browser support
6. **Test user behavior**, not implementation details or internal component state
7. **Clean state between tests** using beforeEach hooks, clearing storage, and resetting data
8. **Handle async operations** properly with automatic waiting and explicit wait commands
9. **Keep tests focused and independent** - each test should validate one user flow
10. **Balance E2E and unit tests** - use E2E for critical paths, unit tests for detailed logic

## Resources

### Official Documentation
- [Cypress Documentation](https://docs.cypress.io/)
- [Playwright Documentation](https://playwright.dev/)
- [Angular Testing Guide](https://angular.dev/guide/testing)

### Tools
- [Cypress](https://www.cypress.io/) - E2E testing framework
- [Playwright](https://playwright.dev/) - Browser automation tool
- [Testing Library](https://testing-library.com/) - User-centric testing utilities

### Articles and Guides
- [Cypress Best Practices](https://docs.cypress.io/guides/references/best-practices)
- [Playwright Best Practices](https://playwright.dev/docs/best-practices)
- [Page Object Model Pattern](https://martinfowler.com/bliki/PageObject.html)

### Video Tutorials
- [Cypress Tutorial for Beginners](https://www.youtube.com/watch?v=u8vMu7viCm8)
- [Playwright Tutorial](https://www.youtube.com/watch?v=wawbt1cATsk)

### Books
- "Testing Angular Applications" by Corinna Cohn et al.
- "End-to-End Web Testing with Cypress" by Waweru Mwaura

### Community
- [Cypress Discord](https://discord.gg/cypress)
- [Playwright Discord](https://discord.gg/playwright)
- [Stack Overflow - Cypress](https://stackoverflow.com/questions/tagged/cypress)
- [Stack Overflow - Playwright](https://stackoverflow.com/questions/tagged/playwright)
