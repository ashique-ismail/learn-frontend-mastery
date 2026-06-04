# Testing Strategy: Practical Playbook

## Overview

"What's your testing strategy?" is a staff-level interview question because there is no single right answer — it requires judgment about ROI, team size, product risk, and the cost of false confidence. This document is the framework for making and defending those decisions.

---

## The Core Question: What Are You Protecting Against?

Before choosing a testing approach, name the failure modes you're trying to prevent:

| Failure mode | Cost | Best test type |
|---|---|---|
| Broken business logic (calculation wrong, edge case missed) | High | Unit test |
| Component renders wrong when data changes | Medium | Component/integration test |
| Two components don't work together | Medium-High | Integration test |
| Critical user journey broken after refactor | Very High | E2E test |
| API contract changed, frontend breaks silently | High | Contract test |
| Visual regression after CSS change | Medium | Visual regression |
| Accessible to keyboard/screen reader | High (legal risk) | Accessibility test |
| Performance degrades | Medium | Synthetic monitoring + perf budget |

Write tests where the cost of the failure mode exceeds the cost of writing and maintaining the test. Don't test where the failure mode would be caught immediately by a developer looking at the screen.

---

## The Testing Trophy (Kent C. Dodds Model)

```
         ╱‾‾‾‾‾‾‾‾‾‾‾‾╲
        ╱    E2E (few)  ╲      ← Happy path + critical journeys only
       ╱‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾╲
      ╱  Integration       ╲   ← Components with their real dependencies
     ╱    (most tests)      ╲  ← This is where the highest ROI lives
    ╱‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾╲
   ╱    Unit (some)           ╲ ← Pure functions, complex algorithms
  ╱‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾╲
 ╱   Static (always)           ╲ ← TypeScript, ESLint — free continuous testing
╱‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾╲
```

**The insight:** Integration tests at the component level (React Testing Library, Angular TestBed) give the best ROI because they test real behavior without the brittleness of E2E tests or the false confidence of unit tests that mock everything.

---

## Decision Framework: What Level for Each Test?

### Write a Unit Test When

```
✓ The logic is pure (no side effects, no DOM, no network)
✓ There are many edge cases that would be tedious to cover via component tests
✓ The function is complex enough to warrant isolated testing
✓ The function is reused by many components

Examples:
  - Date formatting utilities
  - Price calculation functions
  - Data transformation pipelines
  - Validation rule implementations
```

### Write a Component/Integration Test When

```
✓ You're testing user-visible behavior (click, type, see result)
✓ The component has conditional rendering
✓ The component has state that changes via interaction
✓ Multiple components collaborate to produce a result

Don't mock:
  - Child components (test the full subtree)
  - CSS modules
  - Routing (use MemoryRouter or test utilities)

Do mock:
  - Network calls (use MSW — intercept at the HTTP layer, not the module)
  - Time (use jest.useFakeTimers() for debounced/animated behavior)
  - External services (auth, analytics)
```

### Write an E2E Test When

```
✓ The journey is critical to the business (checkout, signup, core workflow)
✓ Multiple real services must work together
✓ The test would require extensive mocking at component level

Keep E2E tests small and focused:
  - One happy path per critical journey
  - One error path for the most likely failure
  - Don't test edge cases in E2E — that's what unit tests are for
```

---

## What to Mock and What Not To

### The MSW Pattern (Mock at HTTP, Not Module)

```typescript
// Bad: mock the module
jest.mock('../api/userService', () => ({
  getUser: jest.fn().mockResolvedValue({ id: '1', name: 'Alice' }),
}));

// Why it's bad:
// - Tests pass when the real API contract changes
// - Tests the mock, not the code path through the real function
// - Brittle (tied to implementation, not behavior)

// Good: mock at the HTTP layer with MSW
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';

const server = setupServer(
  http.get('/api/user/:id', ({ params }) => {
    return HttpResponse.json({ id: params.id, name: 'Alice' });
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

// Why it's better:
// - Tests the real fetch() path, including error handling
// - Works for any HTTP client (fetch, axios, etc.)
// - Can override per-test for error scenarios
```

### What to Always Mock

```typescript
// Date/time — tests must be deterministic
beforeEach(() => jest.useFakeTimers());
afterEach(() => jest.useRealTimers());
jest.setSystemTime(new Date('2024-01-15'));

// Randomness — prevent flaky tests
jest.spyOn(Math, 'random').mockReturnValue(0.5);

// External services you can't control
// Analytics, logging, crash reporting — mock these to avoid
// test output pollution and side effects
jest.mock('../analytics', () => ({ track: jest.fn() }));
```

---

## Coverage: What It Measures and What It Doesn't

### What Coverage Measures

Line/branch/statement coverage measures which lines were *executed* during tests, not whether the behavior was *verified*.

```typescript
// 100% line coverage — test is useless
function add(a, b) { return a + b; }

test('add', () => {
  add(1, 2); // executes the line — coverage: 100%
  // No assertion — we never verify the result
});
```

### When to Use Coverage

```
✓ As a ratchet — enforce that coverage never decreases
✓ To find untested code paths (low coverage = blind spots)
✗ As a quality metric — high coverage ≠ good tests
✗ As a goal — teams writing assertion-free tests to hit 80% are gaming the metric
```

**Practical coverage targets:**

```javascript
// jest.config.js
module.exports = {
  coverageThresholds: {
    global: {
      lines:      80,  // minimum, not a goal
      branches:   75,
      functions:  85,
      statements: 80,
    },
    // Stricter for critical paths
    './src/checkout/**': {
      lines: 95,
      branches: 90,
    },
  },
};
```

---

## Snapshot Testing: When It Helps and When It Hurts

### When snapshots help

```typescript
// Component that has complex, stable output that rarely changes
// and where visual inspection on every test failure is useful

test('renders invoice line items correctly', () => {
  const { container } = render(
    <InvoiceLineItems items={mockItems} currency="USD" />
  );
  expect(container).toMatchSnapshot();
  // On failure: diff shows exactly what changed in the rendered output
  // This is useful for a component that renders a complex table
});
```

### When snapshots hurt

```typescript
// Component that changes frequently — snapshots become noise

test('renders button', () => {
  const { container } = render(<Button>Click me</Button>);
  expect(container).toMatchSnapshot();
  // This snapshot will fail every time you:
  // - Change a class name
  // - Add a data-testid
  // - Update aria attributes
  // Developers start running `jest --updateSnapshot` without reading diffs
  // Snapshot tests provide false confidence — the component could be broken
  // but the snapshot still passes if you just accepted the new broken output
});

// Better: assert specific behavior, not full DOM structure
test('renders button with correct label', () => {
  render(<Button>Click me</Button>);
  expect(screen.getByRole('button', { name: 'Click me' })).toBeInTheDocument();
});
```

**Rule of thumb:** Snapshot the *output* of a pure transformation function (markdown renderer, code formatter). Don't snapshot full component trees.

---

## Testing Async Code Correctly

```typescript
// Common mistake: test passes before async code runs
test('shows loaded data', () => {
  render(<UserProfile userId="1" />);
  expect(screen.getByText('Alice')).toBeInTheDocument(); // ← runs before fetch resolves
  // Test passes because React hasn't updated yet — false positive
});

// Correct: wait for the expected state
test('shows loaded data', async () => {
  render(<UserProfile userId="1" />);

  // waitFor retries until the assertion passes or times out
  await waitFor(() => {
    expect(screen.getByText('Alice')).toBeInTheDocument();
  });
});

// Or use findBy* which has waitFor built in
test('shows loaded data', async () => {
  render(<UserProfile userId="1" />);
  const name = await screen.findByText('Alice'); // waits up to 1000ms
  expect(name).toBeInTheDocument();
});
```

---

## E2E Test Design for Stability

Flaky E2E tests are worse than no E2E tests — they erode trust in the test suite and get skipped or deleted.

```typescript
// Flaky: relies on timing
test('search returns results', async ({ page }) => {
  await page.fill('#search', 'laptop');
  await page.waitForTimeout(2000); // ← timing-based — flaky on slow CI
  expect(await page.locator('.results').count()).toBeGreaterThan(0);
});

// Stable: wait for the network request to complete
test('search returns results', async ({ page }) => {
  await page.fill('#search', 'laptop');
  await page.waitForResponse(resp => resp.url().includes('/api/search'));
  await expect(page.locator('.results')).not.toBeEmpty();
});

// Or: wait for the loading state to resolve
test('search returns results', async ({ page }) => {
  await page.fill('#search', 'laptop');
  await page.keyboard.press('Enter');
  await expect(page.getByRole('status')).toHaveText('Searching...');  // loading
  await expect(page.getByRole('status')).not.toBeVisible();           // done
  await expect(page.getByRole('list', { name: 'results' })).toBeVisible();
});
```

---

## The Testing Pyramid Anti-Patterns

```
Anti-pattern 1: Ice cream cone (too many E2E)
  Many slow, brittle E2E → few integration → almost no unit
  Result: CI takes 45 minutes, tests fail intermittently, developers ignore them

Anti-pattern 2: Inverted pyramid (too many units with heavy mocking)
  All unit tests mock dependencies → tests pass → app is broken
  Example: unit tests for each Redux action pass, but the actual
  dispatch → reducer → selector → component chain is never tested

Anti-pattern 3: Testing implementation, not behavior
  test('calls setLoading(true) when fetch starts')
  → This test breaks on every internal refactor
  → Tests internal state, not user-visible behavior
  Better: test('shows loading indicator while data is loading')
```

---

## Interview Questions

**Q: "How do you decide what to test and at what level?"**
A: I start by identifying the failure modes and their costs. High-cost failures (broken checkout, auth bypass, data loss) get thorough coverage across multiple levels. Lower-cost failures get proportionally less coverage. For most application code, the best ROI is component/integration tests with RTL that test user-visible behavior — not implementation. Unit tests for pure functions with complex edge cases. E2E only for critical user journeys where multiple real services must work together.

**Q: "We have 80% code coverage but our app breaks in production constantly. Why?"**
A: Coverage measures execution, not verification. You can have 100% coverage with no assertions. More commonly: tests are testing implementation details (mocked modules, internal state) rather than behavior. A test that mocks the API call verifies that the component calls the API, not that the user sees the right data. Switching from module mocking to MSW (HTTP-level interception) and from testing internal state to testing user-visible output gives you tests that would actually catch production failures.

**Q: "When would you use snapshot testing?"**
A: For stable, complex pure transformations — a markdown renderer, a code formatter, a complex serialization function — where the output is large but rarely intentionally changes. The snapshot diff is genuinely informative when it does change. I avoid snapshot tests for React component trees because any cosmetic change (new class, data attribute) fails the snapshot, developers learn to blindly accept updates, and the tests provide false confidence — a broken component gets committed with `jest --updateSnapshot`.
