# Testing Pyramid vs Testing Trophy

## Table of Contents
- [Introduction](#introduction)
- [The Testing Pyramid](#the-testing-pyramid)
- [The Testing Trophy](#the-testing-trophy)
- [Unit Tests](#unit-tests)
- [Integration Tests](#integration-tests)
- [End-to-End Tests](#end-to-end-tests)
- [Pyramid vs Trophy: Which to Choose](#pyramid-vs-trophy-which-to-choose)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Testing strategies have evolved significantly in frontend development. Two dominant philosophies guide how teams structure their test suites: the **Testing Pyramid** (traditional approach) and the **Testing Trophy** (modern approach). Understanding both helps you make informed decisions about test coverage and resource allocation.

```
Testing Pyramid (Traditional)        Testing Trophy (Modern)
========================             =====================

         /\                                  ___
        /  \                                /   \
       / E2E\                              / E2E \
      /______\                            /-------\
     /        \                          /         \
    /Integration\                      /Integration \
   /____________ \                    /___________  \
  /              \                  /               \
 /   Unit Tests   \                /  Integration    \
/__________________\              /_________________  \
                                 /                   \
                                /     Unit Tests      \
                               /_______________________\

70% Unit                       10% Static/Manual
20% Integration                40% Integration
10% E2E                        40% Integration
                               10% E2E
```

The key difference: **The pyramid prioritizes unit tests, while the trophy emphasizes integration tests**.

## The Testing Pyramid

### Concept

Introduced by Mike Cohn, the testing pyramid suggests:
- **Base (70%)**: Many fast unit tests
- **Middle (20%)**: Fewer integration tests
- **Top (10%)**: Very few E2E tests

### Rationale

```typescript
/*
Why the Pyramid Made Sense (Backend/Traditional):
- Unit tests are fast (milliseconds)
- Integration tests are slower (seconds)
- E2E tests are slowest (minutes)
- Cost increases as you go up
- Feedback time increases
*/
```

### Example in React

```typescript
// Unit Test (Base of Pyramid) - 70%
import { render, screen } from '@testing-library/react';
import { Button } from './Button';

describe('Button Component', () => {
  it('renders with correct text', () => {
    render(<Button>Click me</Button>);
    expect(screen.getByText('Click me')).toBeInTheDocument();
  });

  it('calls onClick when clicked', () => {
    const handleClick = jest.fn();
    render(<Button onClick={handleClick}>Click</Button>);
    screen.getByText('Click').click();
    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  it('applies disabled state', () => {
    render(<Button disabled>Click</Button>);
    expect(screen.getByRole('button')).toBeDisabled();
  });
});

// Integration Test (Middle) - 20%
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { LoginForm } from './LoginForm';

describe('LoginForm Integration', () => {
  it('submits form with valid credentials', async () => {
    const onSubmit = jest.fn();
    render(<LoginForm onSubmit={onSubmit} />);

    await userEvent.type(screen.getByLabelText('Email'), 'user@example.com');
    await userEvent.type(screen.getByLabelText('Password'), 'password123');
    await userEvent.click(screen.getByRole('button', { name: 'Login' }));

    await waitFor(() => {
      expect(onSubmit).toHaveBeenCalledWith({
        email: 'user@example.com',
        password: 'password123'
      });
    });
  });

  it('shows validation errors', async () => {
    render(<LoginForm onSubmit={jest.fn()} />);

    await userEvent.click(screen.getByRole('button', { name: 'Login' }));

    expect(await screen.findByText('Email is required')).toBeInTheDocument();
    expect(await screen.findByText('Password is required')).toBeInTheDocument();
  });
});

// E2E Test (Top) - 10%
import { test, expect } from '@playwright/test';

test.describe('User Login Flow', () => {
  test('user can log in successfully', async ({ page }) => {
    await page.goto('http://localhost:3000/login');
    
    await page.fill('input[name="email"]', 'user@example.com');
    await page.fill('input[name="password"]', 'password123');
    await page.click('button:has-text("Login")');
    
    await expect(page).toHaveURL('http://localhost:3000/dashboard');
    await expect(page.locator('h1')).toHaveText('Dashboard');
  });
});
```

### Problems with the Pyramid for Frontend

```typescript
/*
Challenges:
1. Over-reliance on mocks
2. Tests pass but app is broken
3. Not testing real user interactions
4. Missing integration issues
5. Confidence gap between tests and reality
*/

// Example: Unit test passes but app is broken
// ❌ Component test (passes)
it('renders username', () => {
  const user = { name: 'John' };
  render(<UserProfile user={user} />);
  expect(screen.getByText('John')).toBeInTheDocument();
});

// ❌ But in production, API returns different structure
// API actually returns: { username: 'John' } not { name: 'John' }
// Test passes, but app crashes!
```

## The Testing Trophy

### Concept

Popularized by Kent C. Dodds, the testing trophy suggests:
- **Base (10%)**: Static analysis (TypeScript, ESLint)
- **Lower (40%)**: Integration tests
- **Upper (40%)**: More integration tests
- **Top (10%)**: E2E tests
- **Foundation**: Static analysis (bonus layer)

### Rationale

```typescript
/*
Why the Trophy Works for Frontend:
- Integration tests provide best confidence
- Test how components work together
- Less mocking = more realistic tests
- Catches real user-facing bugs
- Better return on investment
*/
```

### Example in React

```typescript
// Static Analysis (Foundation) - Free confidence
// TypeScript catches many bugs at compile time
interface User {
  id: string;
  name: string;
  email: string;
}

const UserProfile: React.FC<{ user: User }> = ({ user }) => {
  return <div>{user.name}</div>;
};

// TypeScript will error if you pass wrong type:
// <UserProfile user={{ id: 1, username: 'John' }} /> ❌

// Integration Tests (Main Focus) - 80%
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { rest } from 'msw';
import { setupServer } from 'msw/node';
import { UserDashboard } from './UserDashboard';

const server = setupServer(
  rest.get('/api/user', (req, res, ctx) => {
    return res(ctx.json({ id: '1', name: 'John', email: 'john@example.com' }));
  }),
  rest.get('/api/posts', (req, res, ctx) => {
    return res(ctx.json([
      { id: '1', title: 'Post 1', author: 'John' },
      { id: '2', title: 'Post 2', author: 'John' }
    ]));
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe('UserDashboard Integration', () => {
  it('loads and displays user data with posts', async () => {
    render(<UserDashboard userId="1" />);

    // Loading state
    expect(screen.getByText('Loading...')).toBeInTheDocument();

    // User data loaded
    await waitFor(() => {
      expect(screen.getByText('John')).toBeInTheDocument();
    });

    // Posts loaded
    expect(await screen.findByText('Post 1')).toBeInTheDocument();
    expect(await screen.findByText('Post 2')).toBeInTheDocument();
  });

  it('handles API errors gracefully', async () => {
    server.use(
      rest.get('/api/user', (req, res, ctx) => {
        return res(ctx.status(500));
      })
    );

    render(<UserDashboard userId="1" />);

    expect(await screen.findByText('Error loading user data')).toBeInTheDocument();
  });

  it('allows user to create new post', async () => {
    server.use(
      rest.post('/api/posts', async (req, res, ctx) => {
        const body = await req.json();
        return res(ctx.json({ id: '3', ...body }));
      })
    );

    render(<UserDashboard userId="1" />);

    await waitFor(() => screen.getByText('John'));

    await userEvent.click(screen.getByRole('button', { name: 'New Post' }));
    await userEvent.type(screen.getByLabelText('Title'), 'New Post Title');
    await userEvent.type(screen.getByLabelText('Content'), 'Post content');
    await userEvent.click(screen.getByRole('button', { name: 'Submit' }));

    expect(await screen.findByText('New Post Title')).toBeInTheDocument();
  });
});
```

```typescript
// Angular: Integration test with real dependencies
import { TestBed } from '@angular/core/testing';
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';
import { RouterTestingModule } from '@angular/router/testing';
import { UserDashboardComponent } from './user-dashboard.component';
import { UserService } from './user.service';
import { PostService } from './post.service';

describe('UserDashboardComponent Integration', () => {
  let component: UserDashboardComponent;
  let httpMock: HttpTestingController;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [
        HttpClientTestingModule,
        RouterTestingModule
      ],
      declarations: [UserDashboardComponent],
      providers: [UserService, PostService]
    }).compileComponents();

    const fixture = TestBed.createComponent(UserDashboardComponent);
    component = fixture.componentInstance;
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
  });

  it('loads user and posts on init', () => {
    component.ngOnInit();

    const userReq = httpMock.expectOne('/api/user/1');
    expect(userReq.request.method).toBe('GET');
    userReq.flush({ id: '1', name: 'John', email: 'john@example.com' });

    const postsReq = httpMock.expectOne('/api/posts?userId=1');
    expect(postsReq.request.method).toBe('GET');
    postsReq.flush([
      { id: '1', title: 'Post 1' },
      { id: '2', title: 'Post 2' }
    ]);

    expect(component.user).toEqual({ id: '1', name: 'John', email: 'john@example.com' });
    expect(component.posts.length).toBe(2);
  });

  it('handles error when loading user', () => {
    component.ngOnInit();

    const req = httpMock.expectOne('/api/user/1');
    req.flush('Error', { status: 500, statusText: 'Server Error' });

    expect(component.error).toBe('Error loading user data');
  });
});
```

## Unit Tests

### When to Write Unit Tests

```typescript
// ✅ GOOD: Pure functions, utilities
export function formatCurrency(amount: number): string {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD'
  }).format(amount);
}

describe('formatCurrency', () => {
  it('formats positive numbers', () => {
    expect(formatCurrency(1000)).toBe('$1,000.00');
  });

  it('formats negative numbers', () => {
    expect(formatCurrency(-500)).toBe('-$500.00');
  });

  it('formats zero', () => {
    expect(formatCurrency(0)).toBe('$0.00');
  });
});

// ✅ GOOD: Complex logic/algorithms
export function calculateDiscount(
  price: number,
  discountPercent: number,
  memberLevel: 'bronze' | 'silver' | 'gold'
): number {
  const memberMultiplier = {
    bronze: 1,
    silver: 1.1,
    gold: 1.2
  }[memberLevel];

  const discount = price * (discountPercent / 100) * memberMultiplier;
  return Math.round(discount * 100) / 100;
}

describe('calculateDiscount', () => {
  it('calculates basic discount', () => {
    expect(calculateDiscount(100, 10, 'bronze')).toBe(10);
  });

  it('applies member multiplier', () => {
    expect(calculateDiscount(100, 10, 'gold')).toBe(12);
  });

  it('rounds to 2 decimal places', () => {
    expect(calculateDiscount(33.33, 10, 'bronze')).toBe(3.33);
  });
});

// ❌ AVOID: Over-mocking React components
// This provides little value
it('renders component', () => {
  const wrapper = shallow(<MyComponent />);
  expect(wrapper.find('div')).toHaveLength(1);
});
```

## Integration Tests

### Focus of the Trophy

```typescript
// React: Comprehensive integration test
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { rest } from 'msw';
import { setupServer } from 'msw/node';
import { ShoppingCart } from './ShoppingCart';

const server = setupServer(
  rest.get('/api/products', (req, res, ctx) => {
    return res(ctx.json([
      { id: '1', name: 'Product 1', price: 10 },
      { id: '2', name: 'Product 2', price: 20 }
    ]));
  }),
  rest.post('/api/cart', async (req, res, ctx) => {
    const body = await req.json();
    return res(ctx.json({ success: true, cart: body }));
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

function renderWithProviders(ui: React.ReactElement) {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false }
    }
  });

  return render(
    <QueryClientProvider client={queryClient}>
      {ui}
    </QueryClientProvider>
  );
}

describe('ShoppingCart Integration', () => {
  it('completes full shopping flow', async () => {
    renderWithProviders(<ShoppingCart />);

    // 1. Products load
    expect(await screen.findByText('Product 1')).toBeInTheDocument();
    expect(screen.getByText('Product 2')).toBeInTheDocument();

    // 2. Add to cart
    await userEvent.click(screen.getAllByRole('button', { name: 'Add to Cart' })[0]);
    
    // 3. Cart updates
    expect(await screen.findByText('1 item in cart')).toBeInTheDocument();
    expect(screen.getByText('Total: $10.00')).toBeInTheDocument();

    // 4. Increase quantity
    await userEvent.click(screen.getByRole('button', { name: 'Increase quantity' }));
    expect(await screen.findByText('Total: $20.00')).toBeInTheDocument();

    // 5. Checkout
    await userEvent.click(screen.getByRole('button', { name: 'Checkout' }));
    expect(await screen.findByText('Order confirmed')).toBeInTheDocument();
  });

  it('handles out of stock products', async () => {
    server.use(
      rest.get('/api/products', (req, res, ctx) => {
        return res(ctx.json([
          { id: '1', name: 'Product 1', price: 10, stock: 0 }
        ]));
      })
    );

    renderWithProviders(<ShoppingCart />);

    await waitFor(() => {
      expect(screen.getByRole('button', { name: 'Add to Cart' })).toBeDisabled();
    });
    expect(screen.getByText('Out of stock')).toBeInTheDocument();
  });

  it('shows error when checkout fails', async () => {
    server.use(
      rest.post('/api/cart', (req, res, ctx) => {
        return res(ctx.status(500), ctx.json({ error: 'Payment failed' }));
      })
    );

    renderWithProviders(<ShoppingCart />);

    await waitFor(() => screen.getByText('Product 1'));
    await userEvent.click(screen.getAllByRole('button', { name: 'Add to Cart' })[0]);
    await userEvent.click(await screen.findByRole('button', { name: 'Checkout' }));

    expect(await screen.findByText('Payment failed')).toBeInTheDocument();
  });
});
```

## End-to-End Tests

### When to Use E2E

```typescript
// Playwright: Critical user journeys only
import { test, expect } from '@playwright/test';

test.describe('Critical User Journeys', () => {
  test('complete purchase flow', async ({ page }) => {
    // 1. Browse products
    await page.goto('http://localhost:3000');
    await expect(page.locator('h1')).toHaveText('Products');

    // 2. Add to cart
    await page.click('button:has-text("Add to Cart"):first-of-type');
    await expect(page.locator('.cart-badge')).toHaveText('1');

    // 3. View cart
    await page.click('a:has-text("Cart")');
    await expect(page).toHaveURL(/.*\/cart/);

    // 4. Proceed to checkout
    await page.click('button:has-text("Checkout")');
    await expect(page).toHaveURL(/.*\/checkout/);

    // 5. Fill shipping info
    await page.fill('input[name="name"]', 'John Doe');
    await page.fill('input[name="email"]', 'john@example.com');
    await page.fill('input[name="address"]', '123 Main St');

    // 6. Complete payment
    await page.click('button:has-text("Complete Order")');
    await expect(page.locator('.success-message')).toBeVisible();
    await expect(page).toHaveURL(/.*\/order\/confirmation/);
  });

  test('user authentication flow', async ({ page }) => {
    await page.goto('http://localhost:3000/login');

    await page.fill('input[name="email"]', 'user@example.com');
    await page.fill('input[name="password"]', 'password123');
    await page.click('button:has-text("Login")');

    await expect(page).toHaveURL('http://localhost:3000/dashboard');
    await expect(page.locator('.user-name')).toHaveText('John Doe');
  });
});
```

## Pyramid vs Trophy: Which to Choose

### Decision Matrix

```typescript
/*
Choose Testing Pyramid When:
✓ Backend API testing
✓ Heavy business logic
✓ Pure functions dominate
✓ Minimal UI complexity
✓ Fast feedback is critical

Choose Testing Trophy When:
✓ Frontend applications ✓✓✓
✓ React/Angular/Vue apps
✓ Complex user interactions
✓ API integrations
✓ Real user behavior matters
*/
```

### Hybrid Approach

```typescript
// Modern frontend testing strategy
const TestingStrategy = {
  // Foundation (Free)
  staticAnalysis: {
    typescript: 'Type safety',
    eslint: 'Code quality',
    prettier: 'Code formatting'
  },

  // 20% - Unit Tests
  unit: {
    utilities: 'Pure functions',
    hooks: 'Custom React hooks',
    helpers: 'Complex algorithms',
    reducers: 'State management logic'
  },

  // 60% - Integration Tests
  integration: {
    components: 'Component + children + state',
    features: 'Feature workflows',
    api: 'API integration with MSW',
    forms: 'Form submission flows'
  },

  // 20% - E2E Tests
  e2e: {
    critical: 'Purchase, signup, login',
    smoke: 'Homepage, key pages load',
    regression: 'Previous bug scenarios'
  }
};
```

## Common Mistakes

### 1. Testing Implementation Details

```typescript
// ❌ WRONG: Testing internal state
it('updates state when button clicked', () => {
  const wrapper = mount(<Counter />);
  wrapper.find('button').simulate('click');
  expect(wrapper.state('count')).toBe(1); // Implementation detail!
});

// ✅ CORRECT: Test user-visible behavior
it('increments counter when button clicked', async () => {
  render(<Counter />);
  const button = screen.getByRole('button', { name: 'Increment' });
  
  await userEvent.click(button);
  
  expect(screen.getByText('Count: 1')).toBeInTheDocument();
});
```

### 2. Over-Mocking

```typescript
// ❌ WRONG: Mocking everything
jest.mock('./api');
jest.mock('./utils');
jest.mock('./hooks');
// Tests pass but app might be broken

// ✅ CORRECT: Mock only external dependencies
// Use MSW for API, real utilities and hooks
server.use(
  rest.get('/api/data', (req, res, ctx) => {
    return res(ctx.json({ data: 'test' }));
  })
);
```

### 3. Not Testing User Interactions

```typescript
// ❌ WRONG: Shallow tests
it('renders', () => {
  const wrapper = shallow(<LoginForm />);
  expect(wrapper.exists()).toBe(true);
});

// ✅ CORRECT: Test real user flow
it('submits form with valid data', async () => {
  render(<LoginForm />);
  
  await userEvent.type(screen.getByLabelText('Email'), 'user@example.com');
  await userEvent.type(screen.getByLabelText('Password'), 'password123');
  await userEvent.click(screen.getByRole('button', { name: 'Login' }));
  
  expect(await screen.findByText('Welcome back!')).toBeInTheDocument();
});
```

## Best Practices

### 1. Write Tests That Resemble User Behavior

```typescript
// ✅ Query by role, label, text (what users see)
screen.getByRole('button', { name: 'Submit' });
screen.getByLabelText('Email Address');
screen.getByText('Welcome!');

// ❌ Avoid test IDs and implementation details
screen.getByTestId('submit-button'); // Only as last resort
```

### 2. Use Testing Library Queries Correctly

```typescript
/*
Query Priority (from best to worst):
1. getByRole - Accessibility tree
2. getByLabelText - Form fields
3. getByPlaceholderText - Forms without labels
4. getByText - Non-interactive elements
5. getByDisplayValue - Current form value
6. getByAltText - Images
7. getByTitle - Elements with title attribute
8. getByTestId - Last resort
*/
```

### 3. Mock External Services, Not Your Code

```typescript
// ✅ Mock external APIs
server.use(
  rest.get('https://api.external.com/data', (req, res, ctx) => {
    return res(ctx.json({ data: 'mocked' }));
  })
);

// ✅ Mock browser APIs
Object.defineProperty(window, 'localStorage', {
  value: {
    getItem: jest.fn(),
    setItem: jest.fn()
  }
});

// ❌ Don't mock your own modules excessively
```

## Interview Questions

### Junior Level
1. **What is the testing pyramid?**
   - Many unit tests at base
   - Fewer integration tests in middle
   - Few E2E tests at top
   - Traditional testing strategy

2. **What is the testing trophy?**
   - Emphasizes integration tests
   - Static analysis at foundation
   - Fewer unit tests
   - Modern frontend approach

3. **What's the difference between unit and integration tests?**
   - Unit: Test single function/component
   - Integration: Test multiple parts together
   - Unit: Fast, isolated
   - Integration: More realistic

### Mid Level
4. **Why does the trophy emphasize integration tests?**
   - More realistic
   - Better confidence
   - Less mocking
   - Catches real bugs

5. **When should you write unit tests?**
   - Pure functions
   - Complex algorithms
   - Business logic
   - Utilities

6. **What should you mock in integration tests?**
   - External APIs
   - Browser APIs
   - Third-party services
   - Not your own code

### Senior Level
7. **Design a testing strategy for a new React application**
   - TypeScript + ESLint (foundation)
   - Unit tests for utilities (20%)
   - Integration tests for features (60%)
   - E2E for critical paths (20%)
   - MSW for API mocking
   - React Testing Library

8. **How would you convince a team to switch from pyramid to trophy?**
   - Show confidence gap
   - Demonstrate integration test value
   - Calculate test maintenance cost
   - Compare bug detection rates
   - Provide migration plan

9. **What metrics indicate a healthy test suite?**
   - High confidence in changes
   - Fast feedback (< 10 min)
   - Low flakiness (< 1%)
   - Good coverage (> 80%)
   - Tests fail when bugs introduced
   - Easy to maintain

10. **How do you balance test coverage vs development speed?**
    - Focus on high-value tests
    - Integrate into workflow
    - Automate in CI/CD
    - Write tests with features
    - Prioritize critical paths
    - Measure test ROI

## Key Takeaways

1. **Pyramid**: Many unit tests, few E2E (traditional)
2. **Trophy**: Emphasizes integration tests (modern frontend)
3. **Integration tests** provide best confidence for frontend
4. **Mock external services**, not your own code
5. **Test user behavior**, not implementation
6. **Static analysis** is free confidence (TypeScript, ESLint)
7. **E2E tests** for critical user journeys only
8. **Unit tests** for utilities and complex logic
9. **Use real dependencies** when possible
10. **Trophy better suits** React/Angular/Vue applications

## Resources

### Articles
- [Write tests. Not too many. Mostly integration.](https://kentcdodds.com/blog/write-tests) - Kent C. Dodds
- [The Testing Trophy](https://kentcdodds.com/blog/the-testing-trophy-and-testing-classifications)
- [Testing Library Philosophy](https://testing-library.com/docs/guiding-principles/)

### Libraries
- [React Testing Library](https://testing-library.com/react)
- [MSW (Mock Service Worker)](https://mswjs.io/)
- [Playwright](https://playwright.dev/)
- [Vitest](https://vitest.dev/)

### Books
- *Testing JavaScript Applications* by Lucas da Costa
- *Testing React Applications* by Kent C. Dodds (Frontend Masters)

### Videos
- [TestingJavaScript.com](https://testingjavascript.com/) - Kent C. Dodds
- [Common Testing Mistakes](https://www.youtube.com/watch?v=RTq6mmPi4t4) - Kent C. Dodds
