# React Testing Interview Questions

## Overview

Testing questions assess your ability to write maintainable, reliable tests for React applications. Knowledge of React Testing Library, Jest, and testing best practices is essential for all levels.

---

## Core Testing Questions

### Q1: What's the difference between React Testing Library and Enzyme? Why is React Testing Library preferred?

**Answer:**

React Testing Library and Enzyme represent different testing philosophies.

**React Testing Library Philosophy:**

Test how users interact with your app, not implementation details.

```javascript
// React Testing Library approach
import { render, screen, fireEvent } from '@testing-library/react';

function Counter() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}

test('increments count when button clicked', () => {
  render(<Counter />);
  
  // Find elements like a user would
  const button = screen.getByRole('button', { name: /increment/i });
  const count = screen.getByText(/count:/i);
  
  expect(count).toHaveTextContent('Count: 0');
  
  // Interact like a user would
  fireEvent.click(button);
  
  expect(count).toHaveTextContent('Count: 1');
});
```

**Enzyme Approach (Deprecated):**

Test implementation details (state, props, methods).

```javascript
// Enzyme approach (avoid)
import { shallow } from 'enzyme';

test('increments count state', () => {
  const wrapper = shallow(<Counter />);
  
  // Tests implementation (state)
  expect(wrapper.state('count')).toBe(0);
  
  // Simulate DOM event
  wrapper.find('button').simulate('click');
  
  // Tests state again
  expect(wrapper.state('count')).toBe(1);
});
```

**Key Differences:**

| Aspect | React Testing Library | Enzyme |
|--------|----------------------|---------|
| Philosophy | User-centric | Implementation-centric |
| State access | No (by design) | Yes |
| Shallow rendering | No | Yes |
| DOM | Real DOM (jsdom) | Simulated |
| Maintenance | High confidence | Brittle tests |
| Refactoring | Tests still pass | Tests break |

**Why RTL is Better:**

```javascript
// Component refactored from class to hooks
// RTL test: STILL PASSES ✓
// Enzyme test: BREAKS ✗ (no more state to test)

// Before: Class component
class Counter extends React.Component {
  state = { count: 0 };
  
  increment = () => {
    this.setState({ count: this.state.count + 1 });
  };
  
  render() {
    return (
      <div>
        <p>Count: {this.state.count}</p>
        <button onClick={this.increment}>Increment</button>
      </div>
    );
  }
}

// After: Hooks
function Counter() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}

// RTL test works for BOTH versions!
test('increments count', () => {
  render(<Counter />);
  fireEvent.click(screen.getByRole('button'));
  expect(screen.getByText(/count: 1/i)).toBeInTheDocument();
});
```

**What Interviewers Look For:**
- Understanding of testing philosophy
- Preference for user-centric tests
- Awareness of implementation testing pitfalls
- Knowledge of modern testing practices

---

### Q2: How do you test async behavior in React components?

**Answer:**

Testing async operations requires waiting for updates and handling promises.

**Pattern 1: waitFor (Most Common):**

```javascript
import { render, screen, waitFor } from '@testing-library/react';

function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    fetchUser(userId).then(data => {
      setUser(data);
      setLoading(false);
    });
  }, [userId]);
  
  if (loading) return <div>Loading...</div>;
  return <div>{user.name}</div>;
}

test('displays user data after loading', async () => {
  // Mock API
  global.fetch = jest.fn(() =>
    Promise.resolve({
      json: () => Promise.resolve({ name: 'Alice' })
    })
  );
  
  render(<UserProfile userId={1} />);
  
  // Initially shows loading
  expect(screen.getByText(/loading/i)).toBeInTheDocument();
  
  // Wait for loading to disappear
  await waitFor(() => {
    expect(screen.queryByText(/loading/i)).not.toBeInTheDocument();
  });
  
  // Then shows user data
  expect(screen.getByText('Alice')).toBeInTheDocument();
});
```

**Pattern 2: findBy (Recommended):**

```javascript
test('displays user data', async () => {
  global.fetch = jest.fn(() =>
    Promise.resolve({
      json: () => Promise.resolve({ name: 'Alice' })
    })
  );
  
  render(<UserProfile userId={1} />);
  
  // findBy automatically waits (combines getBy + waitFor)
  const userName = await screen.findByText('Alice');
  expect(userName).toBeInTheDocument();
});
```

**Pattern 3: User Events (More Realistic):**

```javascript
import userEvent from '@testing-library/user-event';

function SearchComponent() {
  const [results, setResults] = useState([]);
  
  const handleSearch = async (query) => {
    const data = await searchAPI(query);
    setResults(data);
  };
  
  return (
    <div>
      <input
        placeholder="Search"
        onChange={(e) => handleSearch(e.target.value)}
      />
      <ul>
        {results.map(r => <li key={r.id}>{r.name}</li>)}
      </ul>
    </div>
  );
}

test('shows search results', async () => {
  const user = userEvent.setup();
  
  // Mock API
  global.searchAPI = jest.fn(() =>
    Promise.resolve([
      { id: 1, name: 'Result 1' },
      { id: 2, name: 'Result 2' }
    ])
  );
  
  render(<SearchComponent />);
  
  // Type like a real user
  await user.type(screen.getByPlaceholderText('Search'), 'test query');
  
  // Wait for results
  expect(await screen.findByText('Result 1')).toBeInTheDocument();
  expect(await screen.findByText('Result 2')).toBeInTheDocument();
});
```

**Testing Error States:**

```javascript
test('displays error message on fetch failure', async () => {
  // Mock failed fetch
  global.fetch = jest.fn(() =>
    Promise.reject(new Error('API Error'))
  );
  
  render(<UserProfile userId={1} />);
  
  // Wait for error message
  const errorMessage = await screen.findByText(/error/i);
  expect(errorMessage).toBeInTheDocument();
});
```

**Mocking Fetch with MSW (Better Approach):**

```javascript
import { rest } from 'msw';
import { setupServer } from 'msw/node';

// Setup mock server
const server = setupServer(
  rest.get('/api/users/:id', (req, res, ctx) => {
    return res(
      ctx.json({ id: 1, name: 'Alice' })
    );
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test('displays user data', async () => {
  render(<UserProfile userId={1} />);
  
  expect(await screen.findByText('Alice')).toBeInTheDocument();
});

test('handles error', async () => {
  // Override for this test
  server.use(
    rest.get('/api/users/:id', (req, res, ctx) => {
      return res(ctx.status(500));
    })
  );
  
  render(<UserProfile userId={1} />);
  
  expect(await screen.findByText(/error/i)).toBeInTheDocument();
});
```

**What Interviewers Look For:**
- Using findBy over getBy + waitFor
- Proper async/await usage
- Mocking fetch requests
- Testing loading and error states
- Knowledge of MSW

---

### Q3: How do you test components that use Context?

**Answer:**

Context components need providers in tests. Several patterns work.

**Pattern 1: Wrap with Provider:**

```javascript
import { render, screen } from '@testing-library/react';

const ThemeContext = createContext();

function ThemedButton() {
  const theme = useContext(ThemeContext);
  return <button className={theme}>Click me</button>;
}

test('applies theme from context', () => {
  render(
    <ThemeContext.Provider value="dark">
      <ThemedButton />
    </ThemeContext.Provider>
  );
  
  const button = screen.getByRole('button');
  expect(button).toHaveClass('dark');
});
```

**Pattern 2: Custom Render Function:**

```javascript
// test-utils.js
import { render } from '@testing-library/react';

function customRender(ui, { theme = 'light', ...options } = {}) {
  function Wrapper({ children }) {
    return (
      <ThemeContext.Provider value={theme}>
        {children}
      </ThemeContext.Provider>
    );
  }
  
  return render(ui, { wrapper: Wrapper, ...options });
}

// Re-export everything
export * from '@testing-library/react';
export { customRender as render };

// test file
import { render, screen } from './test-utils';

test('applies theme', () => {
  render(<ThemedButton />, { theme: 'dark' });
  
  expect(screen.getByRole('button')).toHaveClass('dark');
});
```

**Pattern 3: Multiple Providers:**

```javascript
// test-utils.js
function AllTheProviders({ children }) {
  return (
    <AuthProvider>
      <ThemeProvider>
        <LocaleProvider>
          {children}
        </LocaleProvider>
      </ThemeProvider>
    </AuthProvider>
  );
}

function customRender(ui, options) {
  return render(ui, { wrapper: AllTheProviders, ...options });
}
```

**Testing Context Updates:**

```javascript
function ThemeToggle() {
  const { theme, setTheme } = useContext(ThemeContext);
  
  return (
    <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
      Current: {theme}
    </button>
  );
}

test('toggles theme', () => {
  const { rerender } = render(
    <ThemeContext.Provider value={{ theme: 'light', setTheme: jest.fn() }}>
      <ThemeToggle />
    </ThemeContext.Provider>
  );
  
  expect(screen.getByText(/current: light/i)).toBeInTheDocument();
  
  fireEvent.click(screen.getByRole('button'));
  
  // Verify setTheme was called
  expect(mockSetTheme).toHaveBeenCalledWith('dark');
});
```

**What Interviewers Look For:**
- Creating test utilities
- Wrapping with providers
- Testing context updates
- Mocking context values

---

### Q4: How do you test custom hooks?

**Answer:**

Custom hooks need a component to run in. Use `renderHook` from React Testing Library.

**Basic Hook Testing:**

```javascript
import { renderHook, act } from '@testing-library/react';

function useCounter(initialValue = 0) {
  const [count, setCount] = useState(initialValue);
  
  const increment = () => setCount(c => c + 1);
  const decrement = () => setCount(c => c - 1);
  
  return { count, increment, decrement };
}

test('increments count', () => {
  const { result } = renderHook(() => useCounter());
  
  expect(result.current.count).toBe(0);
  
  act(() => {
    result.current.increment();
  });
  
  expect(result.current.count).toBe(1);
});

test('starts with initial value', () => {
  const { result } = renderHook(() => useCounter(10));
  
  expect(result.current.count).toBe(10);
});
```

**Testing Hook with Dependencies:**

```javascript
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    fetch(url)
      .then(r => r.json())
      .then(setData)
      .finally(() => setLoading(false));
  }, [url]);
  
  return { data, loading };
}

test('fetches data', async () => {
  global.fetch = jest.fn(() =>
    Promise.resolve({
      json: () => Promise.resolve({ name: 'Alice' })
    })
  );
  
  const { result } = renderHook(() => useFetch('/api/user'));
  
  expect(result.current.loading).toBe(true);
  
  await waitFor(() => {
    expect(result.current.loading).toBe(false);
  });
  
  expect(result.current.data).toEqual({ name: 'Alice' });
});
```

**Testing Hook Re-renders:**

```javascript
test('refetches when URL changes', async () => {
  const { result, rerender } = renderHook(
    ({ url }) => useFetch(url),
    { initialProps: { url: '/api/users/1' } }
  );
  
  await waitFor(() => expect(result.current.data).toBeTruthy());
  
  const firstData = result.current.data;
  
  // Change URL
  rerender({ url: '/api/users/2' });
  
  await waitFor(() => {
    expect(result.current.data).not.toEqual(firstData);
  });
});
```

**Testing Hook with Context:**

```javascript
function useAuth() {
  const context = useContext(AuthContext);
  if (!context) throw new Error('useAuth must be within AuthProvider');
  return context;
}

test('returns auth context', () => {
  const wrapper = ({ children }) => (
    <AuthContext.Provider value={{ user: { name: 'Alice' } }}>
      {children}
    </AuthContext.Provider>
  );
  
  const { result } = renderHook(() => useAuth(), { wrapper });
  
  expect(result.current.user.name).toBe('Alice');
});
```

**What Interviewers Look For:**
- Using renderHook correctly
- Wrapping act() around state updates
- Testing with different props
- Handling async hooks

---

### Q5: What are the key testing best practices?

**Answer:**

**1. Test User Behavior, Not Implementation:**

```javascript
// ❌ BAD: Tests implementation
test('updates state correctly', () => {
  const { result } = renderHook(() => useCounter());
  act(() => result.current.increment());
  expect(result.current.count).toBe(1);
});

// ✅ GOOD: Tests user behavior
test('displays incremented count', () => {
  render(<Counter />);
  fireEvent.click(screen.getByRole('button', { name: /increment/i }));
  expect(screen.getByText(/count: 1/i)).toBeInTheDocument();
});
```

**2. Use Accessible Queries:**

```javascript
// Priority order:
// 1. getByRole (BEST)
screen.getByRole('button', { name: /submit/i });

// 2. getByLabelText (forms)
screen.getByLabelText(/email/i);

// 3. getByPlaceholderText
screen.getByPlaceholderText(/enter email/i);

// 4. getByText
screen.getByText(/welcome/i);

// 5. getByTestId (LAST RESORT)
screen.getByTestId('submit-button');
```

**3. Avoid Implementation Details:**

```javascript
// ❌ BAD: Relies on CSS classes
expect(button).toHaveClass('btn-primary');

// ✅ GOOD: Tests behavior
expect(button).toBeEnabled();
expect(button).toHaveTextContent('Submit');
```

**4. Use User Event over FireEvent:**

```javascript
import userEvent from '@testing-library/user-event';

// ❌ Less realistic
fireEvent.change(input, { target: { value: 'test' } });

// ✅ More realistic (fires all events a real user would)
const user = userEvent.setup();
await user.type(input, 'test');
```

**5. Test Edge Cases:**

```javascript
describe('SearchInput', () => {
  test('handles empty input', () => {
    render(<SearchInput />);
    expect(screen.getByRole('textbox')).toHaveValue('');
  });
  
  test('handles special characters', async () => {
    const user = userEvent.setup();
    render(<SearchInput />);
    await user.type(screen.getByRole('textbox'), '!@#$%');
    expect(screen.getByRole('textbox')).toHaveValue('!@#$%');
  });
  
  test('handles very long input', async () => {
    const user = userEvent.setup();
    const longText = 'a'.repeat(1000);
    render(<SearchInput maxLength={100} />);
    await user.type(screen.getByRole('textbox'), longText);
    expect(screen.getByRole('textbox')).toHaveValue(longText.slice(0, 100));
  });
});
```

**What Interviewers Look For:**
- Following RTL principles
- Accessible query usage
- Testing user workflows
- Covering edge cases
- Clean, maintainable tests

---

## Key Takeaways

**Essential Concepts:**
1. Test user behavior, not implementation
2. Use findBy for async operations
3. Wrap components with providers in tests
4. Use renderHook for custom hooks
5. Prefer accessible queries

**Best Practices:**
- Use React Testing Library over Enzyme
- Mock API calls with MSW
- Create custom render utilities
- Use userEvent for interactions
- Test loading, success, and error states

**Common Mistakes:**
- Testing state directly
- Using queryBy when should use findBy
- Not wrapping async updates in waitFor
- Over-relying on test IDs
- Testing implementation details

**Interview Tips:**
- Explain testing philosophy
- Demonstrate async testing
- Show custom utility usage
- Discuss accessibility in tests
- Mention MSW for API mocking

**Red Flags to Avoid:**
- Still using Enzyme
- Testing component state
- Not testing async behavior
- Forgetting error states
- Using setTimeout in tests
