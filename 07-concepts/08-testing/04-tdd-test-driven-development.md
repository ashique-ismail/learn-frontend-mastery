# TDD (Test-Driven Development)

## The Idea

**In plain English:** Test-Driven Development (TDD) is a coding approach where you write a "check" (called a test) that confirms your code works correctly before you actually write the code itself — so you always know exactly what you're building and can prove it works.

**Real-world analogy:** Imagine a chef hired to create a new dish for a restaurant. Before cooking anything, the head chef writes a tasting checklist: "The soup must be hot, slightly salty, and have visible vegetables." The new chef then cooks the soup, checks it against the list, and keeps tweaking until every item is ticked off.

- The tasting checklist written before cooking = the test written before the code
- The act of cooking the soup = writing the implementation code
- Checking each item on the list = running the tests to see if the code passes

---

## Overview

Test-Driven Development (TDD) is a software development methodology where tests are written before the production code. The TDD cycle follows a red-green-refactor pattern: write a failing test (red), write minimal code to pass it (green), then improve the code (refactor). This approach results in better design, higher test coverage, and more maintainable code.

## The Red-Green-Refactor Cycle

```text
┌─────────────┐
│    RED      │  Write a failing test
│  (Write a   │
│  test that  │
│  fails)     │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   GREEN     │  Write minimal code to pass
│ (Make the   │
│  test pass) │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  REFACTOR   │  Improve code quality
│  (Clean up  │
│  the code)  │
└──────┬──────┘
       │
       └──────► Repeat
```

### Step 1: Red - Write a Failing Test

```typescript
// Step 1: RED - Write the test first
describe('StringCalculator', () => {
  it('should return 0 for empty string', () => {
    const calculator = new StringCalculator();
    expect(calculator.add('')).toBe(0);
  });
});

// Running this test will fail because StringCalculator doesn't exist yet
// Error: Cannot find name 'StringCalculator'
```

### Step 2: Green - Make It Pass

```typescript
// Step 2: GREEN - Write minimal code to make test pass
export class StringCalculator {
  add(numbers: string): number {
    return 0; // Simplest implementation that makes the test pass
  }
}

// Test now passes ✅
```

### Step 3: Refactor - Improve the Code

```typescript
// Step 3: REFACTOR - No refactoring needed yet (code is simple)
// Tests still pass ✅

// Now add another test (RED again)
describe('StringCalculator', () => {
  it('should return 0 for empty string', () => {
    const calculator = new StringCalculator();
    expect(calculator.add('')).toBe(0);
  });

  it('should return number for single number string', () => {
    const calculator = new StringCalculator();
    expect(calculator.add('1')).toBe(1);
    expect(calculator.add('5')).toBe(5);
  });
});

// Test fails ❌ (returns 0 instead of the number)

// Make it pass (GREEN)
export class StringCalculator {
  add(numbers: string): number {
    if (numbers === '') return 0;
    return parseInt(numbers, 10);
  }
}

// Tests pass ✅
```

## Complete TDD Example: Shopping Cart

### Iteration 1: Add Item

```typescript
// RED: Write first test
describe('ShoppingCart', () => {
  it('should start empty', () => {
    const cart = new ShoppingCart();
    expect(cart.getItemCount()).toBe(0);
  });
});

// GREEN: Make it pass
export class ShoppingCart {
  private items: CartItem[] = [];

  getItemCount(): number {
    return 0;
  }
}

// Tests pass ✅
```

### Iteration 2: Add Items

```typescript
// RED: Add new test
describe('ShoppingCart', () => {
  it('should start empty', () => {
    const cart = new ShoppingCart();
    expect(cart.getItemCount()).toBe(0);
  });

  it('should add items', () => {
    const cart = new ShoppingCart();
    cart.addItem({ id: '1', name: 'Book', price: 10 });
    expect(cart.getItemCount()).toBe(1);
  });
});

// Test fails ❌

// GREEN: Implement
export interface CartItem {
  id: string;
  name: string;
  price: number;
}

export class ShoppingCart {
  private items: CartItem[] = [];

  addItem(item: CartItem): void {
    this.items.push(item);
  }

  getItemCount(): number {
    return this.items.length;
  }
}

// Tests pass ✅
```

### Iteration 3: Calculate Total

```typescript
// RED: Test for total calculation
describe('ShoppingCart', () => {
  // ... previous tests

  it('should calculate total', () => {
    const cart = new ShoppingCart();
    cart.addItem({ id: '1', name: 'Book', price: 10 });
    cart.addItem({ id: '2', name: 'Pen', price: 2 });
    expect(cart.getTotal()).toBe(12);
  });
});

// Test fails ❌

// GREEN: Implement
export class ShoppingCart {
  private items: CartItem[] = [];

  addItem(item: CartItem): void {
    this.items.push(item);
  }

  getItemCount(): number {
    return this.items.length;
  }

  getTotal(): number {
    return this.items.reduce((sum, item) => sum + item.price, 0);
  }
}

// Tests pass ✅
```

### Iteration 4: Handle Quantities

```typescript
// RED: Test for quantities
describe('ShoppingCart', () => {
  it('should handle item quantities', () => {
    const cart = new ShoppingCart();
    cart.addItem({ id: '1', name: 'Book', price: 10 }, 3);
    expect(cart.getItemCount()).toBe(3);
    expect(cart.getTotal()).toBe(30);
  });
});

// Test fails ❌

// GREEN: Implement
export interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

export class ShoppingCart {
  private items: CartItem[] = [];

  addItem(item: Omit<CartItem, 'quantity'>, quantity: number = 1): void {
    this.items.push({ ...item, quantity });
  }

  getItemCount(): number {
    return this.items.reduce((sum, item) => sum + item.quantity, 0);
  }

  getTotal(): number {
    return this.items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }
}

// Tests pass ✅

// REFACTOR: Extract logic
export class ShoppingCart {
  private items: CartItem[] = [];

  addItem(item: Omit<CartItem, 'quantity'>, quantity: number = 1): void {
    this.items.push({ ...item, quantity });
  }

  getItemCount(): number {
    return this.sumItemProperty(item => item.quantity);
  }

  getTotal(): number {
    return this.sumItemProperty(item => item.price * item.quantity);
  }

  private sumItemProperty(fn: (item: CartItem) => number): number {
    return this.items.reduce((sum, item) => sum + fn(item), 0);
  }
}

// Tests still pass ✅ (refactoring doesn't change behavior)
```

## TDD with React Components

### Example: Counter Component

```tsx
// RED: Write test first
import { render, screen, fireEvent } from '@testing-library/react';
import { Counter } from './Counter';

describe('Counter', () => {
  it('should display initial count of 0', () => {
    render(<Counter />);
    expect(screen.getByText('Count: 0')).toBeInTheDocument();
  });
});

// Test fails ❌ - Counter doesn't exist

// GREEN: Minimal implementation
export const Counter: React.FC = () => {
  return <div>Count: 0</div>;
};

// Test passes ✅

// RED: Add increment functionality test
describe('Counter', () => {
  it('should display initial count of 0', () => {
    render(<Counter />);
    expect(screen.getByText('Count: 0')).toBeInTheDocument();
  });

  it('should increment count when button clicked', () => {
    render(<Counter />);
    const button = screen.getByRole('button', { name: 'Increment' });
    fireEvent.click(button);
    expect(screen.getByText('Count: 1')).toBeInTheDocument();
  });
});

// Test fails ❌ - No increment button

// GREEN: Implement increment
import { useState } from 'react';

export const Counter: React.FC = () => {
  const [count, setCount] = useState(0);

  return (
    <div>
      <div>Count: {count}</div>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
};

// Tests pass ✅

// RED: Add decrement functionality
describe('Counter', () => {
  // ... previous tests

  it('should decrement count when decrement button clicked', () => {
    render(<Counter />);
    
    // First increment to 2
    const incrementBtn = screen.getByRole('button', { name: 'Increment' });
    fireEvent.click(incrementBtn);
    fireEvent.click(incrementBtn);
    
    // Then decrement
    const decrementBtn = screen.getByRole('button', { name: 'Decrement' });
    fireEvent.click(decrementBtn);
    
    expect(screen.getByText('Count: 1')).toBeInTheDocument();
  });
});

// Test fails ❌

// GREEN: Add decrement
export const Counter: React.FC = () => {
  const [count, setCount] = useState(0);

  return (
    <div>
      <div>Count: {count}</div>
      <button onClick={() => setCount(count + 1)}>Increment</button>
      <button onClick={() => setCount(count - 1)}>Decrement</button>
    </div>
  );
};

// Tests pass ✅

// RED: Don't allow negative counts
describe('Counter', () => {
  // ... previous tests

  it('should not go below zero', () => {
    render(<Counter />);
    const decrementBtn = screen.getByRole('button', { name: 'Decrement' });
    fireEvent.click(decrementBtn);
    expect(screen.getByText('Count: 0')).toBeInTheDocument();
  });
});

// Test fails ❌ (shows -1)

// GREEN: Add validation
export const Counter: React.FC = () => {
  const [count, setCount] = useState(0);

  const increment = () => setCount(count + 1);
  const decrement = () => setCount(Math.max(0, count - 1));

  return (
    <div>
      <div>Count: {count}</div>
      <button onClick={increment}>Increment</button>
      <button onClick={decrement}>Decrement</button>
    </div>
  );
};

// All tests pass ✅
```

## TDD with Angular Services

```typescript
// RED: Write test first
describe('UserService', () => {
  let service: UserService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [UserService]
    });

    service = TestBed.inject(UserService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
  });

  it('should fetch users', () => {
    const mockUsers: User[] = [
      { id: '1', name: 'John' },
      { id: '2', name: 'Jane' }
    ];

    service.getUsers().subscribe(users => {
      expect(users).toEqual(mockUsers);
    });

    const req = httpMock.expectOne('/api/users');
    expect(req.request.method).toBe('GET');
    req.flush(mockUsers);
  });
});

// Test fails ❌ - UserService doesn't exist

// GREEN: Implement service
@Injectable()
export class UserService {
  constructor(private http: HttpClient) {}

  getUsers(): Observable<User[]> {
    return this.http.get<User[]>('/api/users');
  }
}

// Test passes ✅

// RED: Add caching test
describe('UserService', () => {
  // ... setup

  it('should cache users after first fetch', () => {
    const mockUsers: User[] = [{ id: '1', name: 'John' }];

    // First call
    service.getUsers().subscribe();
    httpMock.expectOne('/api/users').flush(mockUsers);

    // Second call - should use cache, no HTTP request
    service.getUsers().subscribe(users => {
      expect(users).toEqual(mockUsers);
    });

    httpMock.expectNone('/api/users'); // No second request
  });
});

// Test fails ❌ - Makes second HTTP request

// GREEN: Implement caching
@Injectable()
export class UserService {
  private cache: User[] | null = null;

  constructor(private http: HttpClient) {}

  getUsers(): Observable<User[]> {
    if (this.cache) {
      return of(this.cache);
    }

    return this.http.get<User[]>('/api/users').pipe(
      tap(users => this.cache = users)
    );
  }
}

// Test passes ✅

// RED: Add cache invalidation test
describe('UserService', () => {
  // ... setup

  it('should refresh cache on invalidation', () => {
    const mockUsers1: User[] = [{ id: '1', name: 'John' }];
    const mockUsers2: User[] = [{ id: '1', name: 'John' }, { id: '2', name: 'Jane' }];

    // First fetch
    service.getUsers().subscribe();
    httpMock.expectOne('/api/users').flush(mockUsers1);

    // Invalidate cache
    service.invalidateCache();

    // Second fetch should make HTTP request
    service.getUsers().subscribe(users => {
      expect(users).toEqual(mockUsers2);
    });

    httpMock.expectOne('/api/users').flush(mockUsers2);
  });
});

// Test fails ❌ - invalidateCache doesn't exist

// GREEN: Implement cache invalidation
@Injectable()
export class UserService {
  private cache: User[] | null = null;

  constructor(private http: HttpClient) {}

  getUsers(): Observable<User[]> {
    if (this.cache) {
      return of(this.cache);
    }

    return this.http.get<User[]>('/api/users').pipe(
      tap(users => this.cache = users)
    );
  }

  invalidateCache(): void {
    this.cache = null;
  }
}

// All tests pass ✅
```

## TDD with Async Operations

```typescript
// RED: Test async function
describe('fetchUserData', () => {
  it('should fetch and transform user data', async () => {
    const mockResponse = {
      id: '123',
      first_name: 'John',
      last_name: 'Doe',
      email_address: 'john@example.com'
    };

    global.fetch = jest.fn().mockResolvedValue({
      ok: true,
      json: async () => mockResponse
    });

    const result = await fetchUserData('123');

    expect(result).toEqual({
      id: '123',
      name: 'John Doe',
      email: 'john@example.com'
    });
  });
});

// Test fails ❌ - Function doesn't exist

// GREEN: Implement
export interface User {
  id: string;
  name: string;
  email: string;
}

export async function fetchUserData(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  const data = await response.json();

  return {
    id: data.id,
    name: `${data.first_name} ${data.last_name}`,
    email: data.email_address
  };
}

// Test passes ✅

// RED: Test error handling
describe('fetchUserData', () => {
  // ... previous test

  it('should throw error when fetch fails', async () => {
    global.fetch = jest.fn().mockResolvedValue({
      ok: false,
      status: 404
    });

    await expect(fetchUserData('999')).rejects.toThrow('User not found');
  });
});

// Test fails ❌ - Doesn't throw error

// GREEN: Add error handling
export async function fetchUserData(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  
  if (!response.ok) {
    throw new Error('User not found');
  }

  const data = await response.json();

  return {
    id: data.id,
    name: `${data.first_name} ${data.last_name}`,
    email: data.email_address
  };
}

// Tests pass ✅

// RED: Test retry logic
describe('fetchUserData', () => {
  // ... previous tests

  it('should retry on network error', async () => {
    global.fetch = jest.fn()
      .mockRejectedValueOnce(new Error('Network error'))
      .mockRejectedValueOnce(new Error('Network error'))
      .mockResolvedValueOnce({
        ok: true,
        json: async () => ({
          id: '123',
          first_name: 'John',
          last_name: 'Doe',
          email_address: 'john@example.com'
        })
      });

    const result = await fetchUserData('123');
    expect(result.id).toBe('123');
    expect(global.fetch).toHaveBeenCalledTimes(3);
  });
});

// Test fails ❌ - No retry logic

// GREEN: Implement retry
export async function fetchUserData(
  id: string,
  retries: number = 3
): Promise<User> {
  let lastError: Error | null = null;

  for (let i = 0; i < retries; i++) {
    try {
      const response = await fetch(`/api/users/${id}`);
      
      if (!response.ok) {
        throw new Error('User not found');
      }

      const data = await response.json();

      return {
        id: data.id,
        name: `${data.first_name} ${data.last_name}`,
        email: data.email_address
      };
    } catch (error) {
      lastError = error as Error;
      if (i < retries - 1) {
        await new Promise(resolve => setTimeout(resolve, 1000));
      }
    }
  }

  throw lastError;
}

// Tests pass ✅
```

## Benefits of TDD

### 1. Better Design

```typescript
// TDD forces you to think about API design first
describe('PasswordValidator', () => {
  it('should validate password length', () => {
    const validator = new PasswordValidator({ minLength: 8 });
    expect(validator.validate('short')).toEqual({
      valid: false,
      errors: ['Password must be at least 8 characters']
    });
  });
});

// Writing the test first makes you design a clean API
// Before writing any implementation, you've defined:
// - Constructor takes configuration object
// - validate() returns object with valid flag and errors array
// This is better than designing as you implement
```

### 2. Living Documentation

```typescript
// Tests serve as executable documentation
describe('DateFormatter', () => {
  it('should format date as MM/DD/YYYY by default', () => {
    const formatter = new DateFormatter();
    expect(formatter.format(new Date('2025-01-27'))).toBe('01/27/2025');
  });

  it('should format date with custom pattern', () => {
    const formatter = new DateFormatter();
    expect(formatter.format(new Date('2025-01-27'), 'YYYY-MM-DD')).toBe('2025-01-27');
  });

  it('should handle invalid dates', () => {
    const formatter = new DateFormatter();
    expect(() => formatter.format(new Date('invalid'))).toThrow('Invalid date');
  });
});

// Anyone reading these tests understands exactly how DateFormatter works
```

### 3. Confidence to Refactor

```typescript
// Original implementation
export class Calculator {
  add(a: number, b: number): number {
    let result = 0;
    result = result + a;
    result = result + b;
    return result;
  }
}

// Tests pass ✅

// Refactor with confidence
export class Calculator {
  add(a: number, b: number): number {
    return a + b; // Much simpler!
  }
}

// Tests still pass ✅
// You can refactor freely because tests verify behavior
```

### 4. Regression Prevention

```typescript
// You fixed a bug once
describe('calculateDiscount', () => {
  it('should handle zero prices correctly', () => {
    // This bug was found in production
    expect(calculateDiscount(0, 0.2)).toBe(0);
  });
});

// The test prevents the bug from returning
// If someone changes the code and breaks it, the test fails immediately
```

## TDD Patterns

### Test List Pattern

```typescript
// Before starting, write a list of tests you need
/*
TODO: ShoppingCart tests
- [x] should start empty
- [x] should add items
- [x] should calculate total
- [x] should handle quantities
- [ ] should remove items
- [ ] should clear cart
- [ ] should handle duplicate items
- [ ] should apply discounts
- [ ] should calculate tax
*/

// Work through the list one test at a time
```

### Fake It Till You Make It

```typescript
// RED
it('should calculate fibonacci', () => {
  expect(fibonacci(0)).toBe(0);
  expect(fibonacci(1)).toBe(1);
  expect(fibonacci(5)).toBe(5);
});

// GREEN - Fake it first
export function fibonacci(n: number): number {
  if (n === 0) return 0;
  if (n === 1) return 1;
  if (n === 5) return 5; // Hardcoded!
  return 0;
}

// Tests pass, now add more tests to force real implementation
it('should calculate fibonacci for various numbers', () => {
  expect(fibonacci(0)).toBe(0);
  expect(fibonacci(1)).toBe(1);
  expect(fibonacci(5)).toBe(5);
  expect(fibonacci(6)).toBe(8); // Forces real implementation
  expect(fibonacci(7)).toBe(13);
});

// GREEN - Real implementation
export function fibonacci(n: number): number {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}
```

### Triangulation

```typescript
// Start with one example
it('should sum numbers', () => {
  expect(sum([1])).toBe(1);
});

// GREEN
export function sum(numbers: number[]): number {
  return 1; // Hardcoded
}

// Add another example to triangulate toward general solution
it('should sum numbers', () => {
  expect(sum([1])).toBe(1);
  expect(sum([1, 2])).toBe(3);
});

// GREEN - Now implement general solution
export function sum(numbers: number[]): number {
  return numbers.reduce((acc, n) => acc + n, 0);
}

// Add edge cases
it('should sum numbers', () => {
  expect(sum([])).toBe(0);
  expect(sum([1])).toBe(1);
  expect(sum([1, 2])).toBe(3);
  expect(sum([1, 2, 3, 4])).toBe(10);
});
```

## Common Mistakes

### 1. Writing Tests After Implementation

```typescript
// ❌ BAD: Code-first approach
export class UserService {
  async createUser(data: CreateUserDTO): Promise<User> {
    // Implementation written first
    const user = await this.repository.create(data);
    await this.emailService.sendWelcome(user.email);
    return user;
  }
}

// Then writing tests - loses TDD benefits

// ✅ GOOD: Test-first approach
describe('UserService', () => {
  it('should create user and send welcome email', async () => {
    // Test drives the design
  });
});
```

### 2. Testing Too Much at Once

```typescript
// ❌ BAD: Big leap in test complexity
describe('ComplexCalculator', () => {
  it('should handle complex expressions', () => {
    expect(calculate('(2 + 3) * 4 - 5 / 2')).toBe(17.5);
  });
});

// ✅ GOOD: Baby steps
describe('ComplexCalculator', () => {
  it('should add two numbers', () => {
    expect(calculate('2 + 3')).toBe(5);
  });

  it('should subtract numbers', () => {
    expect(calculate('5 - 3')).toBe(2);
  });

  it('should multiply numbers', () => {
    expect(calculate('2 * 3')).toBe(6);
  });

  // Build up gradually
});
```

### 3. Not Refactoring

```typescript
// ❌ BAD: Tests pass but code is messy
export class OrderProcessor {
  process(order: Order): void {
    if (order.items.length > 0) {
      let total = 0;
      for (let i = 0; i < order.items.length; i++) {
        total = total + order.items[i].price * order.items[i].quantity;
      }
      if (order.coupon) {
        if (order.coupon.type === 'PERCENT') {
          total = total - (total * order.coupon.value / 100);
        } else if (order.coupon.type === 'FIXED') {
          total = total - order.coupon.value;
        }
      }
      order.total = total;
    }
  }
}

// ✅ GOOD: Refactor after tests pass
export class OrderProcessor {
  process(order: Order): void {
    order.total = this.calculateTotal(order);
  }

  private calculateTotal(order: Order): number {
    const subtotal = this.calculateSubtotal(order.items);
    const discount = this.calculateDiscount(subtotal, order.coupon);
    return subtotal - discount;
  }

  private calculateSubtotal(items: OrderItem[]): number {
    return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }

  private calculateDiscount(subtotal: number, coupon?: Coupon): number {
    if (!coupon) return 0;
    return coupon.type === 'PERCENT'
      ? subtotal * coupon.value / 100
      : coupon.value;
  }
}
```

## Best Practices

1. **Write Test First**: Always red-green-refactor, never code-first
2. **Take Small Steps**: Baby steps, one test at a time
3. **Run Tests Frequently**: After every code change
4. **Refactor Regularly**: Clean up code while tests are green
5. **Test Behavior**: Focus on what, not how
6. **Keep Tests Fast**: Unit tests should run in milliseconds
7. **Make Tests Readable**: Tests are documentation
8. **One Assertion Per Test**: Test one thing at a time
9. **Follow AAA Pattern**: Arrange, Act, Assert
10. **Commit on Green**: Only commit when tests pass

## When to Use TDD

**Use TDD when:**
- Starting new features from scratch
- Working on complex business logic
- Fixing bugs (write failing test first)
- Refactoring existing code
- Learning a new API or library

**TDD might be challenging for:**
- Prototyping and experimentation
- UI-heavy work (but still possible)
- Working with unfamiliar third-party APIs
- Very simple trivial code
- Spike solutions

## Interview Questions

1. **What is the TDD cycle?**
   - Red (write failing test), Green (make it pass), Refactor (improve code). Repeat.

2. **What are the benefits of TDD?**
   - Better design, living documentation, confidence to refactor, regression prevention, higher test coverage.

3. **How is TDD different from writing tests after code?**
   - TDD drives design, ensures testability, prevents over-engineering, and guarantees test coverage.

4. **What is the "red" phase in TDD?**
   - Writing a failing test before any implementation code exists. Confirms the test can actually fail.

5. **Why refactor in TDD?**
   - To improve code quality while tests guarantee behavior doesn't change. Tests provide safety net for refactoring.

## Key Takeaways

1. TDD follows red-green-refactor cycle
2. Write tests before implementation code
3. Take small steps, one test at a time
4. Tests drive better design decisions
5. Refactor freely with tests as safety net
6. Tests serve as executable documentation
7. TDD prevents over-engineering
8. Ensures high test coverage naturally
9. Makes code more maintainable
10. Builds confidence in code changes

## Resources

- **"Test-Driven Development: By Example"** by Kent Beck
- **"Growing Object-Oriented Software, Guided by Tests"** by Steve Freeman
- **Kent C. Dodds - Testing Blog**: https://kentcdodds.com/blog
- **Uncle Bob - TDD**: https://blog.cleancoder.com/
- **Martin Fowler - TDD**: https://martinfowler.com/articles/practical-test-pyramid.html
- **TDD Kata Exercises**: http://codingdojo.org/kata/
