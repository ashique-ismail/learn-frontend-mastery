# Jest

## The Idea

**In plain English:** Jest is a tool that automatically checks whether your code does what you expect it to do — you write small scripts that say "when I call this function with these inputs, I expect this output," and Jest runs all of them and tells you which ones passed or failed.

**Real-world analogy:** Imagine a car factory quality inspector who has a checklist of tests for every car that rolls off the assembly line — checking that the brakes stop within a certain distance, the horn makes sound, and the doors lock properly. If any test fails, the car gets flagged before it ships.

- The inspector's checklist = your test file (the list of things to verify)
- Each individual check on the list = a single test case written with `test()` or `it()`
- The car being inspected = the function or component your code exports
- "The brakes must stop within 10 feet" = an assertion like `expect(result).toBe(expectedValue)`

---

## What It Is

Jest is the dominant JavaScript testing framework. It includes a test runner, assertion library, mocking system, code coverage, and snapshot testing out of the box. Vitest has become the preferred choice for Vite-based projects, but Jest remains ubiquitous in Create React App, Angular, and Node.js projects.

---

## Configuration

```js
// jest.config.ts
import type { Config } from 'jest';

const config: Config = {
  preset: 'ts-jest',             // TypeScript support
  testEnvironment: 'jsdom',      // browser-like environment
  setupFilesAfterFramework: ['<rootDir>/src/setupTests.ts'],
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',  // path aliases
    '\\.(css|less|scss)$': 'identity-obj-proxy',  // CSS modules
    '\\.(png|jpg|svg)$': '<rootDir>/__mocks__/fileMock.js',
  },
  coverageDirectory: 'coverage',
  collectCoverageFrom: ['src/**/*.{ts,tsx}', '!src/**/*.d.ts'],
  coverageThreshold: {
    global: { branches: 80, functions: 80, lines: 80, statements: 80 }
  },
};

export default config;
```

---

## Mocking

### Module Mocking

```ts
// Auto-mock entire module
jest.mock('./api');

// Mock with implementation
jest.mock('./api', () => ({
  fetchUser: jest.fn().mockResolvedValue({ id: '1', name: 'Alice' }),
  createUser: jest.fn().mockResolvedValue({ id: '2', name: 'Bob' }),
}));

// In tests
import { fetchUser } from './api';
const mockFetchUser = fetchUser as jest.MockedFunction<typeof fetchUser>;

test('loads user', async () => {
  mockFetchUser.mockResolvedValueOnce({ id: '1', name: 'Alice' });
  const result = await loadUser('1');
  expect(result.name).toBe('Alice');
});
```

### `jest.spyOn`

```ts
// Spy on a method without replacing it
const consoleSpy = jest.spyOn(console, 'error').mockImplementation(() => {});

// Restore after test
afterEach(() => consoleSpy.mockRestore());

// Spy on module method
import * as api from './api';
const spy = jest.spyOn(api, 'fetchUser').mockResolvedValue({ id: '1', name: 'Alice' });
```

---

## Fake Timers

```ts
beforeEach(() => jest.useFakeTimers());
afterEach(() => jest.useRealTimers());

test('debounced function', () => {
  const fn = jest.fn();
  const debounced = debounce(fn, 300);

  debounced();
  debounced();
  debounced();

  expect(fn).not.toHaveBeenCalled();

  jest.advanceTimersByTime(300);

  expect(fn).toHaveBeenCalledTimes(1);
});

// Async timers
test('async with fake timers', async () => {
  const promise = fetchWithRetry(url, { delay: 1000 });
  await jest.runAllTimersAsync(); // advance and flush
  const result = await promise;
  expect(result).toBeDefined();
});
```

---

## Snapshot Testing

```ts
// Creates/updates a .snap file
test('Button renders correctly', () => {
  const { container } = render(<Button variant="primary">Click me</Button>);
  expect(container.firstChild).toMatchSnapshot();
});

// Inline snapshot (stored in the test file)
test('Button renders correctly', () => {
  const { container } = render(<Button>Click</Button>);
  expect(container.firstChild).toMatchInlineSnapshot(`
    <button class="btn btn-primary">
      Click
    </button>
  `);
});
```

Update snapshots: `jest --updateSnapshot` or `jest -u`

---

## Code Coverage

```bash
# Run with coverage
jest --coverage

# Output (with threshold configured):
# Statements   : 87.5%
# Branches     : 78.3%  ← below 80% threshold → FAIL
# Functions    : 92.1%
# Lines        : 87.5%
```

---

## Setup File

```ts
// src/setupTests.ts
import '@testing-library/jest-dom'; // adds .toBeInTheDocument(), etc.

// Global mocks
global.ResizeObserver = jest.fn().mockImplementation(() => ({
  observe: jest.fn(),
  unobserve: jest.fn(),
  disconnect: jest.fn(),
}));

global.IntersectionObserver = jest.fn().mockImplementation(() => ({
  observe: jest.fn(),
  disconnect: jest.fn(),
}));
```

---

## Jest vs Vitest

| | Jest | Vitest |
|---|---|---|
| Speed | Slower (requires transform) | 2-5x faster (native ESM) |
| Config | jest.config + babel/ts-jest | vitest.config (shares Vite config) |
| Mocking | Mature, full-featured | Same API, growing |
| ESM support | Requires configuration | Native |
| Watch mode | Good | Excellent (Vite HMR) |
| Ecosystem | Huge | Growing fast |
| Best for | CRA, Angular, Node.js | Vite-based projects |

---

## Common Interview Questions

**Q: When should you use `jest.fn()` vs `jest.spyOn()`?**
`jest.fn()` creates a standalone mock function. `jest.spyOn()` wraps an existing method — it calls the original by default and you can restore it afterward. Use `spyOn` when you want to track calls to an existing method without fully replacing it.

**Q: What's the difference between `mockReturnValue` and `mockResolvedValue`?**
`mockReturnValue(x)` makes the mock return `x` synchronously. `mockResolvedValue(x)` makes the mock return `Promise.resolve(x)` — for async functions. Use `mockRejectedValue(error)` to simulate rejection.

**Q: What's the risk of snapshot testing?**
Snapshots can become "approval tests" — `jest -u` updates them without reviewing what changed. Large snapshots obscure meaningful changes. Best practice: use small, focused snapshots for specific output, not full component renders.
