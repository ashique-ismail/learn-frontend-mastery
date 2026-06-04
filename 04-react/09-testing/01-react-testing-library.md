# React Testing Library

## Overview

React Testing Library (RTL) is a lightweight testing library that encourages testing user behavior rather than implementation details. It provides utilities to query and interact with React components in a way that resembles how users interact with your application.

## Table of Contents

1. [RTL Philosophy](#rtl-philosophy)
2. [Query Hierarchy](#query-hierarchy)
3. [Screen vs Container](#screen-vs-container)
4. [User-Centric Queries](#user-centric-queries)
5. [Async Testing](#async-testing)
6. [Scoped Queries with within](#scoped-queries-with-within)
7. [Accessibility Testing](#accessibility-testing)
8. [Common Mistakes](#common-mistakes)
9. [Best Practices](#best-practices)
10. [Interview Questions](#interview-questions)

## RTL Philosophy

RTL focuses on testing behavior, not implementation details.

### Test Behavior, Not Implementation

```typescript
// ❌ BAD: Testing implementation details
import { render } from '@testing-library/react';

function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>Count: {count}</button>;
}

test('counter increments (BAD)', () => {
  const { container } = render(<Counter />);
  const button = container.querySelector('button');
  
  // Testing internal state
  expect(button.textContent).toBe('Count: 0');
  
  // Using DOM methods directly
  button.click();
  
  expect(button.textContent).toBe('Count: 1');
});

// ✅ GOOD: Testing user behavior
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

test('counter increments (GOOD)', async () => {
  const user = userEvent.setup();
  render(<Counter />);
  
  const button = screen.getByRole('button', { name: /count: 0/i });
  
  await user.click(button);
  
  expect(screen.getByRole('button', { name: /count: 1/i })).toBeInTheDocument();
});
```

### Query by User-Visible Text

```typescript
// ❌ BAD: Querying by test IDs or classes
test('shows welcome message (BAD)', () => {
  render(<HomePage />);
  expect(screen.getByTestId('welcome-message')).toBeInTheDocument();
  expect(document.querySelector('.welcome')).toHaveTextContent('Welcome');
});

// ✅ GOOD: Querying by visible text
test('shows welcome message (GOOD)', () => {
  render(<HomePage />);
  expect(screen.getByText('Welcome')).toBeInTheDocument();
  expect(screen.getByRole('heading', { name: 'Welcome' })).toBeInTheDocument();
});
```

## Query Hierarchy

RTL provides three types of queries: getBy, queryBy, and findBy.

### getBy (Sync, Throws)

```typescript
import { render, screen } from '@testing-library/react';

function UserProfile({ name }: { name: string }) {
  return <h1>Welcome, {name}</h1>;
}

test('getBy throws if not found', () => {
  render(<UserProfile name="John" />);
  
  // ✅ Element exists - returns element
  const heading = screen.getByRole('heading', { name: /welcome, john/i });
  expect(heading).toBeInTheDocument();
  
  // ❌ Element doesn't exist - throws error
  expect(() => {
    screen.getByText('Goodbye');
  }).toThrow();
});

test('getAllBy returns array', () => {
  render(
    <ul>
      <li>Item 1</li>
      <li>Item 2</li>
      <li>Item 3</li>
    </ul>
  );
  
  const items = screen.getAllByRole('listitem');
  expect(items).toHaveLength(3);
});
```

### queryBy (Sync, Returns Null)

```typescript
import { render, screen } from '@testing-library/react';

function ConditionalMessage({ show }: { show: boolean }) {
  return (
    <div>
      {show && <p>Visible message</p>}
      <p>Always visible</p>
    </div>
  );
}

test('queryBy returns null if not found', () => {
  render(<ConditionalMessage show={false} />);
  
  // ✅ Use queryBy to check absence
  expect(screen.queryByText('Visible message')).not.toBeInTheDocument();
  
  // ❌ Don't use getBy for absence checks
  // expect(screen.getByText('Visible message')).not.toBeInTheDocument(); // Throws!
});

test('queryBy vs getBy for conditional rendering', () => {
  const { rerender } = render(<ConditionalMessage show={false} />);
  
  // Not visible initially
  expect(screen.queryByText('Visible message')).not.toBeInTheDocument();
  
  // Rerender with show=true
  rerender(<ConditionalMessage show={true} />);
  
  // Now visible
  expect(screen.getByText('Visible message')).toBeInTheDocument();
});
```

### findBy (Async, Returns Promise)

```typescript
import { render, screen, waitFor } from '@testing-library/react';
import { useState, useEffect } from 'react';

function AsyncData() {
  const [data, setData] = useState<string | null>(null);
  
  useEffect(() => {
    setTimeout(() => {
      setData('Loaded data');
    }, 1000);
  }, []);
  
  return <div>{data ? <p>{data}</p> : <p>Loading...</p>}</div>;
}

test('findBy waits for element to appear', async () => {
  render(<AsyncData />);
  
  // Initially shows loading
  expect(screen.getByText('Loading...')).toBeInTheDocument();
  
  // ✅ findBy waits for element (default timeout: 1000ms)
  const dataElement = await screen.findByText('Loaded data');
  expect(dataElement).toBeInTheDocument();
  
  // Loading message is gone
  expect(screen.queryByText('Loading...')).not.toBeInTheDocument();
});

test('findBy with custom timeout', async () => {
  render(<AsyncData />);
  
  // Wait up to 2 seconds
  const dataElement = await screen.findByText('Loaded data', {}, { timeout: 2000 });
  expect(dataElement).toBeInTheDocument();
});

test('findAllBy returns array', async () => {
  function AsyncList() {
    const [items, setItems] = useState<string[]>([]);
    
    useEffect(() => {
      setTimeout(() => {
        setItems(['Item 1', 'Item 2', 'Item 3']);
      }, 500);
    }, []);
    
    return (
      <ul>
        {items.map(item => (
          <li key={item}>{item}</li>
        ))}
      </ul>
    );
  }
  
  render(<AsyncList />);
  
  const items = await screen.findAllByRole('listitem');
  expect(items).toHaveLength(3);
});
```

## Screen vs Container

### Use screen (Recommended)

```typescript
import { render, screen } from '@testing-library/react';

test('using screen', () => {
  render(<button>Click me</button>);
  
  // ✅ Use screen - cleaner, better errors
  const button = screen.getByRole('button', { name: 'Click me' });
  expect(button).toBeInTheDocument();
});
```

### When to Use Container

```typescript
import { render } from '@testing-library/react';

test('using container for specific cases', () => {
  const { container } = render(
    <div>
      <span>Text</span>
    </div>
  );
  
  // Use container for:
  // 1. snapshot testing
  expect(container).toMatchSnapshot();
  
  // 2. checking document structure
  expect(container.firstChild).toHaveClass('wrapper');
  
  // 3. querySelector when no other option
  const span = container.querySelector('span');
  expect(span).toHaveTextContent('Text');
});
```

## User-Centric Queries

### getByRole (Highest Priority)

```typescript
import { render, screen } from '@testing-library/react';

function Form() {
  return (
    <form>
      <label htmlFor="name">Name</label>
      <input id="name" type="text" />
      
      <button type="submit">Submit</button>
      
      <nav>
        <a href="/about">About</a>
      </nav>
    </form>
  );
}

test('getByRole queries', () => {
  render(<Form />);
  
  // Query by role
  screen.getByRole('textbox', { name: 'Name' });
  screen.getByRole('button', { name: 'Submit' });
  screen.getByRole('navigation');
  screen.getByRole('link', { name: 'About' });
  
  // Check all roles
  screen.debug();
});
```

### getByLabelText (Forms)

```typescript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

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

test('form interactions', async () => {
  const user = userEvent.setup();
  render(<LoginForm />);
  
  // Query by label text
  const emailInput = screen.getByLabelText('Email');
  const passwordInput = screen.getByLabelText('Password');
  
  // Type into inputs
  await user.type(emailInput, 'user@example.com');
  await user.type(passwordInput, 'password123');
  
  expect(emailInput).toHaveValue('user@example.com');
  expect(passwordInput).toHaveValue('password123');
  
  // Click submit
  await user.click(screen.getByRole('button', { name: 'Login' }));
});
```

### getByText

```typescript
import { render, screen } from '@testing-library/react';

function Article() {
  return (
    <article>
      <h1>Article Title</h1>
      <p>This is the article content.</p>
      <button>Read More</button>
    </article>
  );
}

test('getByText queries', () => {
  render(<Article />);
  
  // Exact match
  screen.getByText('Article Title');
  
  // Substring match
  screen.getByText('article content', { exact: false });
  
  // Regex match
  screen.getByText(/article/i);
  
  // Function match
  screen.getByText((content, element) => {
    return element?.tagName.toLowerCase() === 'p' && content.includes('article');
  });
});
```

### getByPlaceholderText

```typescript
test('search input', async () => {
  const user = userEvent.setup();
  
  render(
    <input type="search" placeholder="Search products..." />
  );
  
  const searchInput = screen.getByPlaceholderText('Search products...');
  await user.type(searchInput, 'laptop');
  
  expect(searchInput).toHaveValue('laptop');
});
```

### getByAltText (Images)

```typescript
test('product image', () => {
  render(
    <img src="/product.jpg" alt="Product name" />
  );
  
  const image = screen.getByAltText('Product name');
  expect(image).toHaveAttribute('src', '/product.jpg');
});
```

### getByTestId (Last Resort)

```typescript
// Use only when other queries don't work
function ComplexComponent() {
  return (
    <div data-testid="complex-component">
      <div>Complex content without good accessibility markers</div>
    </div>
  );
}

test('complex component', () => {
  render(<ComplexComponent />);
  
  // Last resort
  const component = screen.getByTestId('complex-component');
  expect(component).toBeInTheDocument();
});
```

## Async Testing

### waitFor

```typescript
import { render, screen, waitFor } from '@testing-library/react';

function SearchResults({ query }: { query: string }) {
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);
  
  useEffect(() => {
    if (query) {
      setLoading(true);
      fetch(`/api/search?q=${query}`)
        .then(res => res.json())
        .then(data => {
          setResults(data);
          setLoading(false);
        });
    }
  }, [query]);
  
  if (loading) return <div>Loading...</div>;
  
  return (
    <ul>
      {results.map((result: any) => (
        <li key={result.id}>{result.title}</li>
      ))}
    </ul>
  );
}

test('shows search results', async () => {
  // Mock fetch
  global.fetch = jest.fn(() =>
    Promise.resolve({
      json: () => Promise.resolve([
        { id: 1, title: 'Result 1' },
        { id: 2, title: 'Result 2' },
      ]),
    })
  ) as jest.Mock;
  
  render(<SearchResults query="test" />);
  
  // Wait for loading to disappear
  await waitFor(() => {
    expect(screen.queryByText('Loading...')).not.toBeInTheDocument();
  });
  
  // Results appear
  expect(screen.getByText('Result 1')).toBeInTheDocument();
  expect(screen.getByText('Result 2')).toBeInTheDocument();
});

test('waitFor with timeout', async () => {
  render(<SearchResults query="test" />);
  
  await waitFor(
    () => {
      expect(screen.getByText('Result 1')).toBeInTheDocument();
    },
    { timeout: 3000 } // Wait up to 3 seconds
  );
});
```

### waitForElementToBeRemoved

```typescript
import { render, screen, waitForElementToBeRemoved } from '@testing-library/react';

test('loading spinner disappears', async () => {
  render(<SearchResults query="test" />);
  
  const loadingElement = screen.getByText('Loading...');
  
  // Wait for element to be removed from DOM
  await waitForElementToBeRemoved(loadingElement);
  
  expect(screen.getByText('Result 1')).toBeInTheDocument();
});
```

## Scoped Queries with within

```typescript
import { render, screen, within } from '@testing-library/react';

function UserList() {
  return (
    <div>
      <div data-testid="user-1">
        <h2>John Doe</h2>
        <button>Edit</button>
        <button>Delete</button>
      </div>
      <div data-testid="user-2">
        <h2>Jane Smith</h2>
        <button>Edit</button>
        <button>Delete</button>
      </div>
    </div>
  );
}

test('scoped queries with within', () => {
  render(<UserList />);
  
  // Get specific user container
  const user1 = screen.getByTestId('user-1');
  
  // Query within that container
  within(user1).getByRole('heading', { name: 'John Doe' });
  within(user1).getByRole('button', { name: 'Edit' });
  within(user1).getByRole('button', { name: 'Delete' });
  
  const user2 = screen.getByTestId('user-2');
  within(user2).getByRole('heading', { name: 'Jane Smith' });
});

test('within with complex structures', () => {
  render(
    <table>
      <thead>
        <tr>
          <th>Name</th>
          <th>Actions</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>John</td>
          <td><button>Edit</button></td>
        </tr>
        <tr>
          <td>Jane</td>
          <td><button>Edit</button></td>
        </tr>
      </tbody>
    </table>
  );
  
  const rows = screen.getAllByRole('row');
  
  // Skip header row
  const johnRow = rows[1];
  within(johnRow).getByText('John');
  within(johnRow).getByRole('button', { name: 'Edit' });
});
```

## Accessibility Testing

### toHaveAccessibleName

```typescript
import { render, screen } from '@testing-library/react';

test('button has accessible name', () => {
  render(<button aria-label="Close dialog">X</button>);
  
  const button = screen.getByRole('button', { name: 'Close dialog' });
  expect(button).toHaveAccessibleName('Close dialog');
});

test('input has accessible name from label', () => {
  render(
    <>
      <label htmlFor="email">Email Address</label>
      <input id="email" type="email" />
    </>
  );
  
  const input = screen.getByRole('textbox', { name: 'Email Address' });
  expect(input).toHaveAccessibleName('Email Address');
});
```

### toHaveAccessibleDescription

```typescript
test('input has accessible description', () => {
  render(
    <>
      <label htmlFor="password">Password</label>
      <input id="password" type="password" aria-describedby="password-hint" />
      <span id="password-hint">Must be at least 8 characters</span>
    </>
  );
  
  const input = screen.getByLabelText('Password');
  expect(input).toHaveAccessibleDescription('Must be at least 8 characters');
});
```

### Testing with axe-core

```typescript
import { render } from '@testing-library/react';
import { axe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

test('no accessibility violations', async () => {
  const { container } = render(
    <form>
      <label htmlFor="name">Name</label>
      <input id="name" type="text" />
      <button type="submit">Submit</button>
    </form>
  );
  
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});

test('catches accessibility violations', async () => {
  const { container } = render(
    <form>
      {/* Missing label */}
      <input type="text" />
      <button type="submit">Submit</button>
    </form>
  );
  
  const results = await axe(container);
  expect(results.violations.length).toBeGreaterThan(0);
});
```

## Common Mistakes

### 1. Using getBy for Absence Checks

```typescript
// ❌ WRONG: getBy throws if not found
expect(() => {
  screen.getByText('Not here');
}).toThrow();

// ✅ CORRECT: Use queryBy
expect(screen.queryByText('Not here')).not.toBeInTheDocument();
```

### 2. Not Awaiting Async Queries

```typescript
// ❌ WRONG: Not awaiting
test('async data', () => {
  render(<AsyncComponent />);
  screen.findByText('Loaded data'); // Missing await!
});

// ✅ CORRECT: Await async queries
test('async data', async () => {
  render(<AsyncComponent />);
  await screen.findByText('Loaded data');
});
```

### 3. Querying by Implementation Details

```typescript
// ❌ WRONG: Querying by class/test ID
screen.getByTestId('submit-button');
document.querySelector('.btn-primary');

// ✅ CORRECT: Query by accessible role
screen.getByRole('button', { name: 'Submit' });
```

## Best Practices

### 1. Use Query Priority

```typescript
// 1. getByRole (most accessible)
screen.getByRole('button', { name: 'Submit' });

// 2. getByLabelText (forms)
screen.getByLabelText('Email');

// 3. getByPlaceholderText
screen.getByPlaceholderText('Search...');

// 4. getByText
screen.getByText('Welcome');

// 5. getByAltText (images)
screen.getByAltText('Product');

// 6. getByTestId (last resort)
screen.getByTestId('complex-component');
```

### 2. Use findBy for Async

```typescript
// ✅ GOOD: Use findBy
const element = await screen.findByText('Loaded');

// ❌ AVOID: waitFor + getBy
await waitFor(() => {
  expect(screen.getByText('Loaded')).toBeInTheDocument();
});
```

### 3. Use within for Scoped Queries

```typescript
const row = screen.getByTestId('user-row-1');
within(row).getByRole('button', { name: 'Edit' });
```

## Interview Questions

### Q1: What's the difference between getBy, queryBy, and findBy?

**Answer:** getBy is synchronous and throws if not found (use for assertions). queryBy is synchronous and returns null if not found (use for absence checks). findBy is async and returns a promise (use for elements that appear asynchronously).

### Q2: When should you use screen vs container?

**Answer:** Use screen for most queries as it provides better error messages and cleaner code. Use container for snapshot testing, checking document structure, or when you absolutely need querySelector.

### Q3: What is the query priority in RTL?

**Answer:** getByRole > getByLabelText > getByPlaceholderText > getByText > getByAltText > getByTestId. Always prefer queries that match how users interact with your app.

### Q4: How do you test accessibility?

**Answer:** Use getByRole queries, toHaveAccessibleName/toHaveAccessibleDescription matchers, and integrate jest-axe for automated accessibility testing with the axe-core engine.

### Q5: What's the purpose of within?

**Answer:** within creates a scoped query context for a specific DOM subtree, useful for querying elements within a specific container like table rows or list items with similar content.

## Key Takeaways

1. Test behavior, not implementation details
2. getBy throws, queryBy returns null, findBy returns promise
3. Prefer screen over container for cleaner tests
4. Use getByRole for most queries (most accessible)
5. Use queryBy to assert absence of elements
6. Use findBy for async elements that appear later
7. waitFor for complex async assertions
8. within for scoped queries in complex structures
9. Test accessibility with axe-core
10. Follow query priority: role > label > text > testId

## Resources

- [React Testing Library Docs](https://testing-library.com/react)
- [Query Priority Guide](https://testing-library.com/docs/queries/about#priority)
- [Common Mistakes](https://kentcdodds.com/blog/common-mistakes-with-react-testing-library)
- [jest-axe](https://github.com/nickcolley/jest-axe)
