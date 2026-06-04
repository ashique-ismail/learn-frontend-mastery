# Snapshot Testing

## Overview

Snapshot testing captures the output of a component or function and saves it as a reference "snapshot". Future test runs compare current output against the stored snapshot, failing if they don't match. While snapshot tests catch unexpected changes quickly, they must be used judiciously to avoid brittle tests that create maintenance burden without providing real value.

## How Snapshot Testing Works

```
First Run:              Subsequent Runs:
┌─────────────┐        ┌─────────────┐
│  Component  │        │  Component  │
└──────┬──────┘        └──────┬──────┘
       │                      │
       ▼                      ▼
  [Render Output]        [Render Output]
       │                      │
       ▼                      ▼
  Save Snapshot           Compare with
  to __snapshots__        Saved Snapshot
       │                      │
       └──────────────────────┼─── Match? → Pass ✅
                              │
                              └─── Diff? → Fail ❌
```

## Basic Snapshot Testing

### React Component Snapshot

```tsx
// UserCard.tsx
interface UserCardProps {
  name: string;
  email: string;
  role: 'admin' | 'user';
}

export const UserCard: React.FC<UserCardProps> = ({ name, email, role }) => {
  return (
    <div className="user-card">
      <h3>{name}</h3>
      <p>{email}</p>
      <span className={`badge ${role}`}>{role}</span>
    </div>
  );
};

// UserCard.test.tsx
import { render } from '@testing-library/react';
import { UserCard } from './UserCard';

describe('UserCard', () => {
  it('should match snapshot', () => {
    const { container } = render(
      <UserCard name="John Doe" email="john@example.com" role="admin" />
    );
    
    expect(container).toMatchSnapshot();
  });
});

// First run creates: __snapshots__/UserCard.test.tsx.snap
/*
// Jest Snapshot v1

exports[`UserCard should match snapshot 1`] = `
<div>
  <div
    class="user-card"
  >
    <h3>
      John Doe
    </h3>
    <p>
      john@example.com
    </p>
    <span
      class="badge admin"
    >
      admin
    </span>
  </div>
</div>
`;
*/
```

### Updating Snapshots

```bash
# When component intentionally changes, update snapshots:
npm test -- -u

# Or interactively:
npm test -- --watch
# Press 'u' to update failing snapshots
```

### Inline Snapshots

```tsx
// Instead of separate file, snapshot is inline
it('should render user info', () => {
  const { container } = render(
    <UserCard name="John" email="john@example.com" role="user" />
  );
  
  expect(container.firstChild).toMatchInlineSnapshot(`
    <div class="user-card">
      <h3>John</h3>
      <p>john@example.com</p>
      <span class="badge user">user</span>
    </div>
  `);
});

// Jest automatically writes the snapshot inline
```

## Partial Snapshots

### Property Matchers

```tsx
interface User {
  id: string;
  name: string;
  createdAt: Date;
}

describe('User API', () => {
  it('should create user', async () => {
    const user = await createUser({ name: 'John' });
    
    // ❌ BAD: Snapshot includes dynamic values
    expect(user).toMatchSnapshot();
    // Snapshot will have specific id and timestamp, causing failures
    
    // ✅ GOOD: Match dynamic properties
    expect(user).toMatchSnapshot({
      id: expect.any(String),
      createdAt: expect.any(Date)
    });
  });
});

// Snapshot only captures stable properties:
/*
exports[`should create user 1`] = `
{
  "createdAt": Any<Date>,
  "id": Any<String>,
  "name": "John",
}
`;
*/
```

### Snapshot Specific Parts

```tsx
// ❌ BAD: Snapshot entire component with dynamic data
it('should render dashboard', () => {
  const { container } = render(<Dashboard />);
  expect(container).toMatchSnapshot();
  // Includes timestamps, random IDs, etc.
});

// ✅ GOOD: Snapshot specific, stable parts
it('should render navigation', () => {
  const { getByRole } = render(<Dashboard />);
  const nav = getByRole('navigation');
  expect(nav).toMatchSnapshot();
  // Only navigation structure, no dynamic content
});
```

## When Snapshots Are Useful

### 1. Component Structure Testing

```tsx
// ✅ GOOD: Snapshot for component structure
describe('ProductCard Structure', () => {
  it('should have correct HTML structure', () => {
    const { container } = render(
      <ProductCard
        name="Test Product"
        price={10.99}
        image="/test.jpg"
      />
    );
    
    expect(container.firstChild).toMatchSnapshot();
  });
});

// Catches unintended structural changes:
// - Removed elements
// - Changed class names
// - Altered hierarchy
```

### 2. API Response Shapes

```typescript
// ✅ GOOD: Snapshot API response structure
describe('User API', () => {
  it('should return user with correct shape', async () => {
    const response = await fetch('/api/users/123');
    const user = await response.json();
    
    expect(user).toMatchSnapshot({
      id: expect.any(String),
      createdAt: expect.any(String)
    });
  });
});

// Ensures API contract doesn't break
```

### 3. Error Messages

```typescript
// ✅ GOOD: Snapshot error messages
describe('Validator', () => {
  it('should return validation errors', () => {
    const errors = validateUserInput({
      email: 'invalid',
      password: '123'
    });
    
    expect(errors).toMatchSnapshot();
  });
});

// Snapshot:
/*
exports[`should return validation errors 1`] = `
[
  {
    "field": "email",
    "message": "Invalid email format",
  },
  {
    "field": "password",
    "message": "Password must be at least 8 characters",
  },
]
`;
*/
```

### 4. CLI Output

```typescript
// ✅ GOOD: Snapshot command-line output
describe('CLI', () => {
  it('should display help text', () => {
    const output = getHelpText();
    expect(output).toMatchSnapshot();
  });
});

// Ensures help text consistency
```

## When Snapshots Are NOT Useful

### 1. Testing Behavior

```tsx
// ❌ BAD: Snapshot for behavior testing
it('should increment counter', () => {
  const { container } = render(<Counter />);
  fireEvent.click(screen.getByText('Increment'));
  expect(container).toMatchSnapshot();
});

// ✅ GOOD: Explicit assertion for behavior
it('should increment counter', () => {
  render(<Counter />);
  fireEvent.click(screen.getByText('Increment'));
  expect(screen.getByText('Count: 1')).toBeInTheDocument();
});
```

### 2. Dynamic Content

```tsx
// ❌ BAD: Snapshot with timestamps
it('should display current time', () => {
  const { container } = render(<Clock />);
  expect(container).toMatchSnapshot();
  // Will fail every second!
});

// ✅ GOOD: Test the format, not the value
it('should display time in correct format', () => {
  render(<Clock />);
  expect(screen.getByText(/\d{2}:\d{2}:\d{2}/)).toBeInTheDocument();
});
```

### 3. Large Components

```tsx
// ❌ BAD: Snapshot entire page
it('should render dashboard', () => {
  const { container } = render(<DashboardPage />);
  expect(container).toMatchSnapshot();
  // 1000+ line snapshot, impossible to review
});

// ✅ GOOD: Test specific aspects
it('should display welcome message', () => {
  render(<DashboardPage user={{ name: 'John' }} />);
  expect(screen.getByText('Welcome, John')).toBeInTheDocument();
});

it('should render navigation', () => {
  const { getByRole } = render(<DashboardPage />);
  const nav = getByRole('navigation');
  expect(nav).toMatchSnapshot(); // Small, focused snapshot
});
```

## Snapshot Serializers

### Custom Serializers

```typescript
// test-utils/serializers.ts
import { addSerializer } from 'jest-specific-snapshot';

// Remove dynamic attributes
addSerializer({
  test: (val) => val && val.$$typeof === Symbol.for('react.element'),
  print: (val, serialize) => {
    const copy = { ...val };
    // Remove dynamic props
    if (copy.props) {
      delete copy.props.key;
      delete copy.props.ref;
    }
    return serialize(copy);
  }
});

// Format dates consistently
expect.addSnapshotSerializer({
  test: (val) => val instanceof Date,
  print: (val) => `Date(${val.toISOString()})`
});
```

### Emotion Serializer (CSS-in-JS)

```tsx
import { matchers } from '@emotion/jest';
import serializer from '@emotion/jest/serializer';

expect.extend(matchers);
expect.addSnapshotSerializer(serializer);

// Now snapshots include computed styles
it('should style button correctly', () => {
  const { container } = render(<PrimaryButton>Click me</PrimaryButton>);
  expect(container).toMatchSnapshot();
});

// Snapshot includes CSS:
/*
exports[`should style button correctly 1`] = `
.emotion-0 {
  background-color: blue;
  color: white;
  padding: 8px 16px;
  border-radius: 4px;
}

<div>
  <button className="emotion-0">
    Click me
  </button>
</div>
`;
*/
```

## Snapshot Testing with Angular

```typescript
// app.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { AppComponent } from './app.component';

describe('AppComponent', () => {
  let component: AppComponent;
  let fixture: ComponentFixture<AppComponent>;

  beforeEach(() => {
    TestBed.configureTestingModule({
      declarations: [AppComponent]
    });

    fixture = TestBed.createComponent(AppComponent);
    component = fixture.componentInstance;
  });

  it('should match snapshot', () => {
    fixture.detectChanges();
    expect(fixture).toMatchSnapshot();
  });

  // More focused snapshot
  it('should render navigation', () => {
    fixture.detectChanges();
    const nav = fixture.nativeElement.querySelector('nav');
    expect(nav).toMatchSnapshot();
  });
});
```

## Managing Snapshot Changes

### Reviewing Snapshot Changes

```bash
# See what changed
npm test

# Output shows diff:
# - Expected
# + Received

# -   <span class="badge user">
# +   <span class="badge admin">
```

### Intentional Changes

```bash
# Update all snapshots
npm test -- -u

# Update specific test file
npm test UserCard.test.tsx -u

# Interactive mode - review each change
npm test -- --watch
# Press 'i' to update one snapshot at a time
```

### Preventing Accidental Updates

```json
// package.json
{
  "scripts": {
    "test": "jest",
    "test:update-snapshots": "jest -u"
  }
}

// CI configuration - fail if snapshots need updating
{
  "scripts": {
    "test:ci": "jest --ci"
  }
}
```

## Best Practices for Snapshot Testing

### 1. Keep Snapshots Small

```tsx
// ❌ BAD: Large snapshot
it('should render page', () => {
  const { container } = render(<EntirePage />);
  expect(container).toMatchSnapshot(); // 500 lines
});

// ✅ GOOD: Focused snapshots
it('should render header', () => {
  const { getByRole } = render(<EntirePage />);
  expect(getByRole('banner')).toMatchSnapshot(); // 20 lines
});

it('should render navigation', () => {
  const { getByRole } = render(<EntirePage />);
  expect(getByRole('navigation')).toMatchSnapshot(); // 15 lines
});
```

### 2. Use Property Matchers for Dynamic Data

```tsx
// ❌ BAD: Snapshot includes dynamic ID
it('should create order', async () => {
  const order = await createOrder({ items: [...] });
  expect(order).toMatchSnapshot();
  // Includes specific ID, will fail on each run
});

// ✅ GOOD: Match dynamic properties
it('should create order', async () => {
  const order = await createOrder({ items: [...] });
  expect(order).toMatchSnapshot({
    id: expect.any(String),
    createdAt: expect.any(String),
    items: expect.arrayContaining([
      expect.objectContaining({
        id: expect.any(String)
      })
    ])
  });
});
```

### 3. Name Tests Descriptively

```tsx
// ❌ BAD: Generic name
it('test 1', () => {
  expect(component).toMatchSnapshot();
});

// Snapshot file:
// exports[`test 1 1`] = `...`; // What does this test?

// ✅ GOOD: Descriptive name
it('should render admin badge for admin users', () => {
  expect(component).toMatchSnapshot();
});

// Snapshot file:
// exports[`should render admin badge for admin users 1`] = `...`;
```

### 4. Don't Snapshot Everything

```tsx
// ❌ BAD: Snapshot overkill
describe('Button', () => {
  it('should render', () => {
    expect(render(<Button />)).toMatchSnapshot();
  });

  it('should render with text', () => {
    expect(render(<Button>Click</Button>)).toMatchSnapshot();
  });

  it('should render disabled', () => {
    expect(render(<Button disabled />)).toMatchSnapshot();
  });

  it('should render loading', () => {
    expect(render(<Button loading />)).toMatchSnapshot();
  });
  
  // 50 more snapshot tests...
});

// ✅ GOOD: Snapshot for structure, assertions for behavior
describe('Button', () => {
  it('should have correct structure', () => {
    const { container } = render(<Button>Click</Button>);
    expect(container.firstChild).toMatchSnapshot();
  });

  it('should be disabled when disabled prop is true', () => {
    render(<Button disabled>Click</Button>);
    expect(screen.getByRole('button')).toBeDisabled();
  });

  it('should show loading spinner when loading', () => {
    render(<Button loading>Click</Button>);
    expect(screen.getByRole('progressbar')).toBeInTheDocument();
  });
});
```

## Snapshot Testing Alternatives

### Visual Regression Testing

```typescript
// Instead of DOM snapshots, use visual screenshots
import { test, expect } from '@playwright/test';

test('should render correctly', async ({ page }) => {
  await page.goto('/');
  await expect(page).toHaveScreenshot('homepage.png');
});

// Better for:
// - CSS changes
// - Layout issues
// - Visual consistency
```

### Explicit Assertions

```tsx
// Instead of snapshot
it('should render user card', () => {
  const { container } = render(<UserCard name="John" email="john@example.com" />);
  expect(container).toMatchSnapshot();
});

// Use explicit assertions
it('should render user card', () => {
  render(<UserCard name="John" email="john@example.com" />);
  
  expect(screen.getByRole('heading', { name: 'John' })).toBeInTheDocument();
  expect(screen.getByText('john@example.com')).toBeInTheDocument();
  expect(screen.getByRole('img')).toHaveAttribute('alt', 'John');
});

// More maintainable, clearer intent
```

## Common Mistakes

### 1. Not Reviewing Snapshot Changes

```bash
# ❌ BAD: Blindly updating snapshots
npm test -- -u
# Without reviewing what changed!

# ✅ GOOD: Review changes first
npm test
# Read the diff carefully
# Understand why it changed
# Then update if intentional
npm test -- -u
```

### 2. Snapshots as Primary Test Strategy

```tsx
// ❌ BAD: Only snapshot tests
describe('ShoppingCart', () => {
  it('empty cart snapshot', () => {
    expect(render(<ShoppingCart items={[]} />)).toMatchSnapshot();
  });

  it('cart with items snapshot', () => {
    expect(render(<ShoppingCart items={mockItems} />)).toMatchSnapshot();
  });
});

// ✅ GOOD: Snapshots supplement behavior tests
describe('ShoppingCart', () => {
  it('should display empty message when no items', () => {
    render(<ShoppingCart items={[]} />);
    expect(screen.getByText('Your cart is empty')).toBeInTheDocument();
  });

  it('should calculate total correctly', () => {
    render(<ShoppingCart items={mockItems} />);
    expect(screen.getByText('Total: $99.99')).toBeInTheDocument();
  });

  it('should have correct HTML structure', () => {
    const { container } = render(<ShoppingCart items={mockItems} />);
    expect(container.querySelector('.cart-items')).toMatchSnapshot();
  });
});
```

### 3. Committing Failing Snapshots

```json
// ❌ BAD: CI with failing snapshots
// .github/workflows/test.yml
{
  "test": "jest" // Allows snapshot updates in CI
}

// ✅ GOOD: CI that enforces snapshot consistency
{
  "test:ci": "jest --ci" // Fails if snapshots need update
}
```

## Best Practices Summary

1. **Keep Snapshots Small**: Focus on specific parts, not entire pages
2. **Review Changes**: Always review snapshot diffs before updating
3. **Use Property Matchers**: Handle dynamic data appropriately
4. **Descriptive Names**: Make snapshot intent clear
5. **Don't Overuse**: Snapshots are not a replacement for behavior tests
6. **Version Control**: Commit snapshots with code changes
7. **Update Intentionally**: Never blindly update all snapshots
8. **CI Enforcement**: Fail CI if snapshots need updating
9. **Consider Alternatives**: Visual regression for UI, explicit assertions for behavior
10. **Document Intent**: Comment why snapshot is needed

## When to Use Snapshot Testing

**Use snapshot testing for:**
- Component HTML structure
- API response shapes
- CLI output formatting
- Error message consistency
- Small, stable components
- Detecting unintended changes

**Don't use snapshot testing for:**
- Behavior testing
- Dynamic or changing content
- Large components
- As primary testing strategy
- When explicit assertions are clearer
- Frequently changing code

## Interview Questions

1. **What is snapshot testing?**
   - Captures component output and compares future renders against saved snapshot to detect unexpected changes.

2. **When should you use snapshot testing?**
   - For component structure, API shapes, error messages, CLI output. Not for behavior, dynamic content, or large components.

3. **What's the risk of snapshot testing?**
   - Brittle tests, developers blindly updating snapshots, large unreadable snapshots, false sense of security.

4. **How do you handle dynamic data in snapshots?**
   - Use property matchers (expect.any(String)), snapshot specific parts, or use explicit assertions instead.

5. **What's better than snapshot testing for UI?**
   - Visual regression testing with screenshots, explicit assertions for behavior, integration tests for user flows.

## Key Takeaways

1. Snapshots capture output for regression detection
2. Keep snapshots small and focused
3. Always review snapshot changes before updating
4. Use property matchers for dynamic data
5. Don't replace behavior tests with snapshots
6. Snapshots work well for structure, not behavior
7. Consider visual regression for UI testing
8. Commit snapshots with code changes
9. Enforce snapshot consistency in CI
10. Use snapshots judiciously, not as default

## Resources

- **Jest Snapshot Testing**: https://jestjs.io/docs/snapshot-testing
- **React Testing Library**: https://testing-library.com/docs/react-testing-library/intro
- **Playwright Screenshots**: https://playwright.dev/docs/screenshots
- **Kent C. Dodds - Snapshot Testing**: https://kentcdodds.com/blog/effective-snapshot-testing
- **Testing Library Philosophy**: https://testing-library.com/docs/guiding-principles
