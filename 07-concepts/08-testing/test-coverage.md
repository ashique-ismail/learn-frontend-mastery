# Test Coverage

## Overview

Test coverage measures which parts of your code are executed by tests. While coverage metrics provide useful feedback, they don't guarantee test quality. High coverage doesn't mean good tests, and chasing 100% coverage often leads to wasteful testing. Understanding what coverage means, what it doesn't mean, and how to use it effectively is crucial for a healthy testing strategy.

## Coverage Metrics

### Line Coverage

Percentage of code lines executed by tests:

```typescript
// Function to test
export function calculateDiscount(price: number, couponCode?: string): number {
  let discount = 0;                              // Line 1
  
  if (couponCode === 'SAVE10') {                 // Line 2
    discount = price * 0.1;                      // Line 3
  } else if (couponCode === 'SAVE20') {          // Line 4
    discount = price * 0.2;                      // Line 5
  }                                              // Line 6
  
  return price - discount;                       // Line 7
}

// Test with incomplete coverage
describe('calculateDiscount', () => {
  it('should apply SAVE10 coupon', () => {
    expect(calculateDiscount(100, 'SAVE10')).toBe(90);
  });
});

// Coverage: 5/7 lines = 71% line coverage
// Lines 1, 2, 3, 6, 7 executed
// Lines 4, 5 not executed
```

### Branch Coverage

Percentage of decision branches executed:

```typescript
export function calculateDiscount(price: number, couponCode?: string): number {
  let discount = 0;
  
  if (couponCode === 'SAVE10') {     // Branch 1: true
    discount = price * 0.1;          // Branch 1: false
  } else if (couponCode === 'SAVE20') { // Branch 2: true
    discount = price * 0.2;          // Branch 2: false
  }
  
  return price - discount;
}

// Same test as above
// Branch Coverage: 2/4 branches = 50%
// Branch 1 true: ✅ tested
// Branch 1 false → Branch 2: ❌ not tested
// Branch 2 true: ❌ not tested
// Branch 2 false: ❌ not tested

// Full branch coverage:
describe('calculateDiscount', () => {
  it('should apply SAVE10 coupon', () => {
    expect(calculateDiscount(100, 'SAVE10')).toBe(90);
  });

  it('should apply SAVE20 coupon', () => {
    expect(calculateDiscount(100, 'SAVE20')).toBe(80);
  });

  it('should apply no discount without coupon', () => {
    expect(calculateDiscount(100)).toBe(100);
  });

  it('should apply no discount with invalid coupon', () => {
    expect(calculateDiscount(100, 'INVALID')).toBe(100);
  });
});

// Branch Coverage: 4/4 = 100% ✅
```

### Function Coverage

Percentage of functions called:

```typescript
// utils.ts
export function add(a: number, b: number): number {
  return a + b;
}

export function subtract(a: number, b: number): number {
  return a - b;
}

export function multiply(a: number, b: number): number {
  return a * b;
}

export function divide(a: number, b: number): number {
  return a / b;
}

// Test file
describe('utils', () => {
  it('should add numbers', () => {
    expect(add(2, 3)).toBe(5);
  });

  it('should subtract numbers', () => {
    expect(subtract(5, 3)).toBe(2);
  });
});

// Function Coverage: 2/4 = 50%
// add: ✅ tested
// subtract: ✅ tested
// multiply: ❌ not tested
// divide: ❌ not tested
```

### Statement Coverage

Percentage of statements executed:

```typescript
export function processOrder(order: Order): ProcessResult {
  const total = calculateTotal(order);        // Statement 1
  const tax = total * 0.1;                    // Statement 2
  const finalAmount = total + tax;            // Statement 3
  
  if (finalAmount > 1000) {                   // Statement 4
    sendToManualReview(order);                // Statement 5
  }
  
  return { total: finalAmount, tax };         // Statement 6
}

// Test
it('should process order', () => {
  const order = { items: [{ price: 100 }] };
  const result = processOrder(order);
  expect(result.total).toBe(110);
});

// Statement Coverage: 5/6 = 83%
// Statement 5 (sendToManualReview) not executed
```

## Generating Coverage Reports

### Jest Configuration

```javascript
// jest.config.js
module.exports = {
  collectCoverage: true,
  coverageDirectory: 'coverage',
  collectCoverageFrom: [
    'src/**/*.{js,jsx,ts,tsx}',
    '!src/**/*.d.ts',
    '!src/**/*.stories.tsx',
    '!src/index.tsx',
  ],
  coverageThresholds: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
  coverageReporters: ['text', 'lcov', 'html'],
};
```

### Running Coverage

```bash
# Run tests with coverage
npm test -- --coverage

# Output:
# -------------------------|---------|----------|---------|---------|
# File                     | % Stmts | % Branch | % Funcs | % Lines |
# -------------------------|---------|----------|---------|---------|
# All files               |   85.5  |   78.2   |   90.1  |   85.3  |
#  src/                   |   88.9  |   82.1   |   92.3  |   88.7  |
#   Calculator.ts         |   100   |   100    |   100   |   100   |
#   UserService.ts        |   75.5  |   66.7   |   80.0  |   75.5  |
#   OrderProcessor.ts     |   90.2  |   85.3   |   95.1  |   90.0  |
# -------------------------|---------|----------|---------|---------|

# View HTML report
open coverage/lcov-report/index.html
```

### Coverage in CI/CD

```yaml
# .github/workflows/test.yml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2
      - run: npm ci
      - run: npm test -- --coverage
      
      # Fail if coverage below threshold
      - name: Check coverage
        run: |
          npm test -- --coverage --coverageThreshold='{"global":{"branches":80,"functions":80,"lines":80,"statements":80}}'
      
      # Upload coverage to Codecov
      - name: Upload coverage
        uses: codecov/codecov-action@v2
        with:
          files: ./coverage/lcov.info
```

## The 100% Coverage Myth

### Why 100% Coverage is Often Wasteful

```typescript
// Trivial getter - testing adds no value
export class User {
  constructor(private name: string) {}
  
  getName(): string {
    return this.name; // Testing this line is wasteful
  }
}

// Test for 100% coverage
describe('User', () => {
  it('should return name', () => {
    const user = new User('John');
    expect(user.getName()).toBe('John'); // No value added
  });
});

// ✅ Better: Don't test trivial code
// Focus on complex business logic instead
```

### Untestable Code That Shouldn't Count

```typescript
// Code that's hard to test often indicates design issues
export class OrderService {
  processOrder(order: Order): void {
    // Direct dependencies - hard to test
    const db = new Database();
    const emailer = new EmailService();
    const payment = new PaymentGateway();
    
    db.save(order);
    payment.charge(order.total);
    emailer.send(order.email, 'Order confirmed');
    
    // Reaching 100% coverage requires mocking all these dependencies
    // Better: Refactor to inject dependencies
  }
}

// ✅ Better design - testable without 100% coverage obsession
export class OrderService {
  constructor(
    private db: Database,
    private emailer: EmailService,
    private payment: PaymentGateway
  ) {}
  
  processOrder(order: Order): void {
    this.db.save(order);
    this.payment.charge(order.total);
    this.emailer.send(order.email, 'Order confirmed');
  }
}
```

### False Sense of Security

```typescript
// ❌ BAD: 100% coverage but useless tests
export function divide(a: number, b: number): number {
  if (b === 0) {
    throw new Error('Division by zero');
  }
  return a / b;
}

describe('divide', () => {
  it('should divide numbers', () => {
    divide(10, 2); // No assertion!
  });

  it('should handle zero', () => {
    try {
      divide(10, 0);
    } catch (e) {
      // Caught but no assertion
    }
  });
});

// 100% coverage ✅ but tests don't verify anything!

// ✅ GOOD: Lower coverage but meaningful tests
describe('divide', () => {
  it('should divide numbers correctly', () => {
    expect(divide(10, 2)).toBe(5);
    expect(divide(9, 3)).toBe(3);
  });

  it('should throw on division by zero', () => {
    expect(() => divide(10, 0)).toThrow('Division by zero');
  });
});
```

## Meaningful Coverage

### Focus on Critical Paths

```typescript
// Payment processing - critical, needs high coverage
export class PaymentProcessor {
  async processPayment(amount: number, card: CreditCard): Promise<PaymentResult> {
    // Validate card
    if (!this.validateCard(card)) {
      throw new Error('Invalid card');
    }
    
    // Check fraud
    if (await this.detectFraud(card, amount)) {
      throw new Error('Fraudulent transaction');
    }
    
    // Process payment
    try {
      const result = await this.gateway.charge(card, amount);
      await this.db.recordTransaction(result);
      return result;
    } catch (error) {
      await this.db.recordFailure(error);
      throw error;
    }
  }
}

// ✅ High coverage warranted - critical business logic
describe('PaymentProcessor', () => {
  // Test all paths - this is critical code
  it('should process valid payment', () => {});
  it('should reject invalid card', () => {});
  it('should detect fraud', () => {});
  it('should handle gateway errors', () => {});
  it('should record transactions', () => {});
  it('should record failures', () => {});
});

// Utility function - less critical
export function formatCurrency(amount: number): string {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD'
  }).format(amount);
}

// ✅ Lower coverage acceptable - not critical
describe('formatCurrency', () => {
  it('should format basic amounts', () => {
    expect(formatCurrency(10)).toBe('$10.00');
  });
  // Don't need to test every edge case
});
```

### Coverage by Risk

```typescript
/*
HIGH RISK - Aim for 90%+ coverage:
- Payment processing
- Authentication/authorization
- Data validation
- Security-critical code
- Financial calculations

MEDIUM RISK - Aim for 70-80% coverage:
- Business logic
- API endpoints
- State management
- Complex algorithms

LOW RISK - Aim for 50-60% coverage:
- UI components
- Utility functions
- Formatters
- Simple getters/setters

MINIMAL RISK - Coverage optional:
- Type definitions
- Constants
- Configuration
- Trivial functions
*/
```

## Coverage Anti-Patterns

### 1. Testing Implementation Details

```typescript
// ❌ BAD: Testing private methods for coverage
export class UserService {
  async createUser(data: CreateUserDTO): Promise<User> {
    const validated = this.validateData(data);
    const hashed = await this.hashPassword(validated.password);
    return this.repository.save({ ...validated, password: hashed });
  }

  private validateData(data: CreateUserDTO): ValidatedUserDTO {
    // Validation logic
    return validated;
  }

  private async hashPassword(password: string): Promise<string> {
    // Hashing logic
    return hashed;
  }
}

describe('UserService', () => {
  // Testing private methods directly
  it('should validate data', () => {
    expect((service as any).validateData(data)).toBeDefined();
  });

  it('should hash password', async () => {
    const hash = await (service as any).hashPassword('password');
    expect(hash).toBeTruthy();
  });
});

// ✅ GOOD: Test public API, private methods covered implicitly
describe('UserService', () => {
  it('should create user with valid data', async () => {
    const user = await service.createUser({
      email: 'test@example.com',
      password: 'password123'
    });
    
    expect(user.email).toBe('test@example.com');
    expect(user.password).not.toBe('password123'); // Hashed
  });

  it('should reject invalid data', async () => {
    await expect(service.createUser({ email: 'invalid' }))
      .rejects.toThrow();
  });
});
// validateData and hashPassword covered through public API
```

### 2. Pointless Tests for Coverage

```typescript
// ❌ BAD: Testing framework features
describe('React useState', () => {
  it('should update state', () => {
    const [value, setValue] = useState(0);
    act(() => setValue(1));
    expect(value).toBe(1);
  });
});
// Testing React, not your code!

// ❌ BAD: Testing obvious code
export const ADD = 'ADD';
export const REMOVE = 'REMOVE';

describe('Action types', () => {
  it('should have ADD constant', () => {
    expect(ADD).toBe('ADD');
  });
});
// Completely pointless

// ✅ GOOD: Test your logic
describe('ShoppingCart', () => {
  it('should add items', () => {
    const cart = new ShoppingCart();
    cart.dispatch({ type: 'ADD', item: product });
    expect(cart.getItems()).toContain(product);
  });
});
```

### 3. Coverage Without Assertions

```typescript
// ❌ BAD: Code runs but nothing verified
describe('OrderProcessor', () => {
  it('should process order', () => {
    processor.processOrder(order);
    // No assertions - code runs but we don't know if it worked
  });
});

// ✅ GOOD: Verify behavior
describe('OrderProcessor', () => {
  it('should save order to database', async () => {
    await processor.processOrder(order);
    const saved = await db.findOrder(order.id);
    expect(saved).toEqual(order);
  });

  it('should send confirmation email', async () => {
    await processor.processOrder(order);
    expect(emailService.send).toHaveBeenCalledWith(
      order.email,
      'Order Confirmed'
    );
  });
});
```

## Excluding Code from Coverage

### Ignore Comments

```typescript
// Exclude from coverage with comments
export function experimentalFeature(): void {
  /* istanbul ignore next */
  if (process.env.FEATURE_FLAG === 'true') {
    // Experimental code not yet tested
    experimentalLogic();
  }
}

// Ignore entire function
/* istanbul ignore next */
export function debugHelper(): void {
  console.log('Debug info');
  // Development-only code
}

// Ignore else branch
if (condition) {
  // Main logic
} /* istanbul ignore next */ else {
  // Defensive code that never happens in practice
}
```

### Configuration Exclusions

```javascript
// jest.config.js
module.exports = {
  collectCoverageFrom: [
    'src/**/*.{js,ts,jsx,tsx}',
    '!src/**/*.d.ts',              // Type definitions
    '!src/**/*.stories.tsx',       // Storybook stories
    '!src/**/__tests__/**',        // Test files
    '!src/**/index.ts',            // Barrel exports
    '!src/setupTests.ts',          // Test setup
    '!src/reportWebVitals.ts',     // CRA boilerplate
  ],
};
```

## React Component Coverage

```tsx
// Component
export const TodoList: React.FC<TodoListProps> = ({ todos, onToggle, onDelete }) => {
  return (
    <ul className="todo-list">
      {todos.length === 0 ? (
        <li className="empty">No todos yet</li>
      ) : (
        todos.map(todo => (
          <li key={todo.id} className={todo.completed ? 'completed' : ''}>
            <input
              type="checkbox"
              checked={todo.completed}
              onChange={() => onToggle(todo.id)}
            />
            <span>{todo.text}</span>
            <button onClick={() => onDelete(todo.id)}>Delete</button>
          </li>
        ))
      )}
    </ul>
  );
};

// ✅ Good coverage approach
describe('TodoList', () => {
  it('should show empty state', () => {
    render(<TodoList todos={[]} onToggle={jest.fn()} onDelete={jest.fn()} />);
    expect(screen.getByText('No todos yet')).toBeInTheDocument();
  });

  it('should render todo items', () => {
    const todos = [
      { id: '1', text: 'Task 1', completed: false },
      { id: '2', text: 'Task 2', completed: true }
    ];
    render(<TodoList todos={todos} onToggle={jest.fn()} onDelete={jest.fn()} />);
    
    expect(screen.getByText('Task 1')).toBeInTheDocument();
    expect(screen.getByText('Task 2')).toBeInTheDocument();
  });

  it('should call onToggle when checkbox clicked', () => {
    const onToggle = jest.fn();
    const todos = [{ id: '1', text: 'Task 1', completed: false }];
    
    render(<TodoList todos={todos} onToggle={onToggle} onDelete={jest.fn()} />);
    fireEvent.click(screen.getByRole('checkbox'));
    
    expect(onToggle).toHaveBeenCalledWith('1');
  });

  it('should call onDelete when delete button clicked', () => {
    const onDelete = jest.fn();
    const todos = [{ id: '1', text: 'Task 1', completed: false }];
    
    render(<TodoList todos={todos} onToggle={jest.fn()} onDelete={onDelete} />);
    fireEvent.click(screen.getByText('Delete'));
    
    expect(onDelete).toHaveBeenCalledWith('1');
  });
});

// Coverage: ~95% - all meaningful branches tested
// Missing: Edge cases like empty text, special characters - acceptable
```

## Coverage Reports in Practice

### HTML Coverage Report

```
coverage/lcov-report/index.html

┌─────────────────────────────────────────────────┐
│ File                  │ Stmts │ Branch │ Funcs  │
├─────────────────────────────────────────────────┤
│ UserService.ts        │ 85.5% │ 78.2%  │ 90.1% │
│   ↳ Line 45-47       │ 🔴 Not covered           │
│   ↳ Line 89          │ 🟡 Partially covered     │
│   ↳ Line 120-150     │ 🟢 Fully covered         │
└─────────────────────────────────────────────────┘

Click line numbers to see which tests cover that line
```

### Coverage Badges

```markdown
<!-- README.md -->
![Coverage](https://img.shields.io/codecov/c/github/username/repo)

# Project Name

[![Coverage: 85%](https://img.shields.io/badge/coverage-85%25-green)]()
```

## Common Mistakes

### 1. Chasing 100% Coverage

```typescript
// ❌ BAD: Writing pointless tests for 100%
describe('constants', () => {
  it('should have correct values', () => {
    expect(API_URL).toBe('https://api.example.com');
    expect(MAX_RETRIES).toBe(3);
    expect(TIMEOUT).toBe(5000);
  });
});

// ✅ GOOD: Focus on behavior, not arbitrary percentage
```

### 2. Ignoring Uncovered Critical Code

```typescript
// Coverage report shows payment validation uncovered
// ❌ BAD: Ignoring it because overall coverage is "good enough"

// ✅ GOOD: Prioritize critical code regardless of overall percentage
describe('PaymentValidator', () => {
  // Add tests for uncovered critical paths
});
```

### 3. Coverage Without Quality

```typescript
// ❌ BAD: Tests that increase coverage but don't verify behavior
it('should process', () => {
  processOrder(order); // No assertions
});

// ✅ GOOD: Tests that verify correct behavior
it('should save order and send confirmation', async () => {
  await processOrder(order);
  expect(await db.findOrder(order.id)).toBeDefined();
  expect(emailService.send).toHaveBeenCalled();
});
```

## Best Practices

1. **Coverage is a Tool, Not a Goal**: Use it to find untested code, not as a target
2. **Focus on Critical Code**: High coverage for important paths, less for trivial code
3. **Quality Over Quantity**: Good assertions matter more than high percentages
4. **Set Reasonable Thresholds**: 70-80% is often sufficient, not 100%
5. **Review Coverage Reports**: Understand what's uncovered and why
6. **Don't Test Framework Code**: Test your logic, not libraries
7. **Exclude Appropriate Code**: Config, types, generated code
8. **Track Trends**: Decreasing coverage might indicate issues
9. **Cover Edge Cases**: Not just happy paths
10. **Use Coverage to Guide, Not Drive**: Let design drive tests, not coverage

## When to Care About Coverage

**High coverage important for:**
- Payment and financial logic
- Authentication and security
- Data validation and sanitization
- Critical business rules
- Public APIs and libraries

**Lower coverage acceptable for:**
- UI components (focus on integration)
- Configuration files
- Type definitions
- Utility functions
- Experimental features

## Interview Questions

1. **What does 100% code coverage guarantee?**
   - Nothing. It means all code was executed, not that it's tested correctly or that all scenarios are covered.

2. **What's the difference between line and branch coverage?**
   - Line coverage: % of lines executed. Branch coverage: % of decision branches (if/else) taken.

3. **Why is 100% coverage often wasteful?**
   - Testing trivial code, framework features, and generated code adds no value and wastes time.

4. **How do you determine appropriate coverage targets?**
   - Based on code criticality and risk. High for critical paths, lower for UI and utilities.

5. **What's more important than high coverage?**
   - Test quality, meaningful assertions, testing critical paths, and good design.

## Key Takeaways

1. Coverage measures code execution, not test quality
2. 100% coverage is rarely worth pursuing
3. Focus on critical and complex code
4. Branch coverage more meaningful than line coverage
5. Coverage helps identify untested code
6. Don't write tests just for coverage
7. Set reasonable thresholds (70-80%)
8. Exclude non-testable code appropriately
9. Use coverage reports to guide testing
10. Quality of tests matters more than coverage percentage

## Resources

- **Jest Coverage**: https://jestjs.io/docs/configuration#collectcoverage-boolean
- **Istanbul Coverage**: https://istanbul.js.org/
- **Codecov**: https://about.codecov.io/
- **Martin Fowler - Test Coverage**: https://martinfowler.com/bliki/TestCoverage.html
- **Google Testing Blog**: https://testing.googleblog.com/
- **"Growing Object-Oriented Software, Guided by Tests"** by Steve Freeman
