# Snapshot Testing

## Overview

Snapshot testing captures the rendered output of a component and saves it as a reference. Future test runs compare the current output against the saved snapshot, detecting unintended changes. While powerful for preventing regressions, snapshots must be used judiciously to avoid becoming maintenance burdens.

## Table of Contents

1. [Snapshot Testing Philosophy](#snapshot-testing-philosophy)
2. [Basic Snapshot Testing](#basic-snapshot-testing)
3. [toMatchSnapshot()](#tomatchsnapshot)
4. [Inline Snapshots](#inline-snapshots)
5. [When Snapshots Are Useful](#when-snapshots-are-useful)
6. [Snapshot Pitfalls](#snapshot-pitfalls)
7. [Property Testing](#property-testing)
8. [Snapshot Testing with RTL](#snapshot-testing-with-rtl)
9. [Alternatives to Snapshots](#alternatives-to-snapshots)
10. [Common Mistakes](#common-mistakes)
11. [Best Practices](#best-practices)
12. [Interview Questions](#interview-questions)

## Snapshot Testing Philosophy

Snapshots verify component contracts and prevent unintended regressions.

### What Snapshots Test

```typescript
import { render } from '@testing-library/react';

interface UserCardProps {
  name: string;
  email: string;
  role: string;
}

function UserCard({ name, email, role }: UserCardProps) {
  return (
    <div className="user-card">
      <h2>{name}</h2>
      <p>{email}</p>
      <span className="role">{role}</span>
    </div>
  );
}

test('UserCard matches snapshot', () => {
  const { container } = render(
    <UserCard name="Alice" email="alice@example.com" role="Admin" />
  );
  
  expect(container.firstChild).toMatchSnapshot();
});

// Generated snapshot:
/*
exports[`UserCard matches snapshot 1`] = `
<div class="user-card">
  <h2>Alice</h2>
  <p>alice@example.com</p>
  <span class="role">Admin</span>
</div>
`;
*/
```

### Snapshot Purpose

```typescript
// ✅ GOOD: Snapshot documents component structure
test('Button component structure', () => {
  const { container } = render(
    <button className="btn btn-primary" disabled>
      Click me
    </button>
  );
  
  // Snapshot shows: classes, disabled state, content
  expect(container.firstChild).toMatchSnapshot();
});

// ❌ BAD: Using snapshot instead of specific assertion
test('Button shows correct text (BAD)', () => {
  const { container } = render(<button>Click me</button>);
  
  // Don't use snapshot for specific checks
  expect(container).toMatchSnapshot();
});

// ✅ GOOD: Specific assertion for specific behavior
test('Button shows correct text (GOOD)', () => {
  const { getByRole } = render(<button>Click me</button>);
  
  expect(getByRole('button')).toHaveTextContent('Click me');
});
```

## Basic Snapshot Testing

### Creating Your First Snapshot

```typescript
import { render } from '@testing-library/react';

function Alert({ type, message }: { type: 'success' | 'error'; message: string }) {
  return (
    <div className={`alert alert-${type}`} role="alert">
      {message}
    </div>
  );
}

test('Alert component snapshot', () => {
  const { container } = render(<Alert type="success" message="Operation successful!" />);
  
  expect(container.firstChild).toMatchSnapshot();
});

// First run creates __snapshots__/Alert.test.tsx.snap
// Subsequent runs compare against saved snapshot
```

### Multiple Snapshots in One Test

```typescript
test('Alert variations', () => {
  const { container, rerender } = render(
    <Alert type="success" message="Success!" />
  );
  
  expect(container.firstChild).toMatchSnapshot('success alert');
  
  rerender(<Alert type="error" message="Error!" />);
  
  expect(container.firstChild).toMatchSnapshot('error alert');
});

// Generates named snapshots:
/*
exports[`Alert variations success alert 1`] = `...`;
exports[`Alert variations error alert 1`] = `...`;
*/
```

### Snapshot Organization

```typescript
describe('Button', () => {
  test('default button', () => {
    const { container } = render(<button>Default</button>);
    expect(container.firstChild).toMatchSnapshot();
  });
  
  test('primary button', () => {
    const { container } = render(
      <button className="btn-primary">Primary</button>
    );
    expect(container.firstChild).toMatchSnapshot();
  });
  
  test('disabled button', () => {
    const { container } = render(<button disabled>Disabled</button>);
    expect(container.firstChild).toMatchSnapshot();
  });
});

// Generates organized snapshots:
/*
exports[`Button default button 1`] = `...`;
exports[`Button primary button 1`] = `...`;
exports[`Button disabled button 1`] = `...`;
*/
```

## toMatchSnapshot()

### Basic Usage

```typescript
test('component snapshot', () => {
  const { container } = render(<MyComponent />);
  
  // Snapshot the entire container
  expect(container).toMatchSnapshot();
  
  // Snapshot a specific element
  expect(container.firstChild).toMatchSnapshot();
});
```

### Named Snapshots

```typescript
test('form states', () => {
  const { container, rerender } = render(<Form />);
  
  expect(container).toMatchSnapshot('initial state');
  
  // Simulate user interaction
  rerender(<Form submitted={true} />);
  
  expect(container).toMatchSnapshot('submitted state');
  
  rerender(<Form error="Invalid input" />);
  
  expect(container).toMatchSnapshot('error state');
});
```

### Property Matchers

```typescript
test('user object with dynamic date', () => {
  const user = {
    id: 1,
    name: 'Alice',
    createdAt: new Date(),
  };
  
  // Use property matchers for dynamic values
  expect(user).toMatchSnapshot({
    createdAt: expect.any(Date),
  });
});

test('API response with timestamps', () => {
  const response = {
    data: { id: 1, title: 'Post' },
    timestamp: Date.now(),
    requestId: '123e4567-e89b-12d3-a456-426614174000',
  };
  
  expect(response).toMatchSnapshot({
    timestamp: expect.any(Number),
    requestId: expect.any(String),
  });
});
```

### Snapshot Serializers

```typescript
// Custom serializer for Date objects
expect.addSnapshotSerializer({
  test: (val) => val instanceof Date,
  print: (val) => `Date<${(val as Date).toISOString()}>`,
});

test('component with date', () => {
  const { container } = render(
    <div>Created: {new Date('2024-01-01').toISOString()}</div>
  );
  
  expect(container).toMatchSnapshot();
});
```

## Inline Snapshots

Inline snapshots embed the snapshot directly in the test file for easier code reviews.

### Basic Inline Snapshots

```typescript
import { render } from '@testing-library/react';

function Badge({ count }: { count: number }) {
  return <span className="badge">{count}</span>;
}

test('Badge component', () => {
  const { container } = render(<Badge count={5} />);
  
  expect(container.firstChild).toMatchInlineSnapshot(`
    <span
      class="badge"
    >
      5
    </span>
  `);
});
```

### Benefits of Inline Snapshots

```typescript
// ✅ GOOD: Inline snapshot visible in code review
test('Alert message', () => {
  const { container } = render(
    <Alert type="warning" message="Check your input" />
  );
  
  expect(container.firstChild).toMatchInlineSnapshot(`
    <div
      class="alert alert-warning"
      role="alert"
    >
      Check your input
    </div>
  `);
});

// Changes are immediately visible in PR diffs
```

### When to Use Inline vs File Snapshots

```typescript
// Use inline for small, frequently changed components
test('Icon component', () => {
  const { container } = render(<Icon name="check" />);
  
  expect(container.firstChild).toMatchInlineSnapshot(`
    <svg class="icon icon-check">
      <use xlink:href="#icon-check" />
    </svg>
  `);
});

// Use file snapshots for large components
test('ComplexDashboard', () => {
  const { container } = render(<ComplexDashboard />);
  
  // Large snapshot in separate file
  expect(container).toMatchSnapshot();
});
```

## When Snapshots Are Useful

### 1. Component API Contracts

```typescript
interface CardProps {
  title: string;
  subtitle?: string;
  actions?: React.ReactNode;
  children: React.ReactNode;
}

function Card({ title, subtitle, actions, children }: CardProps) {
  return (
    <div className="card">
      <div className="card-header">
        <h3>{title}</h3>
        {subtitle && <p>{subtitle}</p>}
      </div>
      <div className="card-body">{children}</div>
      {actions && <div className="card-actions">{actions}</div>}
    </div>
  );
}

test('Card with all props', () => {
  const { container } = render(
    <Card
      title="Card Title"
      subtitle="Card Subtitle"
      actions={<button>Action</button>}
    >
      Card content
    </Card>
  );
  
  // Snapshot documents complete API
  expect(container.firstChild).toMatchSnapshot();
});

test('Card with minimal props', () => {
  const { container } = render(
    <Card title="Title">Content</Card>
  );
  
  expect(container.firstChild).toMatchSnapshot();
});
```

### 2. Preventing Regressions

```typescript
function PricingTable({ plans }: { plans: Plan[] }) {
  return (
    <table className="pricing-table">
      <thead>
        <tr>
          <th>Plan</th>
          <th>Price</th>
          <th>Features</th>
        </tr>
      </thead>
      <tbody>
        {plans.map(plan => (
          <tr key={plan.id}>
            <td>{plan.name}</td>
            <td>${plan.price}</td>
            <td>{plan.features.join(', ')}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}

test('pricing table structure', () => {
  const plans = [
    { id: 1, name: 'Basic', price: 10, features: ['Feature A', 'Feature B'] },
    { id: 2, name: 'Pro', price: 20, features: ['All Basic', 'Feature C'] },
  ];
  
  const { container } = render(<PricingTable plans={plans} />);
  
  // Snapshot catches structural changes
  expect(container.firstChild).toMatchSnapshot();
});
```

### 3. Stable Component Libraries

```typescript
// Component library with stable APIs
describe('UI Components', () => {
  test('Button variants', () => {
    const variants = ['primary', 'secondary', 'danger'] as const;
    
    variants.forEach(variant => {
      const { container } = render(
        <button className={`btn btn-${variant}`}>
          {variant} button
        </button>
      );
      
      expect(container.firstChild).toMatchSnapshot(variant);
    });
  });
  
  test('Input sizes', () => {
    const sizes = ['sm', 'md', 'lg'] as const;
    
    sizes.forEach(size => {
      const { container } = render(
        <input className={`input input-${size}`} />
      );
      
      expect(container.firstChild).toMatchSnapshot(size);
    });
  });
});
```

## Snapshot Pitfalls

### 1. Too Many Snapshots

```typescript
// ❌ BAD: Snapshot for everything
test('counter increments (BAD)', () => {
  const { container } = render(<Counter />);
  
  expect(container).toMatchSnapshot('initial');
  
  fireEvent.click(screen.getByRole('button'));
  
  expect(container).toMatchSnapshot('incremented');
  
  fireEvent.click(screen.getByRole('button'));
  
  expect(container).toMatchSnapshot('incremented again');
});

// ✅ GOOD: Specific assertions
test('counter increments (GOOD)', () => {
  render(<Counter />);
  
  const button = screen.getByRole('button', { name: /count: 0/i });
  
  fireEvent.click(button);
  expect(screen.getByRole('button', { name: /count: 1/i })).toBeInTheDocument();
  
  fireEvent.click(button);
  expect(screen.getByRole('button', { name: /count: 2/i })).toBeInTheDocument();
});
```

### 2. Blind Updates (-u)

```typescript
// ❌ BAD: Updating snapshots without review
// Running: jest -u
// This updates ALL snapshots, including breaking changes!

// ✅ GOOD: Review each snapshot change
// 1. Run tests: jest
// 2. Review failed snapshots
// 3. If change is intentional, update specific snapshot
// 4. Re-run tests to verify
```

### 3. Implementation Detail Snapshots

```typescript
// ❌ BAD: Snapshotting internal structure
test('user list (BAD)', () => {
  const { container } = render(<UserList users={users} />);
  
  // This breaks on any CSS class change
  expect(container.querySelector('.user-list-wrapper')).toMatchSnapshot();
});

// ✅ GOOD: Snapshot stable contract
test('user list (GOOD)', () => {
  render(<UserList users={users} />);
  
  // Test behavior, not implementation
  expect(screen.getAllByRole('listitem')).toHaveLength(users.length);
  users.forEach(user => {
    expect(screen.getByText(user.name)).toBeInTheDocument();
  });
});
```

### 4. Dynamic Values in Snapshots

```typescript
// ❌ BAD: Unstable snapshots
test('post with timestamp (BAD)', () => {
  const { container } = render(
    <Post
      title="My Post"
      timestamp={new Date()}  // Changes every test run!
    />
  );
  
  expect(container).toMatchSnapshot();
});

// ✅ GOOD: Use property matchers or fixed values
test('post with timestamp (GOOD)', () => {
  const fixedDate = new Date('2024-01-01');
  const { container } = render(
    <Post title="My Post" timestamp={fixedDate} />
  );
  
  expect(container).toMatchSnapshot();
});
```

## Property Testing

Property-based testing generates random inputs to find edge cases.

### Basic Property Testing

```typescript
import fc from 'fast-check';

function slugify(text: string): string {
  return text
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, '-')
    .replace(/^-|-$/g, '');
}

test('slugify properties', () => {
  fc.assert(
    fc.property(fc.string(), (input) => {
      const slug = slugify(input);
      
      // Property: output only contains lowercase letters, numbers, and hyphens
      expect(slug).toMatch(/^[a-z0-9-]*$/);
      
      // Property: no leading/trailing hyphens
      expect(slug).not.toMatch(/^-|-$/);
    })
  );
});
```

### Property Testing vs Snapshots

```typescript
// Property testing finds edge cases snapshots miss
function formatPrice(cents: number): string {
  return `$${(cents / 100).toFixed(2)}`;
}

test('formatPrice snapshot', () => {
  expect(formatPrice(1099)).toMatchInlineSnapshot(`"$10.99"`);
  expect(formatPrice(100)).toMatchInlineSnapshot(`"$1.00"`);
});

test('formatPrice properties', () => {
  fc.assert(
    fc.property(fc.integer({ min: 0, max: 1000000 }), (cents) => {
      const formatted = formatPrice(cents);
      
      // Always starts with $
      expect(formatted).toMatch(/^\$/);
      
      // Always has exactly 2 decimal places
      expect(formatted).toMatch(/\.\d{2}$/);
      
      // Can be parsed back to number
      const parsed = parseFloat(formatted.slice(1)) * 100;
      expect(parsed).toBe(cents);
    })
  );
});
```

## Snapshot Testing with RTL

### Container vs Screen Snapshots

```typescript
import { render, screen } from '@testing-library/react';

function LoginForm() {
  return (
    <form>
      <label htmlFor="email">Email</label>
      <input id="email" type="email" />
      
      <label htmlFor="password">Password</label>
      <input id="password" type="password" />
      
      <button type="submit">Login</button>
    </form>
  );
}

// ✅ Use container for snapshots
test('LoginForm structure', () => {
  const { container } = render(<LoginForm />);
  
  expect(container.firstChild).toMatchSnapshot();
});

// ✅ Use screen for behavior tests
test('LoginForm has all fields', () => {
  render(<LoginForm />);
  
  expect(screen.getByLabelText('Email')).toBeInTheDocument();
  expect(screen.getByLabelText('Password')).toBeInTheDocument();
  expect(screen.getByRole('button', { name: 'Login' })).toBeInTheDocument();
});
```

### Snapshot-Assisted Testing

```typescript
test('UserProfile renders correctly', () => {
  const user = {
    name: 'Alice',
    email: 'alice@example.com',
    bio: 'Software developer',
  };
  
  const { container } = render(<UserProfile user={user} />);
  
  // Snapshot for structure
  expect(container.firstChild).toMatchSnapshot();
  
  // Specific assertions for critical behavior
  expect(screen.getByText(user.name)).toBeInTheDocument();
  expect(screen.getByText(user.email)).toBeInTheDocument();
  expect(screen.getByText(user.bio)).toBeInTheDocument();
});
```

## Alternatives to Snapshots

### 1. Visual Regression Testing

```typescript
// Percy visual testing
import percySnapshot from '@percy/puppeteer';

test('HomePage visual regression', async () => {
  await page.goto('http://localhost:3000');
  await percySnapshot(page, 'HomePage');
});

// Chromatic visual testing
test('Button visual regression', () => {
  render(<Button>Click me</Button>);
  
  // Chromatic captures screenshots automatically in CI
});
```

### 2. Specific Assertions

```typescript
// ❌ Snapshot for simple checks
test('displays user name (snapshot)', () => {
  const { container } = render(<UserCard name="Alice" />);
  expect(container).toMatchSnapshot();
});

// ✅ Specific assertion
test('displays user name (specific)', () => {
  render(<UserCard name="Alice" />);
  expect(screen.getByText('Alice')).toBeInTheDocument();
});
```

### 3. Contract Testing

```typescript
// Test component props contract
describe('Button contract', () => {
  test('renders children', () => {
    render(<Button>Click me</Button>);
    expect(screen.getByText('Click me')).toBeInTheDocument();
  });
  
  test('applies variant class', () => {
    render(<Button variant="primary">Click</Button>);
    expect(screen.getByRole('button')).toHaveClass('btn-primary');
  });
  
  test('handles onClick', () => {
    const onClick = jest.fn();
    render(<Button onClick={onClick}>Click</Button>);
    fireEvent.click(screen.getByRole('button'));
    expect(onClick).toHaveBeenCalledTimes(1);
  });
  
  test('can be disabled', () => {
    render(<Button disabled>Click</Button>);
    expect(screen.getByRole('button')).toBeDisabled();
  });
});
```

## Common Mistakes

### 1. Snapshot Everything

```typescript
// ❌ WRONG: Unnecessary snapshots
test('counter', () => {
  const { container } = render(<Counter />);
  expect(container).toMatchSnapshot();
  
  fireEvent.click(screen.getByRole('button'));
  expect(container).toMatchSnapshot();
});

// ✅ CORRECT: Meaningful assertions
test('counter', () => {
  render(<Counter />);
  
  const button = screen.getByRole('button');
  expect(button).toHaveTextContent('Count: 0');
  
  fireEvent.click(button);
  expect(button).toHaveTextContent('Count: 1');
});
```

### 2. Not Reviewing Snapshot Changes

```typescript
// ❌ WRONG: Blindly accepting changes
// jest -u  // Updates all snapshots!

// ✅ CORRECT: Review each change
// 1. Check which snapshots failed
// 2. Review the diff carefully
// 3. Verify the change is intentional
// 4. Update the snapshot if correct
```

### 3. Snapshotting Volatile Data

```typescript
// ❌ WRONG: Snapshot with random/time-based data
test('post metadata (BAD)', () => {
  const { container } = render(
    <PostMeta
      id={Math.random()}
      createdAt={new Date()}
    />
  );
  
  expect(container).toMatchSnapshot();
});

// ✅ CORRECT: Fixed test data
test('post metadata (GOOD)', () => {
  const { container } = render(
    <PostMeta
      id={123}
      createdAt={new Date('2024-01-01')}
    />
  );
  
  expect(container).toMatchSnapshot();
});
```

## Best Practices

### 1. Use Snapshots Sparingly

```typescript
// ✅ Good snapshot use cases:
// - Component library documentation
// - Stable component contracts
// - Regression prevention for complex output

// ❌ Poor snapshot use cases:
// - Testing specific values (use toBe/toEqual)
// - Testing user interactions (use RTL queries)
// - Rapidly changing components
```

### 2. Review Snapshot Diffs

```typescript
// Always review snapshot changes in PR reviews
// Question: "Why did this snapshot change?"
// - Intentional refactoring? ✅
// - Bug fix? ✅
// - Accidental breakage? ❌
```

### 3. Keep Snapshots Small

```typescript
// ✅ GOOD: Snapshot specific element
test('navbar structure', () => {
  const { container } = render(<App />);
  
  const navbar = container.querySelector('nav');
  expect(navbar).toMatchSnapshot();
});

// ❌ BAD: Snapshot entire app
test('app structure', () => {
  const { container } = render(<App />);
  
  expect(container).toMatchSnapshot();
});
```

### 4. Name Snapshots Clearly

```typescript
test('Button variants', () => {
  const { container, rerender } = render(<Button variant="primary">Click</Button>);
  expect(container.firstChild).toMatchSnapshot('primary button');
  
  rerender(<Button variant="secondary">Click</Button>);
  expect(container.firstChild).toMatchSnapshot('secondary button');
  
  rerender(<Button variant="danger">Click</Button>);
  expect(container.firstChild).toMatchSnapshot('danger button');
});
```

## Interview Questions

### Q1: What is snapshot testing and when should you use it?

**Answer:** Snapshot testing captures the rendered output of a component and saves it as a reference for future comparisons. Use it for stable component APIs, preventing regressions in component libraries, and documenting component structure. Avoid it for testing specific values, user interactions, or frequently changing components.

### Q2: What's the difference between toMatchSnapshot() and toMatchInlineSnapshot()?

**Answer:** `toMatchSnapshot()` saves snapshots in a separate `__snapshots__` directory, while `toMatchInlineSnapshot()` embeds the snapshot directly in the test file. Inline snapshots are better for code reviews as changes are visible in the PR diff, but file snapshots are better for large components.

### Q3: What are common pitfalls of snapshot testing?

**Answer:** Common pitfalls include: creating too many snapshots that become maintenance burdens, blindly updating snapshots with `jest -u` without reviewing changes, snapshotting implementation details instead of contracts, and including dynamic values (timestamps, random IDs) that make snapshots brittle.

### Q4: How do you handle dynamic values in snapshots?

**Answer:** Use property matchers to match dynamic values by type rather than exact value: `expect(obj).toMatchSnapshot({ timestamp: expect.any(Number) })`. Alternatively, use fixed values in tests, or create custom snapshot serializers to normalize dynamic data.

### Q5: What's the difference between snapshot testing and visual regression testing?

**Answer:** Snapshot testing compares DOM structure (HTML/JSX), while visual regression testing compares actual rendered pixels using screenshots. Visual regression catches CSS/styling issues that snapshots miss, using tools like Percy, Chromatic, or Playwright's screenshot testing.

### Q6: When should you update a snapshot?

**Answer:** Update snapshots only when the change is intentional: refactoring component structure, fixing bugs that change output, or updating component APIs. Always review the diff before updating. Never blindly update all snapshots with `jest -u` without understanding what changed.

## Key Takeaways

1. Snapshots document component structure and prevent regressions
2. Use sparingly—prefer specific assertions for behavior
3. Review snapshot changes carefully in code reviews
4. Never blindly update snapshots with `-u`
5. Inline snapshots better for small components and code reviews
6. File snapshots better for large, complex components
7. Avoid snapshotting implementation details
8. Use property matchers for dynamic values
9. Keep snapshots small and focused
10. Consider visual regression testing for CSS/styling
11. Name snapshots clearly for better organization
12. Snapshot stable APIs, not volatile code

## Resources

- [Jest Snapshot Testing](https://jestjs.io/docs/snapshot-testing)
- [Effective Snapshot Testing](https://kentcdodds.com/blog/effective-snapshot-testing)
- [RTL and Snapshots](https://testing-library.com/docs/react-testing-library/api/#container)
- [Percy Visual Testing](https://percy.io/)
- [Chromatic](https://www.chromatic.com/)
- [fast-check Property Testing](https://github.com/dubzzz/fast-check)
