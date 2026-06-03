# Testing Library - React Testing Philosophy

## Overview

Testing Library is a family of packages that help test UI components in a user-centric way. The React Testing Library encourages testing components as users would interact with them, focusing on accessibility and actual behavior rather than implementation details. It's built on the principle: "The more your tests resemble the way your software is used, the more confidence they can give you."

## Installation and Setup

```bash
# Install React Testing Library
npm install -D @testing-library/react @testing-library/jest-dom

# With user event (recommended)
npm install -D @testing-library/user-event

# For React 18
npm install -D @testing-library/react@^14

# Full testing stack
npm install -D vitest @testing-library/react @testing-library/jest-dom @testing-library/user-event jsdom
```

### Setup with Vitest

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

## Core Concepts

### Rendering Components

```typescript
import { render, screen } from '@testing-library/react';
import { Button } from './Button';

describe('Button', () => {
  it('renders with text', () => {
    render(<Button>Click me</Button>);
    expect(screen.getByText('Click me')).toBeInTheDocument();
  });

  it('renders with props', () => {
    render(<Button variant="primary">Submit</Button>);
    const button = screen.getByRole('button', { name: 'Submit' });
    expect(button).toHaveClass('btn-primary');
  });

  it('renders multiple components', () => {
    const { container } = render(
      <div>
        <Button>First</Button>
        <Button>Second</Button>
      </div>
    );
    expect(container.querySelectorAll('button')).toHaveLength(2);
  });
});
```

### Query Methods

```typescript
import { render, screen } from '@testing-library/react';

describe('Query methods', () => {
  it('demonstrates getBy* queries', () => {
    render(
      <div>
        <button>Submit</button>
        <input aria-label="Email" />
        <span data-testid="status">Active</span>
      </div>
    );

    // getBy* - throws error if not found
    expect(screen.getByRole('button', { name: 'Submit' })).toBeInTheDocument();
    expect(screen.getByLabelText('Email')).toBeInTheDocument();
    expect(screen.getByTestId('status')).toHaveTextContent('Active');
    expect(screen.getByText('Submit')).toBeInTheDocument();
  });

  it('demonstrates queryBy* queries', () => {
    render(<div><button>Submit</button></div>);

    // queryBy* - returns null if not found (doesn't throw)
    expect(screen.queryByText('Submit')).toBeInTheDocument();
    expect(screen.queryByText('Cancel')).not.toBeInTheDocument();
    expect(screen.queryByText('Cancel')).toBeNull();
  });

  it('demonstrates findBy* queries', async () => {
    const AsyncComponent = () => {
      const [show, setShow] = React.useState(false);
      React.useEffect(() => {
        setTimeout(() => setShow(true), 100);
      }, []);
      return show ? <div>Loaded</div> : <div>Loading...</div>;
    };

    render(<AsyncComponent />);

    // findBy* - returns promise, waits for element
    expect(await screen.findByText('Loaded')).toBeInTheDocument();
  });

  it('demonstrates getAllBy* queries', () => {
    render(
      <ul>
        <li>Item 1</li>
        <li>Item 2</li>
        <li>Item 3</li>
      </ul>
    );

    // getAllBy* - returns array of elements
    const items = screen.getAllByRole('listitem');
    expect(items).toHaveLength(3);
    expect(items[0]).toHaveTextContent('Item 1');
  });
});
```

### Query Priority

```typescript
import { render, screen } from '@testing-library/react';

describe('Query priority', () => {
  it('uses recommended query priority', () => {
    render(
      <div>
        <button>Submit</button>
        <input aria-label="Email" placeholder="Enter email" />
        <img alt="Logo" src="logo.png" />
        <span title="Status">Active</span>
        <div data-testid="fallback">Content</div>
      </div>
    );

    // 1. getByRole (most preferred)
    screen.getByRole('button', { name: 'Submit' });
    screen.getByRole('textbox', { name: 'Email' });

    // 2. getByLabelText (forms)
    screen.getByLabelText('Email');

    // 3. getByPlaceholderText (if no label)
    screen.getByPlaceholderText('Enter email');

    // 4. getByText (non-interactive)
    screen.getByText('Active');

    // 5. getByAltText (images)
    screen.getByAltText('Logo');

    // 6. getByTitle (last resort)
    screen.getByTitle('Status');

    // 7. getByTestId (fallback only)
    screen.getByTestId('fallback');
  });
});
```

## User Interactions

### Using userEvent (Recommended)

```typescript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

describe('User interactions', () => {
  it('handles click events', async () => {
    const handleClick = vi.fn();
    const user = userEvent.setup();

    render(<button onClick={handleClick}>Click me</button>);

    await user.click(screen.getByRole('button'));

    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  it('handles typing', async () => {
    const user = userEvent.setup();

    render(<input aria-label="Email" />);

    const input = screen.getByLabelText('Email');
    await user.type(input, 'test@example.com');

    expect(input).toHaveValue('test@example.com');
  });

  it('handles keyboard navigation', async () => {
    const user = userEvent.setup();

    render(
      <div>
        <button>First</button>
        <button>Second</button>
        <button>Third</button>
      </div>
    );

    const firstButton = screen.getByRole('button', { name: 'First' });
    firstButton.focus();

    await user.keyboard('{Tab}');
    expect(screen.getByRole('button', { name: 'Second' })).toHaveFocus();

    await user.keyboard('{Tab}');
    expect(screen.getByRole('button', { name: 'Third' })).toHaveFocus();
  });

  it('handles form submission', async () => {
    const handleSubmit = vi.fn((e) => e.preventDefault());
    const user = userEvent.setup();

    render(
      <form onSubmit={handleSubmit}>
        <input aria-label="Email" />
        <button type="submit">Submit</button>
      </form>
    );

    await user.type(screen.getByLabelText('Email'), 'test@example.com');
    await user.click(screen.getByRole('button', { name: 'Submit' }));

    expect(handleSubmit).toHaveBeenCalledTimes(1);
  });

  it('handles select element', async () => {
    const user = userEvent.setup();

    render(
      <select aria-label="Country">
        <option value="us">United States</option>
        <option value="uk">United Kingdom</option>
        <option value="ca">Canada</option>
      </select>
    );

    await user.selectOptions(screen.getByLabelText('Country'), 'uk');

    expect(screen.getByRole('option', { name: 'United Kingdom' })).toBeSelected();
  });

  it('handles checkbox', async () => {
    const user = userEvent.setup();

    render(<input type="checkbox" aria-label="Accept terms" />);

    const checkbox = screen.getByLabelText('Accept terms');

    await user.click(checkbox);
    expect(checkbox).toBeChecked();

    await user.click(checkbox);
    expect(checkbox).not.toBeChecked();
  });

  it('handles file upload', async () => {
    const user = userEvent.setup();
    const file = new File(['hello'], 'hello.png', { type: 'image/png' });

    render(<input type="file" aria-label="Upload" />);

    const input = screen.getByLabelText('Upload');
    await user.upload(input, file);

    expect(input.files[0]).toBe(file);
    expect(input.files).toHaveLength(1);
  });
});
```

### Waiting for Elements

```typescript
import { render, screen, waitFor } from '@testing-library/react';

describe('Async testing', () => {
  it('waits for element to appear', async () => {
    const AsyncComponent = () => {
      const [data, setData] = React.useState(null);

      React.useEffect(() => {
        setTimeout(() => setData('Loaded'), 100);
      }, []);

      return data ? <div>{data}</div> : <div>Loading...</div>;
    };

    render(<AsyncComponent />);

    expect(screen.getByText('Loading...')).toBeInTheDocument();
    expect(await screen.findByText('Loaded')).toBeInTheDocument();
  });

  it('uses waitFor for complex conditions', async () => {
    const Component = () => {
      const [count, setCount] = React.useState(0);

      React.useEffect(() => {
        const interval = setInterval(() => {
          setCount((c) => c + 1);
        }, 100);
        return () => clearInterval(interval);
      }, []);

      return <div>Count: {count}</div>;
    };

    render(<Component />);

    await waitFor(() => {
      expect(screen.getByText(/Count: [5-9]|Count: \d{2}/)).toBeInTheDocument();
    });
  });

  it('waits for element to disappear', async () => {
    const Component = () => {
      const [show, setShow] = React.useState(true);

      React.useEffect(() => {
        setTimeout(() => setShow(false), 100);
      }, []);

      return show ? <div>Visible</div> : null;
    };

    render(<Component />);

    expect(screen.getByText('Visible')).toBeInTheDocument();

    await waitFor(() => {
      expect(screen.queryByText('Visible')).not.toBeInTheDocument();
    });
  });

  it('handles API calls', async () => {
    const fetchUser = () =>
      Promise.resolve({ id: 1, name: 'John Doe' });

    const UserComponent = () => {
      const [user, setUser] = React.useState(null);

      React.useEffect(() => {
        fetchUser().then(setUser);
      }, []);

      if (!user) return <div>Loading...</div>;
      return <div>{user.name}</div>;
    };

    render(<UserComponent />);

    expect(screen.getByText('Loading...')).toBeInTheDocument();
    expect(await screen.findByText('John Doe')).toBeInTheDocument();
  });
});
```

## Testing Patterns

### Testing Forms

```typescript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

describe('LoginForm', () => {
  it('submits form with valid data', async () => {
    const handleSubmit = vi.fn();
    const user = userEvent.setup();

    render(<LoginForm onSubmit={handleSubmit} />);

    await user.type(screen.getByLabelText(/email/i), 'test@example.com');
    await user.type(screen.getByLabelText(/password/i), 'password123');
    await user.click(screen.getByRole('button', { name: /login/i }));

    expect(handleSubmit).toHaveBeenCalledWith({
      email: 'test@example.com',
      password: 'password123',
    });
  });

  it('shows validation errors', async () => {
    const user = userEvent.setup();

    render(<LoginForm onSubmit={vi.fn()} />);

    await user.click(screen.getByRole('button', { name: /login/i }));

    expect(await screen.findByText(/email is required/i)).toBeInTheDocument();
    expect(await screen.findByText(/password is required/i)).toBeInTheDocument();
  });

  it('clears errors on input', async () => {
    const user = userEvent.setup();

    render(<LoginForm onSubmit={vi.fn()} />);

    await user.click(screen.getByRole('button', { name: /login/i }));
    expect(await screen.findByText(/email is required/i)).toBeInTheDocument();

    await user.type(screen.getByLabelText(/email/i), 'test@example.com');

    expect(screen.queryByText(/email is required/i)).not.toBeInTheDocument();
  });
});
```

### Testing with Context

```typescript
import { render, screen } from '@testing-library/react';
import { ThemeProvider } from './ThemeContext';

describe('ThemedButton', () => {
  const renderWithTheme = (component, theme = 'light') => {
    return render(
      <ThemeProvider value={theme}>
        {component}
      </ThemeProvider>
    );
  };

  it('renders with light theme', () => {
    renderWithTheme(<ThemedButton>Click</ThemedButton>, 'light');
    expect(screen.getByRole('button')).toHaveClass('btn-light');
  });

  it('renders with dark theme', () => {
    renderWithTheme(<ThemedButton>Click</ThemedButton>, 'dark');
    expect(screen.getByRole('button')).toHaveClass('btn-dark');
  });
});
```

### Testing Hooks

```typescript
import { renderHook, waitFor } from '@testing-library/react';

describe('useCounter', () => {
  it('increments counter', () => {
    const { result } = renderHook(() => useCounter());

    expect(result.current.count).toBe(0);

    act(() => {
      result.current.increment();
    });

    expect(result.current.count).toBe(1);
  });

  it('decrements counter', () => {
    const { result } = renderHook(() => useCounter(10));

    act(() => {
      result.current.decrement();
    });

    expect(result.current.count).toBe(9);
  });

  it('resets counter', () => {
    const { result } = renderHook(() => useCounter(5));

    act(() => {
      result.current.increment();
      result.current.increment();
    });

    expect(result.current.count).toBe(7);

    act(() => {
      result.current.reset();
    });

    expect(result.current.count).toBe(5);
  });
});

describe('useFetch', () => {
  it('fetches data', async () => {
    const mockFetch = vi.fn(() =>
      Promise.resolve({
        json: () => Promise.resolve({ data: 'test' }),
      })
    );
    global.fetch = mockFetch;

    const { result } = renderHook(() => useFetch('/api/data'));

    expect(result.current.loading).toBe(true);

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(result.current.data).toEqual({ data: 'test' });
    expect(mockFetch).toHaveBeenCalledWith('/api/data');
  });
});
```

## Debugging Tests

```typescript
import { render, screen, prettyDOM } from '@testing-library/react';

describe('Debugging', () => {
  it('uses debug helpers', () => {
    const { container, debug } = render(
      <div>
        <button>Submit</button>
        <span>Status</span>
      </div>
    );

    // Print entire component tree
    debug();

    // Print specific element
    debug(screen.getByRole('button'));

    // Print container
    console.log(prettyDOM(container));

    // Get suggested queries
    screen.getByRole('button'); // Logs suggestion if not found

    // Log all available queries
    screen.logTestingPlaygroundURL();
  });

  it('finds elements with screen.debug()', () => {
    render(
      <div>
        <div data-testid="parent">
          <button>Child Button</button>
        </div>
      </div>
    );

    const parent = screen.getByTestId('parent');
    // Debug only the parent element
    console.log(prettyDOM(parent));
  });
});
```

## Custom Queries

```typescript
import { render, buildQueries, within } from '@testing-library/react';

// Custom query
const queryAllByDataCy = (container, id) =>
  container.querySelectorAll(`[data-cy="${id}"]`);

const getMultipleError = (c, dataCyValue) =>
  `Found multiple elements with data-cy="${dataCyValue}"`;

const getMissingError = (c, dataCyValue) =>
  `Unable to find element with data-cy="${dataCyValue}"`;

const [
  queryByDataCy,
  getAllByDataCy,
  getByDataCy,
  findAllByDataCy,
  findByDataCy,
] = buildQueries(queryAllByDataCy, getMultipleError, getMissingError);

// Usage
describe('Custom queries', () => {
  it('uses custom data-cy query', () => {
    const { container } = render(
      <div>
        <button data-cy="submit-btn">Submit</button>
      </div>
    );

    expect(getByDataCy(container, 'submit-btn')).toBeInTheDocument();
  });
});
```

## Common Mistakes

1. **Testing implementation details**: Checking component state or internal methods instead of user-visible behavior.

2. **Using wrong queries**: Using `getByTestId` when semantic queries like `getByRole` are available.

3. **Not using userEvent**: Using `fireEvent` instead of `@testing-library/user-event`.

4. **Waiting incorrectly**: Using `waitFor` for everything instead of `findBy*` queries.

5. **Querying too early**: Querying before rendering or before elements appear.

6. **Not cleaning up**: Forgetting to cleanup between tests causing test pollution.

7. **Over-mocking**: Mocking too much, testing mocks instead of real behavior.

8. **Poor assertions**: Using `.toBeTruthy()` instead of specific matchers like `.toBeInTheDocument()`.

9. **Ignoring accessibility**: Not using proper labels and ARIA attributes.

10. **Testing internal state**: Accessing component state directly instead of testing output.

## Best Practices

1. **Query by accessibility**: Use `getByRole`, `getByLabelText` for better tests and accessible components.

2. **Use userEvent**: Prefer `@testing-library/user-event` over `fireEvent` for realistic interactions.

3. **Wait for changes**: Use `findBy*` or `waitFor` for async operations.

4. **Avoid test IDs**: Only use `data-testid` as last resort when semantic queries don't work.

5. **Test user behavior**: Focus on what users see and do, not implementation.

6. **Keep tests simple**: One logical assertion per test, clear test names.

7. **Use custom render**: Create custom render functions with common providers.

8. **Leverage matchers**: Use jest-dom matchers like `.toBeInTheDocument()`, `.toHaveValue()`.

9. **Query within elements**: Use `within()` to scope queries to specific containers.

10. **Clean setup**: Use `beforeEach` and `afterEach` for consistent test state.

## When to Use Testing Library

**Use Testing Library when:**
- Testing React components
- Want user-centric tests
- Need accessibility-focused testing
- Building maintainable test suites
- Want to avoid implementation details
- Testing user interactions
- Building component libraries

**Consider alternatives when:**
- Need E2E browser testing (use Cypress/Playwright)
- Testing non-React frameworks (use framework-specific library)
- Need visual regression testing (use Percy/Chromatic)
- Testing performance (use Lighthouse)

## Interview Questions

### Q1: What's the philosophy behind Testing Library?

**Answer**: Testing Library encourages testing components the way users interact with them, focusing on accessibility and behavior rather than implementation details. The guiding principle is: "The more your tests resemble the way your software is used, the more confidence they can give you." This means querying by accessible labels/roles, testing user interactions, and avoiding testing internal state or implementation.

### Q2: What's the difference between getBy, queryBy, and findBy?

**Answer**: `getBy*` throws error if element not found (synchronous). `queryBy*` returns null if not found, useful for asserting non-existence (synchronous). `findBy*` returns promise, waits for element to appear (asynchronous). Use `getBy` for elements that should exist, `queryBy` to check elements don't exist, `findBy` for elements that appear asynchronously.

### Q3: Why is userEvent preferred over fireEvent?

**Answer**: `userEvent` simulates real user interactions with proper event sequences. For example, `userEvent.click()` triggers mousedown, mouseup, and click events in correct order, while `fireEvent.click()` only triggers click. `userEvent.type()` fires keydown, keypress, keyup events for each character. This makes tests more reliable and closer to real user behavior.

### Q4: What's the correct query priority in Testing Library?

**Answer**: Priority order (most to least preferred):
1. `getByRole` (most accessible)
2. `getByLabelText` (forms)
3. `getByPlaceholderText` (if no label)
4. `getByText` (non-interactive)
5. `getByAltText` (images)
6. `getByTitle` (last resort)
7. `getByTestId` (fallback only)

This priority encourages accessible components and maintainable tests.

### Q5: How do you test async operations?

**Answer**: Use `findBy*` queries or `waitFor`:

```typescript
// Using findBy (preferred)
expect(await screen.findByText('Loaded')).toBeInTheDocument();

// Using waitFor
await waitFor(() => {
  expect(screen.getByText('Loaded')).toBeInTheDocument();
});
```

`findBy*` is cleaner for single element waits, `waitFor` for complex conditions.

### Q6: How do you test components with context?

**Answer**: Create custom render function wrapping providers:

```typescript
const renderWithProviders = (component, options = {}) => {
  return render(
    <ThemeProvider>
      <AuthProvider>
        {component}
      </AuthProvider>
    </ThemeProvider>,
    options
  );
};
```

This ensures components have required context in all tests.

### Q7: How do you debug failing tests in Testing Library?

**Answer**: Use debugging tools: 1) `screen.debug()` to print DOM, 2) `screen.logTestingPlaygroundURL()` for interactive playground, 3) `prettyDOM(element)` for specific elements, 4) `screen.getByRole()` without arguments to see all available roles, 5) Add `screen.debug()` at failure point to see DOM state. Testing Library's error messages also suggest correct queries.

## Key Takeaways

1. Testing Library focuses on testing components from user perspective, promoting accessible and maintainable tests.

2. Query priority favors accessible selectors (`getByRole`, `getByLabelText`) over implementation details (`getByTestId`).

3. Three query variants serve different purposes: `getBy` for assertions, `queryBy` for non-existence, `findBy` for async.

4. userEvent simulates realistic user interactions better than fireEvent with proper event sequences.

5. Waiting strategies include `findBy*` queries for elements and `waitFor` for complex conditions.

6. jest-dom matchers provide semantic assertions like `toBeInTheDocument`, `toHaveValue`, `toBeDisabled`.

7. Custom render functions wrap components with providers, reducing boilerplate and ensuring consistent setup.

8. Testing hooks uses `renderHook` utility with `act` for state updates and `waitFor` for async operations.

9. Debugging tools include `screen.debug()`, `prettyDOM()`, and `logTestingPlaygroundURL()` for troubleshooting.

10. The library encourages avoiding implementation details, testing what users see and experience instead.

## Resources

- [Official Documentation](https://testing-library.com/docs/react-testing-library/intro/)
- [Common Mistakes](https://kentcdodds.com/blog/common-mistakes-with-react-testing-library)
- [Cheatsheet](https://testing-library.com/docs/react-testing-library/cheatsheet)
- [Which Query](https://testing-library.com/docs/queries/about#priority)
- [userEvent API](https://testing-library.com/docs/user-event/intro)
- [Testing Playground](https://testing-playground.com/)
