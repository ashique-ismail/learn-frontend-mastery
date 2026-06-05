# Cypress - End-to-End Testing

## The Idea

**In plain English:** Cypress is a tool that automatically clicks through your website and checks that everything works — like a robot tester that opens your app in a real browser, fills in forms, clicks buttons, and verifies the right things appear on screen. "End-to-end" means it tests the whole journey a user takes, from start to finish.

**Real-world analogy:** Imagine a restaurant quality inspector who visits every day, orders the same meal, and checks that the order goes smoothly — from placing the order at the counter, to watching the kitchen prepare it, to confirming the right dish arrives at the table. If anything goes wrong at any step, the inspector flags it.

- The inspector = Cypress (the automated tester)
- The step-by-step order checklist = the test script (the instructions Cypress follows)
- The restaurant visit = running your web app in a real browser
- Flagging a wrong dish = a failing assertion (Cypress reporting that the page did not show what was expected)

---

## Overview

Cypress is a modern end-to-end testing framework built for the web. It runs tests directly in the browser, provides real-time reloading, automatic waiting, time-travel debugging, and excellent developer experience. Cypress is particularly popular for its intuitive API, powerful debugging capabilities, and component testing mode.

## Installation and Setup

```bash
# Install Cypress
npm install -D cypress

# Open Cypress for first time
npx cypress open

# Run Cypress headless
npx cypress run

# With TypeScript
npm install -D cypress typescript @types/node
```

### Basic Configuration

```javascript
// cypress.config.js
const { defineConfig } = require('cypress');

module.exports = defineConfig({
  e2e: {
    baseUrl: 'http://localhost:3000',
    specPattern: 'cypress/e2e/**/*.cy.{js,jsx,ts,tsx}',
    supportFile: 'cypress/support/e2e.js',
    video: true,
    screenshotOnRunFailure: true,
    viewportWidth: 1280,
    viewportHeight: 720,
    defaultCommandTimeout: 10000,
    setupNodeEvents(on, config) {
      // implement node event listeners here
    },
  },
  component: {
    devServer: {
      framework: 'react',
      bundler: 'vite',
    },
    specPattern: 'src/**/*.cy.{js,jsx,ts,tsx}',
  },
});
```

```javascript
// cypress/support/e2e.js
import './commands';

// Global beforeEach
beforeEach(() => {
  // Reset app state, clear cookies, etc.
  cy.clearCookies();
  cy.clearLocalStorage();
});

// Global configuration
Cypress.on('uncaught:exception', (err, runnable) => {
  // Returning false prevents Cypress from failing the test
  if (err.message.includes('ResizeObserver')) {
    return false;
  }
  return true;
});
```

## Core Testing Features

### Basic Test Structure

```javascript
describe('User Authentication', () => {
  beforeEach(() => {
    cy.visit('/login');
  });

  it('successfully logs in', () => {
    cy.get('[data-testid="email"]').type('user@example.com');
    cy.get('[data-testid="password"]').type('password123');
    cy.get('button[type="submit"]').click();
    
    cy.url().should('include', '/dashboard');
    cy.contains('Welcome back').should('be.visible');
  });

  it('displays error for invalid credentials', () => {
    cy.get('[data-testid="email"]').type('wrong@example.com');
    cy.get('[data-testid="password"]').type('wrongpass');
    cy.get('button[type="submit"]').click();
    
    cy.contains('Invalid credentials').should('be.visible');
    cy.url().should('include', '/login');
  });

  it('validates required fields', () => {
    cy.get('button[type="submit"]').click();
    
    cy.contains('Email is required').should('be.visible');
    cy.contains('Password is required').should('be.visible');
  });
});

// Nested describe blocks
describe('Shopping Cart', () => {
  beforeEach(() => {
    cy.login('user@test.com', 'password123'); // Custom command
    cy.visit('/products');
  });

  context('Adding items', () => {
    it('adds single item to cart', () => {
      cy.get('[data-testid="product-1"]').click();
      cy.get('[data-testid="add-to-cart"]').click();
      
      cy.get('[data-testid="cart-count"]').should('have.text', '1');
    });

    it('adds multiple items', () => {
      cy.get('[data-testid="product-1"]').click();
      cy.get('[data-testid="add-to-cart"]').click();
      cy.go('back');
      
      cy.get('[data-testid="product-2"]').click();
      cy.get('[data-testid="add-to-cart"]').click();
      
      cy.get('[data-testid="cart-count"]').should('have.text', '2');
    });
  });

  context('Removing items', () => {
    beforeEach(() => {
      cy.addToCart('product-1'); // Custom command
    });

    it('removes item from cart', () => {
      cy.visit('/cart');
      cy.get('[data-testid="remove-item"]').first().click();
      
      cy.contains('Cart is empty').should('be.visible');
    });
  });
});

// Test hooks
describe('Test lifecycle', () => {
  before(() => {
    // Runs once before all tests
    cy.task('seedDatabase');
  });

  beforeEach(() => {
    // Runs before each test
    cy.visit('/');
  });

  afterEach(() => {
    // Runs after each test
    cy.screenshot();
  });

  after(() => {
    // Runs once after all tests
    cy.task('clearDatabase');
  });

  it('example test', () => {
    cy.get('h1').should('be.visible');
  });
});
```

### Selectors and Queries

```javascript
describe('Selector strategies', () => {
  it('uses various selectors', () => {
    cy.visit('/');

    // Get by data-testid (recommended)
    cy.get('[data-testid="submit-button"]').click();
    cy.get('[data-cy="user-profile"]').should('be.visible');
    
    // Get by ID
    cy.get('#email-input').type('test@example.com');
    
    // Get by class
    cy.get('.btn-primary').click();
    
    // Get by attribute
    cy.get('[type="submit"]').click();
    cy.get('[aria-label="Close"]').click();
    
    // Get by text
    cy.contains('Submit').click();
    cy.contains('button', 'Submit').click();
    cy.contains(/submit/i).click();
    
    // Chain selectors
    cy.get('.modal').find('.close-button').click();
    cy.get('.product-list').find('[data-testid="product"]').first().click();
    
    // Parent/children traversal
    cy.get('[data-testid="item"]').parent().should('have.class', 'container');
    cy.get('.parent').children('.child').should('have.length', 3);
    
    // Siblings
    cy.get('[data-testid="active"]').siblings().should('have.length', 2);
    
    // Filter
    cy.get('li').filter('.active').should('have.length', 1);
    cy.get('li').not('.disabled').should('have.length', 5);
    
    // Positional
    cy.get('li').first().should('contain', 'First');
    cy.get('li').last().should('contain', 'Last');
    cy.get('li').eq(2).should('contain', 'Third');
  });
});
```

### Assertions

```javascript
describe('Cypress assertions', () => {
  it('demonstrates various assertions', () => {
    cy.visit('/');

    // Visibility
    cy.get('[data-testid="header"]').should('be.visible');
    cy.get('[data-testid="modal"]').should('not.be.visible');
    cy.get('[data-testid="loading"]').should('not.exist');
    
    // Text content
    cy.get('h1').should('have.text', 'Welcome');
    cy.get('h1').should('contain', 'Wel');
    cy.get('h1').should('include.text', 'come');
    
    // Value
    cy.get('input[name="email"]').should('have.value', 'test@example.com');
    cy.get('input[name="email"]').should('not.be.empty');
    
    // Attributes
    cy.get('a').should('have.attr', 'href', '/about');
    cy.get('button').should('have.class', 'btn-primary');
    cy.get('img').should('have.attr', 'alt');
    
    // CSS
    cy.get('button').should('have.css', 'background-color', 'rgb(0, 123, 255)');
    cy.get('div').should('have.css', 'display', 'flex');
    
    // State
    cy.get('button').should('be.enabled');
    cy.get('button').should('be.disabled');
    cy.get('input[type="checkbox"]').should('be.checked');
    cy.get('input[type="checkbox"]').should('not.be.checked');
    cy.get('input').should('be.focused');
    
    // Count
    cy.get('li').should('have.length', 5);
    cy.get('li').should('have.length.greaterThan', 3);
    cy.get('li').should('have.length.lessThan', 10);
    
    // URL
    cy.url().should('eq', 'http://localhost:3000/dashboard');
    cy.url().should('include', '/dashboard');
    cy.url().should('match', /\/dashboard$/);
    
    // Location
    cy.location('pathname').should('eq', '/dashboard');
    cy.location('search').should('include', 'page=1');
    
    // Multiple assertions
    cy.get('button')
      .should('be.visible')
      .and('contain', 'Submit')
      .and('have.class', 'btn-primary')
      .and('be.enabled');
    
    // Custom timeout
    cy.get('[data-testid="loading"]', { timeout: 10000 })
      .should('not.exist');
  });

  it('uses expect assertions', () => {
    cy.get('h1').then(($el) => {
      expect($el).to.have.text('Welcome');
      expect($el).to.be.visible;
    });

    cy.wrap({ name: 'John', age: 30 }).should('deep.equal', {
      name: 'John',
      age: 30,
    });
  });
});
```

## Custom Commands

```javascript
// cypress/support/commands.js

// Login command
Cypress.Commands.add('login', (email, password) => {
  cy.session([email, password], () => {
    cy.visit('/login');
    cy.get('[data-testid="email"]').type(email);
    cy.get('[data-testid="password"]').type(password);
    cy.get('button[type="submit"]').click();
    cy.url().should('include', '/dashboard');
  });
});

// Add to cart command
Cypress.Commands.add('addToCart', (productId) => {
  cy.get(`[data-testid="${productId}"]`).click();
  cy.get('[data-testid="add-to-cart"]').click();
  cy.get('[data-testid="cart-count"]').should('not.have.text', '0');
});

// Child command
Cypress.Commands.add('selectDropdown', { prevSubject: 'element' }, (subject, option) => {
  cy.wrap(subject).click();
  cy.contains(option).click();
});

// Dual command (works with or without subject)
Cypress.Commands.add('dataCy', { prevSubject: 'optional' }, (subject, value) => {
  const selector = `[data-cy="${value}"]`;
  return subject ? cy.wrap(subject).find(selector) : cy.get(selector);
});

// API commands
Cypress.Commands.add('apiLogin', (email, password) => {
  cy.request('POST', '/api/auth/login', { email, password })
    .its('body.token')
    .then((token) => {
      window.localStorage.setItem('authToken', token);
    });
});

// Database commands
Cypress.Commands.add('seedDatabase', () => {
  cy.task('db:seed');
});

Cypress.Commands.add('clearDatabase', () => {
  cy.task('db:clear');
});

// Usage
describe('Custom commands', () => {
  it('uses login command', () => {
    cy.login('user@test.com', 'password123');
    cy.visit('/dashboard');
    cy.contains('Welcome back').should('be.visible');
  });

  it('uses child command', () => {
    cy.visit('/settings');
    cy.get('[data-testid="language"]').selectDropdown('English');
    cy.get('[data-testid="timezone"]').selectDropdown('UTC');
  });

  it('uses API login', () => {
    cy.apiLogin('user@test.com', 'password123');
    cy.visit('/dashboard');
    cy.contains('Welcome back').should('be.visible');
  });
});
```

## Network Stubbing

```javascript
describe('Network interception', () => {
  it('stubs API response', () => {
    cy.intercept('GET', '/api/users', {
      statusCode: 200,
      body: [
        { id: 1, name: 'John Doe', email: 'john@test.com' },
        { id: 2, name: 'Jane Doe', email: 'jane@test.com' },
      ],
    }).as('getUsers');

    cy.visit('/users');
    cy.wait('@getUsers');
    
    cy.contains('John Doe').should('be.visible');
    cy.contains('Jane Doe').should('be.visible');
  });

  it('stubs error response', () => {
    cy.intercept('GET', '/api/users', {
      statusCode: 500,
      body: { error: 'Internal server error' },
    }).as('getUsersError');

    cy.visit('/users');
    cy.wait('@getUsersError');
    
    cy.contains('Failed to load users').should('be.visible');
  });

  it('modifies response', () => {
    cy.intercept('GET', '/api/users', (req) => {
      req.continue((res) => {
        res.body.forEach((user) => {
          user.email = `modified_${user.email}`;
        });
      });
    }).as('getUsers');

    cy.visit('/users');
    cy.wait('@getUsers');
    
    cy.contains(/modified_/).should('be.visible');
  });

  it('waits for multiple requests', () => {
    cy.intercept('GET', '/api/users').as('getUsers');
    cy.intercept('GET', '/api/posts').as('getPosts');

    cy.visit('/dashboard');
    
    cy.wait(['@getUsers', '@getPosts']).then((interceptions) => {
      expect(interceptions[0].response.statusCode).to.eq(200);
      expect(interceptions[1].response.statusCode).to.eq(200);
    });
  });

  it('asserts on request', () => {
    cy.intercept('POST', '/api/users').as('createUser');

    cy.visit('/users/new');
    cy.get('[data-testid="name"]').type('John Doe');
    cy.get('[data-testid="email"]').type('john@test.com');
    cy.get('button[type="submit"]').click();

    cy.wait('@createUser').its('request.body').should('deep.equal', {
      name: 'John Doe',
      email: 'john@test.com',
    });
  });

  it('delays response', () => {
    cy.intercept('GET', '/api/users', (req) => {
      req.continue((res) => {
        res.delay(2000); // 2 second delay
      });
    }).as('getUsers');

    cy.visit('/users');
    cy.get('[data-testid="loading"]').should('be.visible');
    cy.wait('@getUsers');
    cy.get('[data-testid="loading"]').should('not.exist');
  });
});
```

## Fixtures

```javascript
// cypress/fixtures/users.json
[
  {
    "id": 1,
    "name": "John Doe",
    "email": "john@test.com",
    "role": "admin"
  },
  {
    "id": 2,
    "name": "Jane Doe",
    "email": "jane@test.com",
    "role": "user"
  }
]

// Using fixtures
describe('Fixtures', () => {
  it('uses fixture data', () => {
    cy.fixture('users').then((users) => {
      cy.intercept('GET', '/api/users', users).as('getUsers');
      
      cy.visit('/users');
      cy.wait('@getUsers');
      
      users.forEach((user) => {
        cy.contains(user.name).should('be.visible');
      });
    });
  });

  it('uses fixture with alias', () => {
    cy.fixture('users').as('usersData');
    
    cy.get('@usersData').then((users) => {
      cy.intercept('GET', '/api/users', users);
      cy.visit('/users');
      cy.contains(users[0].name).should('be.visible');
    });
  });

  it('modifies fixture data', () => {
    cy.fixture('users').then((users) => {
      const modifiedUsers = users.map((user) => ({
        ...user,
        email: `modified_${user.email}`,
      }));
      
      cy.intercept('GET', '/api/users', modifiedUsers);
      cy.visit('/users');
      cy.contains(/modified_/).should('be.visible');
    });
  });
});
```

## Component Testing

```javascript
// Button.cy.jsx
import { Button } from './Button';

describe('Button Component', () => {
  it('renders with text', () => {
    cy.mount(<Button>Click me</Button>);
    cy.contains('Click me').should('be.visible');
  });

  it('handles click events', () => {
    const onClickSpy = cy.spy().as('onClickSpy');
    cy.mount(<Button onClick={onClickSpy}>Click me</Button>);
    
    cy.contains('Click me').click();
    cy.get('@onClickSpy').should('have.been.calledOnce');
  });

  it('renders disabled state', () => {
    cy.mount(<Button disabled>Click me</Button>);
    cy.contains('Click me').should('be.disabled');
  });

  it('renders different variants', () => {
    cy.mount(<Button variant="primary">Primary</Button>);
    cy.get('button').should('have.class', 'btn-primary');
    
    cy.mount(<Button variant="secondary">Secondary</Button>);
    cy.get('button').should('have.class', 'btn-secondary');
  });
});

// Form.cy.jsx
import { LoginForm } from './LoginForm';

describe('LoginForm Component', () => {
  it('submits form with valid data', () => {
    const onSubmitSpy = cy.spy().as('onSubmitSpy');
    cy.mount(<LoginForm onSubmit={onSubmitSpy} />);
    
    cy.get('[data-testid="email"]').type('test@example.com');
    cy.get('[data-testid="password"]').type('password123');
    cy.get('button[type="submit"]').click();
    
    cy.get('@onSubmitSpy').should('have.been.calledWith', {
      email: 'test@example.com',
      password: 'password123',
    });
  });

  it('shows validation errors', () => {
    cy.mount(<LoginForm onSubmit={cy.spy()} />);
    
    cy.get('button[type="submit"]').click();
    
    cy.contains('Email is required').should('be.visible');
    cy.contains('Password is required').should('be.visible');
  });
});
```

## Advanced Features

### Tasks (Node.js integration)

```javascript
// cypress.config.js
const { defineConfig } = require('cypress');

module.exports = defineConfig({
  e2e: {
    setupNodeEvents(on, config) {
      on('task', {
        'db:seed'() {
          // Seed database logic
          return null;
        },
        'db:clear'() {
          // Clear database logic
          return null;
        },
        log(message) {
          console.log(message);
          return null;
        },
        'readFile'(filename) {
          return require('fs').readFileSync(filename, 'utf8');
        },
      });
    },
  },
});

// Using tasks in tests
describe('Tasks', () => {
  beforeEach(() => {
    cy.task('db:clear');
    cy.task('db:seed');
  });

  it('uses task to log', () => {
    cy.task('log', 'Test started');
    cy.visit('/');
    cy.task('log', 'Test completed');
  });

  it('reads file', () => {
    cy.task('readFile', 'data.json').then((content) => {
      const data = JSON.parse(content);
      expect(data).to.have.property('users');
    });
  });
});
```

### Screenshots and Videos

```javascript
describe('Visual testing', () => {
  it('takes screenshots', () => {
    cy.visit('/');
    
    // Full page screenshot
    cy.screenshot('homepage');
    
    // Element screenshot
    cy.get('[data-testid="modal"]').screenshot('modal');
    
    // Screenshot with options
    cy.screenshot('custom', {
      capture: 'viewport',
      clip: { x: 0, y: 0, width: 100, height: 100 },
    });
  });

  it('captures video', () => {
    // Video is automatically recorded if configured
    cy.visit('/');
    cy.get('[data-testid="video-trigger"]').click();
    cy.get('[data-testid="video-player"]').should('be.visible');
  });
});
```

## Common Mistakes

1. **Using arbitrary waits**: Using `cy.wait(5000)` instead of waiting for specific elements or requests.

2. **Not aliasing intercepts**: Forgetting to use `.as()` for intercepts making tests less readable.

3. **Conditional testing**: Using if/else statements which make tests non-deterministic.

4. **Returning values incorrectly**: Not understanding Cypress async nature and command chaining.

5. **Not using custom commands**: Duplicating code instead of creating reusable commands.

6. **Poor selector strategies**: Using fragile CSS selectors instead of data-testid attributes.

7. **Testing implementation details**: Coupling tests to internal state instead of user behavior.

8. **Not cleaning state**: Tests affecting each other through shared cookies, localStorage.

9. **Overusing fixtures**: Using fixtures when intercepts with inline data would be simpler.

10. **Not leveraging time-travel**: Not using Cypress Test Runner's time-travel debugging.

## Best Practices

1. **Use data-testid attributes**: Add `data-testid` attributes for reliable selectors.

2. **Create custom commands**: Extract common patterns into reusable commands.

3. **Leverage intercepts**: Mock API responses for reliable, fast tests.

4. **Use aliases**: Name intercepts and elements with `.as()` for better readability.

5. **Clean state between tests**: Use `beforeEach` to reset application state.

6. **Organize with context**: Use `context` blocks to group related tests.

7. **Use fixtures wisely**: Store complex mock data in fixtures.

8. **Enable video on failure**: Configure video recording for CI debugging.

9. **Write deterministic tests**: Ensure tests produce same result every time.

10. **Leverage TypeScript**: Use TypeScript for better IDE support and type safety.

## When to Use Cypress

**Use Cypress when:**
- Want excellent developer experience
- Need time-travel debugging
- Building modern web applications
- Want built-in test runner UI
- Need component testing
- Want easy network mocking
- Building single-page applications

**Consider alternatives when:**
- Need true cross-browser support (use Playwright)
- Need mobile testing (use Playwright/Appium)
- Building non-web applications
- Need faster test execution (use Playwright)
- Want native multi-tab support

## Interview Questions

### Q1: How does Cypress differ from Selenium?

**Answer**: Cypress runs directly in the browser with the application, enabling real-time reloading and time-travel debugging. Selenium runs outside the browser via WebDriver protocol. Cypress has automatic waiting built-in, while Selenium requires explicit waits. Cypress provides better DX with instant feedback, but Selenium supports more browsers. Cypress is JavaScript-only; Selenium supports multiple languages.

### Q2: What is automatic waiting in Cypress and why is it important?

**Answer**: Cypress automatically waits for elements to exist, be visible, and be actionable before executing commands. This eliminates flaky tests from timing issues without manual waits. Commands like `cy.get()` retry until element is found or timeout is reached. This makes tests more reliable and easier to write compared to explicit wait strategies.

### Q3: How do you handle authentication in Cypress tests?

**Answer**: Use `cy.session()` to cache authentication state and reuse it across tests:

```javascript
Cypress.Commands.add('login', (email, password) => {
  cy.session([email, password], () => {
    cy.visit('/login');
    cy.get('[data-testid="email"]').type(email);
    cy.get('[data-testid="password"]').type(password);
    cy.get('button[type="submit"]').click();
  });
});
```

This logs in once and reuses the session, dramatically speeding up tests.

### Q4: What's the difference between `cy.visit()` and `cy.request()`?

**Answer**: `cy.visit()` navigates the browser to a URL and loads the page, simulating user navigation. `cy.request()` makes HTTP requests programmatically without loading the page, useful for API testing or setting up state. Use `cy.request()` for API calls and `cy.visit()` for page navigation.

### Q5: How do you test file uploads in Cypress?

**Answer**: Use `cy.selectFile()` to upload files:

```javascript
cy.get('input[type="file"]').selectFile('path/to/file.pdf');
cy.get('input[type="file"]').selectFile({
  contents: Cypress.Buffer.from('file contents'),
  fileName: 'file.txt',
  mimeType: 'text/plain',
});
```

### Q6: What are Cypress tasks and when should you use them?

**Answer**: Tasks are Node.js code that runs outside the browser, defined in `cypress.config.js`. Use them for operations that can't run in the browser: database operations, file system access, or external API calls. They enable test setup/teardown that requires Node.js capabilities.

### Q7: How do you handle flaky tests in Cypress?

**Answer**: Strategies include: 1) Use test retries with `retries` config, 2) Ensure proper waits with intercepts instead of arbitrary timeouts, 3) Use `cy.session()` for authentication, 4) Reset state with `beforeEach`, 5) Use deterministic data with fixtures, 6) Avoid conditional testing, 7) Leverage automatic retry on assertions. Cypress's architecture minimizes flakiness compared to Selenium.

## Key Takeaways

1. Cypress provides excellent developer experience with real-time reloading, time-travel debugging, and intuitive API.

2. Automatic waiting eliminates flaky tests by retrying commands until elements are actionable or timeout is reached.

3. Network stubbing with `cy.intercept()` enables reliable testing by mocking API responses and controlling network behavior.

4. Custom commands promote code reuse and create domain-specific testing language for your application.

5. Component testing mode enables testing React/Vue/Angular components in isolation without full E2E setup.

6. Fixtures provide organized storage for mock data, keeping tests clean and maintainable.

7. Tasks enable Node.js code execution for operations requiring file system, database, or external API access.

8. Session caching with `cy.session()` dramatically speeds up tests by reusing authentication state.

9. Time-travel debugging in Test Runner shows DOM snapshots for each command, making debugging intuitive.

10. Cypress runs tests directly in the browser, providing unique capabilities but limiting true cross-browser support.

## Resources

- [Official Documentation](https://docs.cypress.io/)
- [API Reference](https://docs.cypress.io/api/table-of-contents)
- [Best Practices](https://docs.cypress.io/guides/references/best-practices)
- [Real World App](https://github.com/cypress-io/cypress-realworld-app)
- [Component Testing](https://docs.cypress.io/guides/component-testing/overview)
- [Cypress Cloud](https://www.cypress.io/cloud)
