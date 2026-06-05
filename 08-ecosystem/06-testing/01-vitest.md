# Vitest - Modern Testing Framework

## The Idea

**In plain English:** Vitest is a tool that automatically checks your code for you — you write small scripts that say "if I do X, I expect Y to happen," and Vitest runs all those checks instantly to tell you if anything is broken. A "test" is just a mini-program that verifies your real code behaves correctly.

**Real-world analogy:** Imagine a car factory that uses a checklist to inspect every car before it leaves the building. A worker tests the headlights, checks the brakes, and honks the horn — each check confirms one specific thing works. If a new change accidentally breaks the brakes, the checklist catches it before the car ships.

- The checklist = your test suite (the collection of all tests)
- Each individual inspection step = a single test (e.g., "expect button to show text")
- The worker running the checklist = Vitest (the tool that executes all your tests automatically)

---

## Overview

Vitest is a blazing-fast unit testing framework powered by Vite. It provides a Jest-compatible API with native ESM support, TypeScript support out of the box, and exceptional performance through smart caching and parallel execution. Vitest is the natural choice for Vite projects but works with any modern JavaScript/TypeScript application.

## Installation and Setup

```bash
# Install Vitest
npm install -D vitest

# Install with testing library
npm install -D vitest @testing-library/react @testing-library/jest-dom

# Install with coverage
npm install -D vitest @vitest/coverage-v8

# Install UI for browser-based test viewing
npm install -D vitest @vitest/ui
```

### Basic Configuration

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: './src/test/setup.ts',
    css: true,
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      exclude: ['node_modules/', 'src/test/'],
    },
  },
});
```

```typescript
// src/test/setup.ts
import { expect, afterEach } from 'vitest';
import { cleanup } from '@testing-library/react';
import * as matchers from '@testing-library/jest-dom/matchers';

expect.extend(matchers);

afterEach(() => {
  cleanup();
});
```

## Core Testing Features

### Basic Test Structure

```typescript
import { describe, it, expect, test } from 'vitest';

// Basic test
test('adds 1 + 2 to equal 3', () => {
  expect(1 + 2).toBe(3);
});

// Using describe for grouping
describe('Math operations', () => {
  it('should add numbers correctly', () => {
    expect(2 + 2).toBe(4);
  });

  it('should subtract numbers correctly', () => {
    expect(5 - 3).toBe(2);
  });

  it('should multiply numbers correctly', () => {
    expect(3 * 4).toBe(12);
  });
});

// Nested describe blocks
describe('Array operations', () => {
  describe('push', () => {
    it('should add element to end', () => {
      const arr = [1, 2, 3];
      arr.push(4);
      expect(arr).toEqual([1, 2, 3, 4]);
    });
  });

  describe('pop', () => {
    it('should remove last element', () => {
      const arr = [1, 2, 3];
      const last = arr.pop();
      expect(last).toBe(3);
      expect(arr).toEqual([1, 2]);
    });
  });
});

// Test.each for parameterized tests
test.each([
  [1, 1, 2],
  [1, 2, 3],
  [2, 2, 4],
])('adds %i + %i to equal %i', (a, b, expected) => {
  expect(a + b).toBe(expected);
});

// Describe.each for multiple test suites
describe.each([
  { a: 1, b: 1, expected: 2 },
  { a: 1, b: 2, expected: 3 },
  { a: 2, b: 1, expected: 3 },
])('add($a, $b)', ({ a, b, expected }) => {
  it(`returns ${expected}`, () => {
    expect(a + b).toBe(expected);
  });
});
```

### Assertions and Matchers

```typescript
import { expect } from 'vitest';

// Equality matchers
expect(2 + 2).toBe(4); // Strict equality (===)
expect({ name: 'John' }).toEqual({ name: 'John' }); // Deep equality
expect({ name: 'John' }).toStrictEqual({ name: 'John' }); // Strict deep equality

// Truthiness
expect(true).toBeTruthy();
expect(false).toBeFalsy();
expect(null).toBeNull();
expect(undefined).toBeUndefined();
expect('hello').toBeDefined();

// Numbers
expect(10).toBeGreaterThan(5);
expect(10).toBeGreaterThanOrEqual(10);
expect(5).toBeLessThan(10);
expect(5).toBeLessThanOrEqual(5);
expect(0.1 + 0.2).toBeCloseTo(0.3); // Floating point

// Strings
expect('Hello World').toMatch(/World/);
expect('Hello').toContain('ell');
expect('hello').toHaveLength(5);

// Arrays and Iterables
expect([1, 2, 3]).toContain(2);
expect([1, 2, 3]).toContainEqual(2);
expect([1, 2, 3]).toHaveLength(3);
expect(['apple', 'banana']).toEqual(expect.arrayContaining(['banana']));

// Objects
expect({ name: 'John', age: 30 }).toHaveProperty('name');
expect({ name: 'John', age: 30 }).toHaveProperty('age', 30);
expect({ a: 1, b: 2 }).toMatchObject({ a: 1 });

// Exceptions
expect(() => {
  throw new Error('error');
}).toThrow();
expect(() => {
  throw new Error('Invalid input');
}).toThrow('Invalid input');
expect(() => {
  throw new Error('Not found');
}).toThrow(/not found/i);

// Promises
await expect(Promise.resolve('success')).resolves.toBe('success');
await expect(Promise.reject(new Error('fail'))).rejects.toThrow('fail');

// Async matchers
await expect(fetchUser()).resolves.toMatchObject({ name: 'John' });
await expect(failingRequest()).rejects.toThrow('Network error');

// Negation
expect(2 + 2).not.toBe(5);
expect('hello').not.toMatch(/world/);

// Snapshots
expect(component).toMatchSnapshot();
expect({ data: [1, 2, 3] }).toMatchInlineSnapshot();
```

### Setup and Teardown

```typescript
import { beforeAll, beforeEach, afterAll, afterEach, describe, it } from 'vitest';

describe('Database operations', () => {
  // Runs once before all tests in this block
  beforeAll(async () => {
    await database.connect();
  });

  // Runs before each test in this block
  beforeEach(async () => {
    await database.clear();
  });

  // Runs after each test in this block
  afterEach(async () => {
    await database.cleanup();
  });

  // Runs once after all tests in this block
  afterAll(async () => {
    await database.disconnect();
  });

  it('should insert user', async () => {
    const user = await database.insert({ name: 'John' });
    expect(user.id).toBeDefined();
  });

  it('should find user by id', async () => {
    const user = await database.insert({ name: 'John' });
    const found = await database.findById(user.id);
    expect(found).toEqual(user);
  });
});

// Scoped setup - only affects nested tests
describe('outer', () => {
  beforeEach(() => {
    console.log('outer beforeEach');
  });

  describe('inner', () => {
    beforeEach(() => {
      console.log('inner beforeEach');
    });

    it('test', () => {
      // Both beforeEach hooks run
      expect(true).toBe(true);
    });
  });
});
```

## Mocking

### Function Mocks

```typescript
import { vi, describe, it, expect, beforeEach } from 'vitest';

// Create mock function
const mockFn = vi.fn();

describe('Mock functions', () => {
  beforeEach(() => {
    mockFn.mockClear();
  });

  it('tracks calls', () => {
    mockFn('hello', 123);
    mockFn('world', 456);

    expect(mockFn).toHaveBeenCalledTimes(2);
    expect(mockFn).toHaveBeenCalledWith('hello', 123);
    expect(mockFn).toHaveBeenLastCalledWith('world', 456);
  });

  it('returns values', () => {
    mockFn.mockReturnValue('mocked');
    expect(mockFn()).toBe('mocked');

    mockFn.mockReturnValueOnce('first').mockReturnValueOnce('second');
    expect(mockFn()).toBe('first');
    expect(mockFn()).toBe('second');
  });

  it('resolves promises', async () => {
    mockFn.mockResolvedValue('success');
    await expect(mockFn()).resolves.toBe('success');

    mockFn.mockRejectedValue(new Error('failed'));
    await expect(mockFn()).rejects.toThrow('failed');
  });

  it('implements custom logic', () => {
    mockFn.mockImplementation((a, b) => a + b);
    expect(mockFn(2, 3)).toBe(5);

    mockFn.mockImplementationOnce(() => 'once');
    expect(mockFn()).toBe('once');
    expect(mockFn(1, 2)).toBe(3); // Falls back to original implementation
  });
});

// Spy on existing object methods
const calculator = {
  add: (a: number, b: number) => a + b,
  subtract: (a: number, b: number) => a - b,
};

describe('Spies', () => {
  it('spies on method', () => {
    const spy = vi.spyOn(calculator, 'add');
    
    calculator.add(2, 3);
    
    expect(spy).toHaveBeenCalledWith(2, 3);
    expect(spy).toHaveReturnedWith(5);
    
    spy.mockRestore(); // Restore original implementation
  });

  it('mocks implementation', () => {
    const spy = vi.spyOn(calculator, 'add').mockReturnValue(100);
    
    expect(calculator.add(2, 3)).toBe(100);
    
    spy.mockRestore();
    expect(calculator.add(2, 3)).toBe(5); // Original behavior restored
  });
});
```

### Module Mocks

```typescript
import { vi, describe, it, expect, beforeEach } from 'vitest';

// Mock entire module
vi.mock('./api', () => ({
  fetchUser: vi.fn(),
  createUser: vi.fn(),
}));

import { fetchUser, createUser } from './api';

describe('API mocks', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('mocks fetchUser', async () => {
    (fetchUser as any).mockResolvedValue({ id: 1, name: 'John' });

    const user = await fetchUser(1);

    expect(fetchUser).toHaveBeenCalledWith(1);
    expect(user).toEqual({ id: 1, name: 'John' });
  });

  it('mocks createUser', async () => {
    const newUser = { name: 'Jane', email: 'jane@example.com' };
    (createUser as any).mockResolvedValue({ id: 2, ...newUser });

    const user = await createUser(newUser);

    expect(createUser).toHaveBeenCalledWith(newUser);
    expect(user.id).toBeDefined();
  });
});

// Partial module mock
vi.mock('./utils', async () => {
  const actual = await vi.importActual('./utils');
  return {
    ...actual,
    generateId: vi.fn(() => 'mocked-id'),
  };
});

// Auto-mock with factory
vi.mock('./database', () => {
  return {
    default: {
      connect: vi.fn(),
      query: vi.fn(),
      disconnect: vi.fn(),
    },
  };
});

// Mock with implementation
vi.mock('axios', () => ({
  default: {
    get: vi.fn(),
    post: vi.fn(),
    put: vi.fn(),
    delete: vi.fn(),
  },
}));

import axios from 'axios';

describe('HTTP requests', () => {
  it('makes GET request', async () => {
    const mockData = { id: 1, name: 'Test' };
    (axios.get as any).mockResolvedValue({ data: mockData });

    const response = await axios.get('/api/users/1');

    expect(axios.get).toHaveBeenCalledWith('/api/users/1');
    expect(response.data).toEqual(mockData);
  });
});
```

### Timers and Dates

```typescript
import { vi, describe, it, expect, beforeEach, afterEach } from 'vitest';

describe('Timer mocks', () => {
  beforeEach(() => {
    vi.useFakeTimers();
  });

  afterEach(() => {
    vi.restoreAllMocks();
  });

  it('advances timers', () => {
    const callback = vi.fn();
    setTimeout(callback, 1000);

    expect(callback).not.toHaveBeenCalled();

    vi.advanceTimersByTime(1000);

    expect(callback).toHaveBeenCalledOnce();
  });

  it('runs all timers', () => {
    const callback1 = vi.fn();
    const callback2 = vi.fn();

    setTimeout(callback1, 1000);
    setTimeout(callback2, 2000);

    vi.runAllTimers();

    expect(callback1).toHaveBeenCalled();
    expect(callback2).toHaveBeenCalled();
  });

  it('mocks Date', () => {
    const mockDate = new Date('2024-01-01');
    vi.setSystemTime(mockDate);

    expect(new Date().getFullYear()).toBe(2024);
    expect(new Date().getMonth()).toBe(0); // January

    vi.useRealTimers();
  });

  it('advances time incrementally', async () => {
    const callback = vi.fn();
    
    const intervalId = setInterval(() => {
      callback();
    }, 100);

    await vi.advanceTimersByTimeAsync(350);

    expect(callback).toHaveBeenCalledTimes(3);

    clearInterval(intervalId);
  });
});
```

## React Component Testing

```typescript
import { describe, it, expect } from 'vitest';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Button } from './Button';
import { LoginForm } from './LoginForm';

describe('Button component', () => {
  it('renders with text', () => {
    render(<Button>Click me</Button>);
    expect(screen.getByText('Click me')).toBeInTheDocument();
  });

  it('handles click events', async () => {
    const handleClick = vi.fn();
    render(<Button onClick={handleClick}>Click me</Button>);

    await userEvent.click(screen.getByText('Click me'));

    expect(handleClick).toHaveBeenCalledOnce();
  });

  it('disables button', () => {
    render(<Button disabled>Click me</Button>);
    expect(screen.getByText('Click me')).toBeDisabled();
  });
});

describe('LoginForm component', () => {
  it('submits form with credentials', async () => {
    const handleSubmit = vi.fn();
    render(<LoginForm onSubmit={handleSubmit} />);

    const user = userEvent.setup();

    await user.type(screen.getByLabelText(/email/i), 'test@example.com');
    await user.type(screen.getByLabelText(/password/i), 'password123');
    await user.click(screen.getByRole('button', { name: /login/i }));

    expect(handleSubmit).toHaveBeenCalledWith({
      email: 'test@example.com',
      password: 'password123',
    });
  });

  it('shows validation errors', async () => {
    render(<LoginForm onSubmit={vi.fn()} />);

    await userEvent.click(screen.getByRole('button', { name: /login/i }));

    expect(screen.getByText(/email is required/i)).toBeInTheDocument();
    expect(screen.getByText(/password is required/i)).toBeInTheDocument();
  });

  it('handles async submission', async () => {
    const handleSubmit = vi.fn().mockRejectedValue(new Error('Invalid credentials'));
    render(<LoginForm onSubmit={handleSubmit} />);

    await userEvent.type(screen.getByLabelText(/email/i), 'test@example.com');
    await userEvent.type(screen.getByLabelText(/password/i), 'wrong');
    await userEvent.click(screen.getByRole('button', { name: /login/i }));

    await waitFor(() => {
      expect(screen.getByText(/invalid credentials/i)).toBeInTheDocument();
    });
  });
});
```

## Snapshot Testing

```typescript
import { describe, it, expect } from 'vitest';
import { render } from '@testing-library/react';
import { UserCard } from './UserCard';

describe('UserCard snapshots', () => {
  it('matches snapshot', () => {
    const { container } = render(
      <UserCard 
        name="John Doe" 
        email="john@example.com"
        role="admin"
      />
    );

    expect(container).toMatchSnapshot();
  });

  it('matches inline snapshot', () => {
    const data = {
      id: 1,
      name: 'John Doe',
      email: 'john@example.com',
    };

    expect(data).toMatchInlineSnapshot(`
      {
        "email": "john@example.com",
        "id": 1,
        "name": "John Doe",
      }
    `);
  });

  it('updates snapshot', () => {
    const { container } = render(<UserCard name="Jane" email="jane@test.com" />);
    
    // Run with --update or -u flag to update snapshot
    expect(container).toMatchSnapshot();
  });
});
```

## Coverage Reports

```bash
# Run tests with coverage
npm run vitest -- --coverage

# Run coverage with specific reporter
npm run vitest -- --coverage --coverage.reporter=html

# Coverage with threshold
npm run vitest -- --coverage --coverage.lines=80
```

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    coverage: {
      provider: 'v8', // or 'istanbul'
      reporter: ['text', 'json', 'html', 'lcov'],
      include: ['src/**/*.{js,jsx,ts,tsx}'],
      exclude: [
        'node_modules/',
        'src/**/*.test.{js,jsx,ts,tsx}',
        'src/**/*.spec.{js,jsx,ts,tsx}',
        'src/test/',
      ],
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 80,
        statements: 80,
      },
    },
  },
});
```

## Watch Mode and UI

```bash
# Run in watch mode
npm run vitest

# Run with UI
npm run vitest -- --ui

# Run specific test file
npm run vitest src/components/Button.test.tsx

# Run tests matching pattern
npm run vitest -- --grep="Button"

# Run only changed tests
npm run vitest -- --changed
```

## Workspace and Monorepo Support

```typescript
// vitest.workspace.ts
import { defineWorkspace } from 'vitest/config';

export default defineWorkspace([
  {
    test: {
      name: 'unit',
      include: ['src/**/*.test.ts'],
      environment: 'node',
    },
  },
  {
    test: {
      name: 'integration',
      include: ['tests/**/*.test.ts'],
      environment: 'node',
    },
  },
  {
    test: {
      name: 'browser',
      include: ['src/**/*.browser.test.ts'],
      environment: 'jsdom',
    },
  },
]);
```

## Common Mistakes

1. **Not clearing mocks**: Forgetting to clear mocks between tests causing test pollution.

2. **Missing async/await**: Not awaiting async operations in tests.

3. **Wrong environment**: Using 'node' environment for DOM tests or vice versa.

4. **Not restoring timers**: Forgetting to restore fake timers after tests.

5. **Improper cleanup**: Not cleaning up side effects between tests.

6. **Over-mocking**: Mocking too much, testing implementation instead of behavior.

7. **Missing coverage setup**: Not configuring coverage properly.

8. **Snapshot abuse**: Over-relying on snapshots instead of explicit assertions.

9. **Not using userEvent**: Using fireEvent instead of @testing-library/user-event.

10. **Global state pollution**: Tests affecting each other through shared global state.

## Best Practices

1. **Use globals config**: Enable globals in config to avoid importing test functions.

2. **Organize with describe blocks**: Group related tests for better organization.

3. **Clear setup/teardown**: Use appropriate hooks for test preparation and cleanup.

4. **Mock external dependencies**: Mock API calls, databases, and external services.

5. **Test behavior, not implementation**: Focus on what components do, not how.

6. **Use userEvent over fireEvent**: Simulate real user interactions.

7. **Leverage snapshots wisely**: Use for stable UI components, not business logic.

8. **Configure coverage thresholds**: Enforce minimum coverage standards.

9. **Use test.each for similar tests**: Reduce duplication with parameterized tests.

10. **Enable watch mode during development**: Get instant feedback on code changes.

## When to Use Vitest

**Use Vitest when:**

- Building Vite-based applications
- Want faster test execution
- Need native ESM support
- Want Jest-compatible API
- Building modern TypeScript projects
- Need parallel test execution
- Want excellent developer experience

**Consider alternatives when:**

- Using Create React App (use Jest)
- Need maximum ecosystem maturity (use Jest)
- Working with legacy codebases (use Jest)
- Need extensive third-party plugins (use Jest)

## Interview Questions

### Q1: What makes Vitest faster than Jest?

**Answer**: Vitest achieves superior performance through: 1) Leveraging Vite's transformation pipeline and caching, 2) Native ESM support eliminating transformation overhead, 3) Smart watch mode that only runs affected tests, 4) Parallel test execution by default, 5) Faster module resolution, 6) Using esbuild for TypeScript transformation. In practice, Vitest can be 2-10x faster than Jest.

### Q2: How do you mock modules in Vitest?

**Answer**: Use `vi.mock()` at the top of the file:

```typescript
vi.mock('./api', () => ({
  fetchUser: vi.fn(),
  createUser: vi.fn(),
}));

// Then use mocked functions
(fetchUser as any).mockResolvedValue({ id: 1 });
```

For partial mocks, use `vi.importActual()` to preserve real implementations.

### Q3: What's the difference between `vi.fn()`, `vi.spyOn()`, and `vi.mock()`?

**Answer**: `vi.fn()` creates a new mock function from scratch. `vi.spyOn()` wraps an existing object method to track calls while preserving original behavior (unless mocked). `vi.mock()` replaces an entire module with mocks. Use `fn()` for callbacks, `spyOn()` for existing methods you want to observe, and `mock()` for external dependencies.

### Q4: How do you test async operations in Vitest?

**Answer**: Mark test functions as async and await promises:

```typescript
it('fetches data', async () => {
  const data = await fetchUser(1);
  expect(data).toEqual({ id: 1, name: 'John' });
});

// Or use resolves/rejects matchers
await expect(fetchUser(1)).resolves.toMatchObject({ id: 1 });
await expect(failingRequest()).rejects.toThrow('Error');
```

### Q5: How do you configure different test environments in Vitest?

**Answer**: Use workspace configuration:

```typescript
export default defineWorkspace([
  {
    test: {
      name: 'node',
      environment: 'node',
      include: ['src/**/*.test.ts'],
    },
  },
  {
    test: {
      name: 'browser',
      environment: 'jsdom',
      include: ['src/**/*.browser.test.tsx'],
    },
  },
]);
```

Or set per-file with comment: `// @vitest-environment jsdom`

### Q6: How do you handle timer-dependent code in tests?

**Answer**: Use fake timers:

```typescript
beforeEach(() => {
  vi.useFakeTimers();
});

afterEach(() => {
  vi.restoreAllMocks();
});

it('delays execution', () => {
  const callback = vi.fn();
  setTimeout(callback, 1000);
  
  vi.advanceTimersByTime(1000);
  
  expect(callback).toHaveBeenCalled();
});
```

### Q7: What's the benefit of Vitest's UI mode?

**Answer**: UI mode provides a browser-based interface for viewing and interacting with tests. Benefits include: 1) Visual test runner with results, 2) Test filtering and search, 3) Code coverage visualization, 4) Test re-running on file changes, 5) Better debugging with call stacks, 6) Module graph visualization. Access via `vitest --ui`.

## Key Takeaways

1. Vitest is a modern, fast testing framework built on Vite with Jest-compatible API and superior performance.

2. Configuration uses `vitest.config.ts` with test-specific options including environment, setupFiles, and coverage settings.

3. Mocking uses `vi.fn()` for mock functions, `vi.spyOn()` for spying on methods, and `vi.mock()` for module mocks.

4. Timer mocking with `vi.useFakeTimers()` enables testing time-dependent code without actual delays.

5. React testing integrates with Testing Library using jsdom environment for DOM operations.

6. Coverage reports use v8 or istanbul provider with configurable reporters and thresholds.

7. Watch mode provides instant feedback during development, running only affected tests.

8. Workspace configuration enables multiple test environments and configurations in monorepos.

9. Snapshot testing captures component output for regression testing but should be used judiciously.

10. Native ESM support and Vite integration provide significantly faster test execution compared to Jest.

## Resources

- [Official Documentation](https://vitest.dev/)
- [API Reference](https://vitest.dev/api/)
- [GitHub Repository](https://github.com/vitest-dev/vitest)
- [Migration from Jest](https://vitest.dev/guide/migration.html)
- [Testing Library with Vitest](https://testing-library.com/docs/react-testing-library/setup)
- [Vitest UI](https://vitest.dev/guide/ui.html)
