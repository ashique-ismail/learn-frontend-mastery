# Testing Best Practices

## The Idea

**In plain English:** Testing best practices are a set of rules and habits that help you write tests that are easy to understand, reliable every time they run, and quick to fix when code changes. A "test" is a small piece of code you write to check that another piece of code works the way you expect.

**Real-world analogy:** Imagine a factory quality inspector who checks products coming off an assembly line. Each inspector follows a checklist: they check one thing at a time, they label each check clearly ("does the lid seal?"), they reset the product to its original state before the next check, and they never skip a step just because the last one passed.

- The inspector's checklist = the test suite (collection of tests)
- Each individual check on the checklist = a single test case
- Resetting the product to its original state before each check = clearing shared state between tests (beforeEach / afterEach)
- Labeling each check clearly = writing descriptive test names

---

## Overview

Writing effective tests requires following established principles and patterns that make tests maintainable, reliable, and valuable. This guide covers best practices for test naming, organization, avoiding flaky tests, maintaining test quality, and building a sustainable testing culture.

## Test Naming Conventions

### Descriptive Test Names

```typescript
// ❌ BAD: Vague names
describe('UserService', () => {
  it('test1', () => {});
  it('works', () => {});
  it('should work correctly', () => {});
});

// ✅ GOOD: Descriptive names explain what's being tested
describe('UserService', () => {
  it('should create user with hashed password', () => {});
  it('should throw error when email already exists', () => {});
  it('should send welcome email after registration', () => {});
});
```

### Given-When-Then Pattern

```typescript
// ✅ Clear structure in test name
describe('ShoppingCart', () => {
  it('given empty cart, when adding item, then cart count increases', () => {
    // Arrange (Given)
    const cart = new ShoppingCart();
    
    // Act (When)
    cart.addItem({ id: '1', name: 'Book', price: 10 });
    
    // Assert (Then)
    expect(cart.getItemCount()).toBe(1);
  });

  it('given cart with items, when removing all items, then cart is empty', () => {
    const cart = new ShoppingCart();
    cart.addItem({ id: '1', name: 'Book', price: 10 });
    
    cart.removeAllItems();
    
    expect(cart.isEmpty()).toBe(true);
  });
});
```

### BDD-Style Naming

```typescript
// ✅ Behavior-driven test names
describe('User Authentication', () => {
  describe('when logging in', () => {
    it('should redirect to dashboard on successful login', () => {});
    it('should show error message on invalid credentials', () => {});
    it('should lock account after 5 failed attempts', () => {});
  });

  describe('when logging out', () => {
    it('should clear session data', () => {});
    it('should redirect to login page', () => {});
  });
});
```

### Test Name Templates

```typescript
// Template: should [expected behavior] when [condition]
it('should return 400 when email is invalid', () => {});
it('should cache results when called multiple times', () => {});
it('should throw error when user not found', () => {});

// Template: [method name] should [expected behavior]
it('calculateTotal should sum item prices', () => {});
it('validateEmail should reject emails without @ symbol', () => {});
it('formatDate should handle null values', () => {});

// Template: given [context], should [expected behavior]
it('given empty input, should return default value', () => {});
it('given admin user, should allow access to settings', () => {});
it('given expired token, should throw unauthorized error', () => {});
```

## Test Organization

### Arrange-Act-Assert (AAA) Pattern

```typescript
// ✅ GOOD: Clear AAA structure
describe('OrderService', () => {
  it('should calculate order total with tax', () => {
    // Arrange - Set up test data and dependencies
    const order = {
      items: [
        { price: 10, quantity: 2 },
        { price: 5, quantity: 3 }
      ]
    };
    const taxRate = 0.1;
    const service = new OrderService();
    
    // Act - Execute the function being tested
    const result = service.calculateTotal(order, taxRate);
    
    // Assert - Verify the outcome
    expect(result).toEqual({
      subtotal: 35,
      tax: 3.5,
      total: 38.5
    });
  });
});

// ❌ BAD: Mixed structure, hard to follow
describe('OrderService', () => {
  it('should work', () => {
    const service = new OrderService();
    expect(service.calculateTotal({
      items: [{ price: 10, quantity: 2 }]
    }, 0.1).total).toBe(22);
    expect(service.calculateTotal({
      items: [{ price: 5, quantity: 3 }]
    }, 0.1).total).toBe(16.5);
  });
});
```

### Group Related Tests

```typescript
// ✅ GOOD: Logical grouping with nested describes
describe('UserValidator', () => {
  describe('email validation', () => {
    it('should accept valid email addresses', () => {});
    it('should reject emails without @', () => {});
    it('should reject emails without domain', () => {});
    it('should handle international characters', () => {});
  });

  describe('password validation', () => {
    it('should require minimum 8 characters', () => {});
    it('should require at least one number', () => {});
    it('should require at least one special character', () => {});
    it('should reject common passwords', () => {});
  });

  describe('age validation', () => {
    it('should require age 18 or older', () => {});
    it('should reject negative ages', () => {});
    it('should reject unrealistic ages', () => {});
  });
});
```

### Setup and Teardown

```typescript
// ✅ GOOD: Proper use of lifecycle hooks
describe('DatabaseService', () => {
  let db: Database;
  let service: DatabaseService;

  // Runs once before all tests
  beforeAll(async () => {
    db = await createTestDatabase();
  });

  // Runs before each test
  beforeEach(() => {
    service = new DatabaseService(db);
  });

  // Runs after each test
  afterEach(async () => {
    await db.clear(); // Clean up test data
  });

  // Runs once after all tests
  afterAll(async () => {
    await db.close();
  });

  it('should insert record', async () => {
    await service.insert('users', { name: 'John' });
    const count = await db.count('users');
    expect(count).toBe(1);
  });

  it('should query records', async () => {
    await service.insert('users', { name: 'John' });
    const users = await service.query('users');
    expect(users).toHaveLength(1);
  });
});

// ❌ BAD: Manual setup in each test
describe('DatabaseService', () => {
  it('should insert record', async () => {
    const db = await createTestDatabase();
    const service = new DatabaseService(db);
    // ... test logic
    await db.close();
  });

  it('should query records', async () => {
    const db = await createTestDatabase(); // Repeated setup
    const service = new DatabaseService(db);
    // ... test logic
    await db.close();
  });
});
```

## Avoiding Flaky Tests

### 1. Eliminate Time Dependencies

```typescript
// ❌ BAD: Depends on current time
export function isBusinessHours(): boolean {
  const hour = new Date().getHours();
  return hour >= 9 && hour < 17;
}

it('should return true during business hours', () => {
  expect(isBusinessHours()).toBe(true); // Fails outside 9-5!
});

// ✅ GOOD: Inject time dependency
export function isBusinessHours(date: Date = new Date()): boolean {
  const hour = date.getHours();
  return hour >= 9 && hour < 17;
}

it('should return true during business hours', () => {
  const businessTime = new Date('2025-01-27T10:00:00');
  expect(isBusinessHours(businessTime)).toBe(true);
});

it('should return false outside business hours', () => {
  const afterHours = new Date('2025-01-27T18:00:00');
  expect(isBusinessHours(afterHours)).toBe(false);
});
```

### 2. Use Fake Timers for Delays

```typescript
// ❌ BAD: Real delays make tests slow and flaky
it('should debounce function calls', async () => {
  const fn = jest.fn();
  const debounced = debounce(fn, 1000);
  
  debounced();
  await new Promise(resolve => setTimeout(resolve, 1100)); // Real delay
  
  expect(fn).toHaveBeenCalledTimes(1);
}, 2000);

// ✅ GOOD: Fake timers for fast, deterministic tests
it('should debounce function calls', () => {
  jest.useFakeTimers();
  
  const fn = jest.fn();
  const debounced = debounce(fn, 1000);
  
  debounced();
  jest.advanceTimersByTime(1000);
  
  expect(fn).toHaveBeenCalledTimes(1);
  
  jest.useRealTimers();
});
```

### 3. Avoid Test Order Dependencies

```typescript
// ❌ BAD: Tests depend on execution order
describe('Counter', () => {
  const counter = new Counter(); // Shared instance

  it('should increment', () => {
    counter.increment();
    expect(counter.getValue()).toBe(1);
  });

  it('should decrement', () => {
    counter.decrement(); // Depends on previous test!
    expect(counter.getValue()).toBe(0);
  });
});

// ✅ GOOD: Each test is independent
describe('Counter', () => {
  let counter: Counter;

  beforeEach(() => {
    counter = new Counter(); // Fresh instance for each test
  });

  it('should increment', () => {
    counter.increment();
    expect(counter.getValue()).toBe(1);
  });

  it('should decrement from specific value', () => {
    counter.setValue(5);
    counter.decrement();
    expect(counter.getValue()).toBe(4);
  });
});
```

### 4. Handle Async Operations Properly

```typescript
// ❌ BAD: Race conditions
it('should load data', () => {
  fetchData().then(data => {
    expect(data).toBeDefined(); // Might not run before test ends
  });
});

// ✅ GOOD: Wait for promises
it('should load data', async () => {
  const data = await fetchData();
  expect(data).toBeDefined();
});

// ✅ GOOD: Use testing library utilities
it('should display loaded data', async () => {
  render(<DataComponent />);
  
  await waitFor(() => {
    expect(screen.getByText('Data loaded')).toBeInTheDocument();
  });
});
```

### 5. Avoid Hard-Coded Timeouts

```typescript
// ❌ BAD: Arbitrary timeout
it('should eventually update', async () => {
  triggerUpdate();
  await new Promise(resolve => setTimeout(resolve, 500)); // Magic number
  expect(getState()).toBe('updated');
});

// ✅ GOOD: Wait for condition
it('should eventually update', async () => {
  triggerUpdate();
  
  await waitFor(() => {
    expect(getState()).toBe('updated');
  }, { timeout: 5000 });
});

// ✅ GOOD: Use testing library's built-in waiting
it('should display updated content', async () => {
  render(<Component />);
  fireEvent.click(screen.getByText('Update'));
  
  await screen.findByText('Updated content'); // Automatically waits
});
```

### 6. Isolate Global State

```typescript
// ❌ BAD: Tests share global state
let globalCache = {};

describe('CacheService', () => {
  it('should cache values', () => {
    globalCache['key'] = 'value';
    expect(globalCache['key']).toBe('value');
  });

  it('should retrieve cached values', () => {
    // Depends on previous test!
    expect(globalCache['key']).toBe('value');
  });
});

// ✅ GOOD: Clean up between tests
describe('CacheService', () => {
  let cache: Map<string, any>;

  beforeEach(() => {
    cache = new Map();
  });

  afterEach(() => {
    cache.clear();
  });

  it('should cache values', () => {
    cache.set('key', 'value');
    expect(cache.get('key')).toBe('value');
  });

  it('should retrieve cached values', () => {
    cache.set('key', 'value');
    expect(cache.get('key')).toBe('value');
  });
});
```

## Test Data Management

### Test Data Builders

```typescript
// ✅ GOOD: Builder pattern for test data
class UserBuilder {
  private user: Partial<User> = {
    id: '123',
    name: 'Test User',
    email: 'test@example.com',
    role: 'user',
    active: true
  };

  withId(id: string): this {
    this.user.id = id;
    return this;
  }

  withEmail(email: string): this {
    this.user.email = email;
    return this;
  }

  asAdmin(): this {
    this.user.role = 'admin';
    return this;
  }

  inactive(): this {
    this.user.active = false;
    return this;
  }

  build(): User {
    return this.user as User;
  }
}

// Usage
describe('UserService', () => {
  it('should allow admin users to delete accounts', () => {
    const admin = new UserBuilder().asAdmin().build();
    expect(service.canDelete(admin)).toBe(true);
  });

  it('should not allow inactive users to login', () => {
    const inactiveUser = new UserBuilder().inactive().build();
    expect(service.canLogin(inactiveUser)).toBe(false);
  });
});
```

### Factory Functions

```typescript
// ✅ GOOD: Factory functions for common test data
function createMockUser(overrides?: Partial<User>): User {
  return {
    id: '123',
    name: 'Test User',
    email: 'test@example.com',
    role: 'user',
    createdAt: new Date('2025-01-01'),
    ...overrides
  };
}

function createMockOrder(overrides?: Partial<Order>): Order {
  return {
    id: '456',
    userId: '123',
    items: [],
    total: 0,
    status: 'pending',
    ...overrides
  };
}

// Usage
it('should calculate order total', () => {
  const order = createMockOrder({
    items: [
      { price: 10, quantity: 2 },
      { price: 5, quantity: 3 }
    ]
  });

  expect(calculateTotal(order)).toBe(35);
});
```

### Fixture Files

```typescript
// fixtures/users.ts
export const mockUsers = {
  admin: {
    id: 'admin-1',
    name: 'Admin User',
    email: 'admin@example.com',
    role: 'admin'
  },
  regularUser: {
    id: 'user-1',
    name: 'Regular User',
    email: 'user@example.com',
    role: 'user'
  },
  inactiveUser: {
    id: 'user-2',
    name: 'Inactive User',
    email: 'inactive@example.com',
    role: 'user',
    active: false
  }
};

// Usage
import { mockUsers } from './fixtures/users';

it('should grant admin access', () => {
  expect(hasAccess(mockUsers.admin, 'admin-panel')).toBe(true);
});
```

## Testing Async Code

### Async/Await Pattern

```typescript
// ✅ GOOD: Clear async testing
describe('UserAPI', () => {
  it('should fetch user by id', async () => {
    const mockUser = { id: '123', name: 'John' };
    jest.spyOn(api, 'get').mockResolvedValue(mockUser);

    const user = await userService.fetchUser('123');

    expect(user).toEqual(mockUser);
    expect(api.get).toHaveBeenCalledWith('/users/123');
  });

  it('should handle fetch errors', async () => {
    jest.spyOn(api, 'get').mockRejectedValue(new Error('Network error'));

    await expect(userService.fetchUser('123'))
      .rejects
      .toThrow('Network error');
  });
});
```

### Testing Promises

```typescript
// ✅ GOOD: Testing promise chains
it('should process data pipeline', () => {
  const promise = fetchData()
    .then(transform)
    .then(validate);

  return expect(promise).resolves.toEqual(expectedResult);
});

// Alternative with async/await
it('should process data pipeline', async () => {
  const data = await fetchData();
  const transformed = await transform(data);
  const validated = await validate(transformed);

  expect(validated).toEqual(expectedResult);
});
```

### Testing Callbacks

```typescript
// ✅ GOOD: Testing callback-based code
it('should call callback with data', (done) => {
  fetchDataWithCallback((error, data) => {
    expect(error).toBeNull();
    expect(data).toEqual({ id: '123' });
    done(); // Signal test completion
  });
});

// Better: Convert to promise
it('should fetch data', async () => {
  const data = await new Promise((resolve, reject) => {
    fetchDataWithCallback((error, data) => {
      if (error) reject(error);
      else resolve(data);
    });
  });

  expect(data).toEqual({ id: '123' });
});
```

## Mocking Best Practices

### Mock at the Right Level

```typescript
// ❌ BAD: Mocking too low level
it('should save user', async () => {
  const mockFs = jest.fn();
  jest.mock('fs', () => ({ writeFile: mockFs }));
  
  await userService.saveUser(user);
  expect(mockFs).toHaveBeenCalled();
});

// ✅ GOOD: Mock at architectural boundary
it('should save user', async () => {
  const mockRepository = {
    save: jest.fn().mockResolvedValue(user)
  };

  const service = new UserService(mockRepository);
  await service.saveUser(user);

  expect(mockRepository.save).toHaveBeenCalledWith(user);
});
```

### Verify Interactions

```typescript
// ✅ GOOD: Verify important interactions
it('should send welcome email after registration', async () => {
  const emailService = {
    send: jest.fn().mockResolvedValue(undefined)
  };

  const service = new UserService(userRepo, emailService);
  await service.register(userData);

  expect(emailService.send).toHaveBeenCalledWith(
    userData.email,
    'Welcome!',
    expect.any(String)
  );
});
```

### Reset Mocks Between Tests

```typescript
// ✅ GOOD: Clean mocks between tests
describe('UserService', () => {
  let mockRepository: jest.Mocked<UserRepository>;

  beforeEach(() => {
    mockRepository = {
      findById: jest.fn(),
      save: jest.fn(),
      delete: jest.fn()
    } as any;
  });

  afterEach(() => {
    jest.clearAllMocks();
  });

  it('should find user', async () => {
    mockRepository.findById.mockResolvedValue(user);
    // Test logic
  });

  it('should save user', async () => {
    mockRepository.save.mockResolvedValue(user);
    // Test logic - clean mock state
  });
});
```

## Test Maintenance

### Refactor Tests Like Production Code

```typescript
// ❌ BAD: Duplicated test logic
describe('Calculator', () => {
  it('should add positive numbers', () => {
    const calc = new Calculator();
    const result = calc.add(2, 3);
    expect(result).toBe(5);
  });

  it('should add negative numbers', () => {
    const calc = new Calculator();
    const result = calc.add(-2, -3);
    expect(result).toBe(-5);
  });

  it('should add mixed numbers', () => {
    const calc = new Calculator();
    const result = calc.add(-2, 3);
    expect(result).toBe(1);
  });
});

// ✅ GOOD: DRY tests with helpers
describe('Calculator', () => {
  let calc: Calculator;

  beforeEach(() => {
    calc = new Calculator();
  });

  test.each([
    [2, 3, 5],
    [-2, -3, -5],
    [-2, 3, 1],
    [0, 5, 5]
  ])('add(%i, %i) should return %i', (a, b, expected) => {
    expect(calc.add(a, b)).toBe(expected);
  });
});
```

### Keep Tests Simple

```typescript
// ❌ BAD: Complex test logic
it('should process orders', () => {
  const orders = generateOrders(10);
  const processed = [];
  
  for (let i = 0; i < orders.length; i++) {
    if (orders[i].status === 'pending') {
      const result = processOrder(orders[i]);
      if (result.success) {
        processed.push(result);
      }
    }
  }

  expect(processed.length).toBeGreaterThan(0);
});

// ✅ GOOD: Simple, straightforward test
it('should process pending orders', () => {
  const order = createMockOrder({ status: 'pending' });
  
  const result = processOrder(order);
  
  expect(result.success).toBe(true);
  expect(result.status).toBe('completed');
});
```

### Update Tests When Refactoring

```typescript
// Code refactored from:
class OrderService {
  calculateTotal(items: Item[]): number {
    return items.reduce((sum, item) => sum + item.price, 0);
  }
}

// To:
class OrderService {
  calculateSubtotal(items: Item[]): number {
    return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }

  calculateTotal(items: Item[], taxRate: number): number {
    const subtotal = this.calculateSubtotal(items);
    return subtotal + (subtotal * taxRate);
  }
}

// ✅ GOOD: Update tests to match new interface
describe('OrderService', () => {
  // Old test removed
  // it('should calculate total', () => { ... });

  // New tests for updated interface
  it('should calculate subtotal', () => {
    const items = [{ price: 10, quantity: 2 }];
    expect(service.calculateSubtotal(items)).toBe(20);
  });

  it('should calculate total with tax', () => {
    const items = [{ price: 10, quantity: 2 }];
    expect(service.calculateTotal(items, 0.1)).toBe(22);
  });
});
```

## Common Mistakes

### 1. Testing Implementation Instead of Behavior

```typescript
// ❌ BAD: Testing implementation
it('should use bubble sort', () => {
  const spy = jest.spyOn(sorter, 'bubbleSort');
  sorter.sort([3, 1, 2]);
  expect(spy).toHaveBeenCalled();
});

// ✅ GOOD: Testing behavior
it('should sort array in ascending order', () => {
  const result = sorter.sort([3, 1, 2]);
  expect(result).toEqual([1, 2, 3]);
});
```

### 2. Too Many Assertions Per Test

```typescript
// ❌ BAD: Testing too much in one test
it('should handle user registration', async () => {
  const user = await registerUser(userData);
  expect(user.id).toBeDefined();
  expect(user.email).toBe(userData.email);
  expect(user.password).not.toBe(userData.password); // Hashed
  expect(await db.findUser(user.id)).toBeDefined();
  expect(emailService.send).toHaveBeenCalled();
  expect(analytics.track).toHaveBeenCalledWith('user_registered');
});

// ✅ GOOD: Split into focused tests
describe('User Registration', () => {
  it('should create user with generated id', async () => {
    const user = await registerUser(userData);
    expect(user.id).toBeDefined();
  });

  it('should hash user password', async () => {
    const user = await registerUser(userData);
    expect(user.password).not.toBe(userData.password);
  });

  it('should persist user to database', async () => {
    const user = await registerUser(userData);
    expect(await db.findUser(user.id)).toBeDefined();
  });

  it('should send welcome email', async () => {
    await registerUser(userData);
    expect(emailService.send).toHaveBeenCalled();
  });
});
```

### 3. Not Testing Edge Cases

```typescript
// ❌ BAD: Only testing happy path
describe('divide', () => {
  it('should divide numbers', () => {
    expect(divide(10, 2)).toBe(5);
  });
});

// ✅ GOOD: Testing edge cases
describe('divide', () => {
  it('should divide positive numbers', () => {
    expect(divide(10, 2)).toBe(5);
  });

  it('should handle division by zero', () => {
    expect(() => divide(10, 0)).toThrow('Division by zero');
  });

  it('should handle negative numbers', () => {
    expect(divide(-10, 2)).toBe(-5);
    expect(divide(10, -2)).toBe(-5);
  });

  it('should handle decimal results', () => {
    expect(divide(5, 2)).toBe(2.5);
  });

  it('should handle zero dividend', () => {
    expect(divide(0, 5)).toBe(0);
  });
});
```

## Best Practices Summary

1. **Write descriptive test names** that explain what's being tested
2. **Follow AAA pattern** for clear test structure
3. **Keep tests independent** - no shared state or order dependencies
4. **Make tests deterministic** - no random values or time dependencies
5. **Use fake timers** for time-based functionality
6. **Clean up after tests** - reset mocks, clear state
7. **Test behavior, not implementation** - focus on what, not how
8. **Keep tests simple** - no complex logic in tests
9. **One logical assertion per test** - test one thing at a time
10. **Avoid flaky tests** - handle async properly, eliminate race conditions
11. **Use test data builders** for maintainable test data
12. **Mock at architectural boundaries** not low-level details
13. **Test edge cases and errors** not just happy paths
14. **Refactor tests** like production code
15. **Update tests when refactoring** keep tests in sync with code

## Interview Questions

1. **What makes a test flaky?**
   - Non-deterministic behavior: time dependencies, race conditions, order dependencies, random values, external dependencies.

2. **What is the AAA pattern?**
   - Arrange (setup), Act (execute), Assert (verify). Provides clear test structure.

3. **How do you handle time-dependent code in tests?**
   - Inject time dependencies, use fake timers, mock Date/time functions for deterministic tests.

4. **What's wrong with testing implementation details?**
   - Tests become brittle, break during refactoring, don't verify actual behavior or user value.

5. **How do you avoid test interdependencies?**
   - Fresh state for each test, proper setup/teardown, no shared mutable state, tests runnable in any order.

## Key Takeaways

1. Descriptive names make tests serve as documentation
2. AAA pattern provides consistent structure
3. Independent tests prevent mysterious failures
4. Deterministic tests are reliable and valuable
5. Fake timers make time-based tests fast and stable
6. Clean up between tests ensures isolation
7. Test behavior, not implementation details
8. Simple tests are maintainable tests
9. Edge cases reveal bugs happy paths miss
10. Quality tests provide confidence to refactor

## Resources

- **Jest Best Practices**: https://jestjs.io/docs/getting-started
- **Testing Library Principles**: https://testing-library.com/docs/guiding-principles
- **Martin Fowler - Testing**: https://martinfowler.com/testing/
- **Kent C. Dodds - Testing Blog**: https://kentcdodds.com/blog/
- **Google Testing Blog**: https://testing.googleblog.com/
- **"Growing Object-Oriented Software, Guided by Tests"** by Steve Freeman
- **"Unit Testing Principles, Practices, and Patterns"** by Vladimir Khorikov
