# Testing React Hooks

## The Idea

**In plain English:** Testing React hooks means checking that the reusable logic pieces in your app (called hooks — functions that remember things or do tasks behind the scenes) behave correctly. You run them in a pretend environment, poke them with inputs, and confirm they give back the right outputs.

**Real-world analogy:** Imagine a vending machine that you test before putting it on the floor. You press buttons (call actions), check what comes out (read the result), and verify the machine cleans up after itself (confirm nothing is left jammed inside).

- The vending machine = the custom hook
- Pressing a button = calling a function the hook returns (like `increment()`)
- The item that drops out = the value the hook gives back (like `count`)
- Checking the machine is empty after your test = verifying cleanup (like `unmount()`)

---

## Overview

Testing custom hooks requires special utilities from React Testing Library. The `renderHook` function allows you to test hooks in isolation without needing to create wrapper components. This guide covers patterns for testing hooks with state, effects, context, and async operations.

## Table of Contents

1. [renderHook Basics](#renderhook-basics)
2. [Testing State Updates](#testing-state-updates)
3. [Testing with act()](#testing-with-act)
4. [Rerendering Hooks](#rerendering-hooks)
5. [Testing Async Hooks](#testing-async-hooks)
6. [Testing Hooks with Context](#testing-hooks-with-context)
7. [Testing Hooks with Dependencies](#testing-hooks-with-dependencies)
8. [Testing useEffect and Cleanup](#testing-useeffect-and-cleanup)
9. [Example: Testing useFetch](#example-testing-usefetch)
10. [Example: Testing useDebounce](#example-testing-usedebounce)
11. [Example: Testing useLocalStorage](#example-testing-uselocalstorage)
12. [Common Mistakes](#common-mistakes)
13. [Best Practices](#best-practices)
14. [Interview Questions](#interview-questions)

## renderHook Basics

The `renderHook` function from `@testing-library/react` allows you to test hooks in isolation.

### Basic Hook Testing

```typescript
import { renderHook } from '@testing-library/react';
import { useState } from 'react';

function useCounter(initialValue = 0) {
  const [count, setCount] = useState(initialValue);
  
  const increment = () => setCount(c => c + 1);
  const decrement = () => setCount(c => c - 1);
  const reset = () => setCount(initialValue);
  
  return { count, increment, decrement, reset };
}

test('useCounter initializes with default value', () => {
  const { result } = renderHook(() => useCounter());
  
  expect(result.current.count).toBe(0);
});

test('useCounter initializes with custom value', () => {
  const { result } = renderHook(() => useCounter(10));
  
  expect(result.current.count).toBe(10);
});
```

### Accessing result.current

```typescript
test('result.current provides hook return value', () => {
  const { result } = renderHook(() => useCounter(5));
  
  // result.current contains the current return value
  expect(result.current).toEqual({
    count: 5,
    increment: expect.any(Function),
    decrement: expect.any(Function),
    reset: expect.any(Function),
  });
  
  // Access individual properties
  expect(result.current.count).toBe(5);
  expect(typeof result.current.increment).toBe('function');
});
```

### Testing Hook Return Values

```typescript
function useToggle(initialValue = false) {
  const [value, setValue] = useState(initialValue);
  
  const toggle = () => setValue(v => !v);
  const setTrue = () => setValue(true);
  const setFalse = () => setValue(false);
  
  return { value, toggle, setTrue, setFalse };
}

test('useToggle returns correct structure', () => {
  const { result } = renderHook(() => useToggle());
  
  expect(result.current.value).toBe(false);
  expect(result.current).toHaveProperty('toggle');
  expect(result.current).toHaveProperty('setTrue');
  expect(result.current).toHaveProperty('setFalse');
});

test('useToggle with initial true', () => {
  const { result } = renderHook(() => useToggle(true));
  
  expect(result.current.value).toBe(true);
});
```

## Testing State Updates

State updates in hooks must be wrapped in `act()` to ensure React processes them correctly.

### Using act() for State Changes

```typescript
import { renderHook, act } from '@testing-library/react';

test('useCounter increments', () => {
  const { result } = renderHook(() => useCounter());
  
  // Wrap state updates in act()
  act(() => {
    result.current.increment();
  });
  
  expect(result.current.count).toBe(1);
});

test('useCounter decrements', () => {
  const { result } = renderHook(() => useCounter(5));
  
  act(() => {
    result.current.decrement();
  });
  
  expect(result.current.count).toBe(4);
});

test('useCounter resets', () => {
  const { result } = renderHook(() => useCounter(10));
  
  act(() => {
    result.current.increment();
    result.current.increment();
  });
  
  expect(result.current.count).toBe(12);
  
  act(() => {
    result.current.reset();
  });
  
  expect(result.current.count).toBe(10);
});
```

### Multiple State Updates

```typescript
test('multiple state updates', () => {
  const { result } = renderHook(() => useCounter());
  
  // Multiple updates in single act
  act(() => {
    result.current.increment();
    result.current.increment();
    result.current.increment();
  });
  
  expect(result.current.count).toBe(3);
  
  // Separate act calls
  act(() => {
    result.current.decrement();
  });
  
  expect(result.current.count).toBe(2);
});
```

### Testing Toggle Behavior

```typescript
test('useToggle changes state', () => {
  const { result } = renderHook(() => useToggle(false));
  
  expect(result.current.value).toBe(false);
  
  act(() => {
    result.current.toggle();
  });
  
  expect(result.current.value).toBe(true);
  
  act(() => {
    result.current.toggle();
  });
  
  expect(result.current.value).toBe(false);
});

test('useToggle setTrue and setFalse', () => {
  const { result } = renderHook(() => useToggle());
  
  act(() => {
    result.current.setTrue();
  });
  
  expect(result.current.value).toBe(true);
  
  act(() => {
    result.current.setFalse();
  });
  
  expect(result.current.value).toBe(false);
});
```

## Testing with act()

The `act()` function ensures all updates are processed before assertions.

### Why act() is Needed

```typescript
// ❌ WRONG: Not using act()
test('counter without act (BAD)', () => {
  const { result } = renderHook(() => useCounter());
  
  // This will cause warnings
  result.current.increment();
  
  // May not have updated yet
  expect(result.current.count).toBe(1);
});

// ✅ CORRECT: Using act()
test('counter with act (GOOD)', () => {
  const { result } = renderHook(() => useCounter());
  
  act(() => {
    result.current.increment();
  });
  
  expect(result.current.count).toBe(1);
});
```

### Async act()

```typescript
function useDelayedCounter() {
  const [count, setCount] = useState(0);
  
  const incrementAsync = () => {
    setTimeout(() => {
      setCount(c => c + 1);
    }, 100);
  };
  
  return { count, incrementAsync };
}

test('async state updates', async () => {
  const { result } = renderHook(() => useDelayedCounter());
  
  // Async act for delayed updates
  await act(async () => {
    result.current.incrementAsync();
    await new Promise(resolve => setTimeout(resolve, 150));
  });
  
  expect(result.current.count).toBe(1);
});
```

## Rerendering Hooks

The `rerender` function allows you to test hooks with changing props.

### Testing Hook Props Changes

```typescript
function useGreeting(name: string) {
  return `Hello, ${name}!`;
}

test('hook responds to prop changes', () => {
  const { result, rerender } = renderHook(
    ({ name }) => useGreeting(name),
    { initialProps: { name: 'Alice' } }
  );
  
  expect(result.current).toBe('Hello, Alice!');
  
  // Rerender with new props
  rerender({ name: 'Bob' });
  
  expect(result.current).toBe('Hello, Bob!');
});
```

### Testing useEffect with Prop Changes

```typescript
function useDocumentTitle(title: string) {
  useEffect(() => {
    document.title = title;
  }, [title]);
}

test('updates document title on prop change', () => {
  const { rerender } = renderHook(
    ({ title }) => useDocumentTitle(title),
    { initialProps: { title: 'Initial Title' } }
  );
  
  expect(document.title).toBe('Initial Title');
  
  rerender({ title: 'New Title' });
  
  expect(document.title).toBe('New Title');
});
```

### Testing Dependencies

```typescript
function useFilteredList<T>(items: T[], filter: string) {
  const [filtered, setFiltered] = useState<T[]>([]);
  
  useEffect(() => {
    setFiltered(
      items.filter(item => 
        String(item).toLowerCase().includes(filter.toLowerCase())
      )
    );
  }, [items, filter]);
  
  return filtered;
}

test('refilters on items change', () => {
  const { result, rerender } = renderHook(
    ({ items, filter }) => useFilteredList(items, filter),
    { initialProps: { items: ['apple', 'banana', 'cherry'], filter: '' } }
  );
  
  expect(result.current).toEqual(['apple', 'banana', 'cherry']);
  
  // Change filter
  rerender({ items: ['apple', 'banana', 'cherry'], filter: 'a' });
  
  expect(result.current).toEqual(['apple', 'banana']);
  
  // Change items
  rerender({ items: ['avocado', 'banana', 'cherry'], filter: 'a' });
  
  expect(result.current).toEqual(['avocado', 'banana']);
});
```

## Testing Async Hooks

Async hooks require `waitFor` or `waitForNextUpdate` patterns.

### Testing with waitFor

```typescript
import { renderHook, waitFor } from '@testing-library/react';

function useAsyncData(url: string) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  
  useEffect(() => {
    let cancelled = false;
    
    fetch(url)
      .then(res => res.json())
      .then(data => {
        if (!cancelled) {
          setData(data);
          setLoading(false);
        }
      })
      .catch(err => {
        if (!cancelled) {
          setError(err);
          setLoading(false);
        }
      });
    
    return () => {
      cancelled = true;
    };
  }, [url]);
  
  return { data, loading, error };
}

test('loads data successfully', async () => {
  global.fetch = jest.fn(() =>
    Promise.resolve({
      json: () => Promise.resolve({ id: 1, name: 'Test' }),
    })
  ) as jest.Mock;
  
  const { result } = renderHook(() => useAsyncData('/api/data'));
  
  expect(result.current.loading).toBe(true);
  expect(result.current.data).toBe(null);
  
  await waitFor(() => {
    expect(result.current.loading).toBe(false);
  });
  
  expect(result.current.data).toEqual({ id: 1, name: 'Test' });
  expect(result.current.error).toBe(null);
});

test('handles errors', async () => {
  const testError = new Error('Network error');
  global.fetch = jest.fn(() => Promise.reject(testError)) as jest.Mock;
  
  const { result } = renderHook(() => useAsyncData('/api/data'));
  
  await waitFor(() => {
    expect(result.current.loading).toBe(false);
  });
  
  expect(result.current.data).toBe(null);
  expect(result.current.error).toBe(testError);
});
```

### Testing Polling/Intervals

```typescript
function useInterval(callback: () => void, delay: number | null) {
  const savedCallback = useRef(callback);
  
  useEffect(() => {
    savedCallback.current = callback;
  }, [callback]);
  
  useEffect(() => {
    if (delay === null) return;
    
    const id = setInterval(() => savedCallback.current(), delay);
    return () => clearInterval(id);
  }, [delay]);
}

test('useInterval calls callback at interval', () => {
  jest.useFakeTimers();
  const callback = jest.fn();
  
  renderHook(() => useInterval(callback, 1000));
  
  expect(callback).not.toHaveBeenCalled();
  
  jest.advanceTimersByTime(1000);
  expect(callback).toHaveBeenCalledTimes(1);
  
  jest.advanceTimersByTime(2000);
  expect(callback).toHaveBeenCalledTimes(3);
  
  jest.useRealTimers();
});
```

## Testing Hooks with Context

Use the `wrapper` option to provide context to hooks.

### Testing with Context Provider

```typescript
import { createContext, useContext, ReactNode } from 'react';

const ThemeContext = createContext({ theme: 'light', toggleTheme: () => {} });

function useTheme() {
  return useContext(ThemeContext);
}

test('useTheme with context provider', () => {
  const wrapper = ({ children }: { children: ReactNode }) => (
    <ThemeContext.Provider value={{ theme: 'dark', toggleTheme: jest.fn() }}>
      {children}
    </ThemeContext.Provider>
  );
  
  const { result } = renderHook(() => useTheme(), { wrapper });
  
  expect(result.current.theme).toBe('dark');
  expect(result.current.toggleTheme).toEqual(expect.any(Function));
});
```

### Custom Render with Multiple Providers

```typescript
interface User {
  id: number;
  name: string;
}

const UserContext = createContext<User | null>(null);
const AuthContext = createContext({ isAuthenticated: false });

function useCurrentUser() {
  return useContext(UserContext);
}

function useAuth() {
  return useContext(AuthContext);
}

test('hook with multiple providers', () => {
  const wrapper = ({ children }: { children: ReactNode }) => (
    <AuthContext.Provider value={{ isAuthenticated: true }}>
      <UserContext.Provider value={{ id: 1, name: 'Alice' }}>
        {children}
      </UserContext.Provider>
    </AuthContext.Provider>
  );
  
  const { result: userResult } = renderHook(() => useCurrentUser(), { wrapper });
  const { result: authResult } = renderHook(() => useAuth(), { wrapper });
  
  expect(userResult.current).toEqual({ id: 1, name: 'Alice' });
  expect(authResult.current.isAuthenticated).toBe(true);
});
```

### Testing Context Updates

```typescript
function useAuthState() {
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  
  const login = () => setIsAuthenticated(true);
  const logout = () => setIsAuthenticated(false);
  
  return { isAuthenticated, login, logout };
}

const AuthContext = createContext<ReturnType<typeof useAuthState> | null>(null);

function AuthProvider({ children }: { children: ReactNode }) {
  const auth = useAuthState();
  return <AuthContext.Provider value={auth}>{children}</AuthContext.Provider>;
}

function useAuth() {
  const context = useContext(AuthContext);
  if (!context) throw new Error('useAuth must be used within AuthProvider');
  return context;
}

test('auth state updates through context', () => {
  const wrapper = ({ children }: { children: ReactNode }) => (
    <AuthProvider>{children}</AuthProvider>
  );
  
  const { result } = renderHook(() => useAuth(), { wrapper });
  
  expect(result.current.isAuthenticated).toBe(false);
  
  act(() => {
    result.current.login();
  });
  
  expect(result.current.isAuthenticated).toBe(true);
  
  act(() => {
    result.current.logout();
  });
  
  expect(result.current.isAuthenticated).toBe(false);
});
```

## Testing Hooks with Dependencies

Mock external dependencies for isolated hook testing.

### Mocking Fetch

```typescript
test('hook with mocked fetch', async () => {
  const mockData = { id: 1, title: 'Test Post' };
  
  global.fetch = jest.fn(() =>
    Promise.resolve({
      json: () => Promise.resolve(mockData),
    })
  ) as jest.Mock;
  
  const { result } = renderHook(() => useAsyncData('/api/posts/1'));
  
  await waitFor(() => {
    expect(result.current.data).toEqual(mockData);
  });
  
  expect(global.fetch).toHaveBeenCalledWith('/api/posts/1');
});
```

### Mocking Modules

```typescript
// api.ts
export const fetchUser = async (id: number) => {
  const response = await fetch(`/api/users/${id}`);
  return response.json();
};

// useUser.ts
import { fetchUser } from './api';

function useUser(id: number) {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    fetchUser(id).then(setUser);
  }, [id]);
  
  return user;
}

// useUser.test.ts
jest.mock('./api');

test('useUser with mocked module', async () => {
  const mockUser = { id: 1, name: 'Alice' };
  (fetchUser as jest.Mock).mockResolvedValue(mockUser);
  
  const { result } = renderHook(() => useUser(1));
  
  await waitFor(() => {
    expect(result.current).toEqual(mockUser);
  });
  
  expect(fetchUser).toHaveBeenCalledWith(1);
});
```

### Mocking localStorage

```typescript
const mockLocalStorage = (() => {
  let store: Record<string, string> = {};
  
  return {
    getItem: (key: string) => store[key] || null,
    setItem: (key: string, value: string) => {
      store[key] = value;
    },
    removeItem: (key: string) => {
      delete store[key];
    },
    clear: () => {
      store = {};
    },
  };
})();

Object.defineProperty(window, 'localStorage', {
  value: mockLocalStorage,
});

test('hook with localStorage', () => {
  mockLocalStorage.clear();
  
  function useLocalStorage(key: string, initialValue: string) {
    const [value, setValue] = useState(() => {
      const stored = localStorage.getItem(key);
      return stored ?? initialValue;
    });
    
    useEffect(() => {
      localStorage.setItem(key, value);
    }, [key, value]);
    
    return [value, setValue] as const;
  }
  
  const { result } = renderHook(() => useLocalStorage('test-key', 'default'));
  
  expect(result.current[0]).toBe('default');
  
  act(() => {
    result.current[1]('new value');
  });
  
  expect(result.current[0]).toBe('new value');
  expect(localStorage.getItem('test-key')).toBe('new value');
});
```

## Testing useEffect and Cleanup

Test side effects and cleanup functions properly.

### Testing Side Effects

```typescript
function useLogger(message: string) {
  useEffect(() => {
    console.log(`Mounted: ${message}`);
    
    return () => {
      console.log(`Unmounted: ${message}`);
    };
  }, [message]);
}

test('useLogger logs on mount and unmount', () => {
  const consoleSpy = jest.spyOn(console, 'log');
  
  const { rerender, unmount } = renderHook(
    ({ message }) => useLogger(message),
    { initialProps: { message: 'Hello' } }
  );
  
  expect(consoleSpy).toHaveBeenCalledWith('Mounted: Hello');
  
  rerender({ message: 'Goodbye' });
  expect(consoleSpy).toHaveBeenCalledWith('Unmounted: Hello');
  expect(consoleSpy).toHaveBeenCalledWith('Mounted: Goodbye');
  
  unmount();
  expect(consoleSpy).toHaveBeenCalledWith('Unmounted: Goodbye');
  
  consoleSpy.mockRestore();
});
```

### Testing Cleanup

```typescript
function useEventListener(eventName: string, handler: EventListener) {
  useEffect(() => {
    window.addEventListener(eventName, handler);
    
    return () => {
      window.removeEventListener(eventName, handler);
    };
  }, [eventName, handler]);
}

test('useEventListener adds and removes listener', () => {
  const addSpy = jest.spyOn(window, 'addEventListener');
  const removeSpy = jest.spyOn(window, 'removeEventListener');
  const handler = jest.fn();
  
  const { unmount } = renderHook(() => useEventListener('click', handler));
  
  expect(addSpy).toHaveBeenCalledWith('click', handler);
  
  unmount();
  
  expect(removeSpy).toHaveBeenCalledWith('click', handler);
  
  addSpy.mockRestore();
  removeSpy.mockRestore();
});
```

## Example: Testing useFetch

Complete example testing a fetch hook.

```typescript
interface FetchState<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
}

function useFetch<T>(url: string): FetchState<T> & { refetch: () => void } {
  const [state, setState] = useState<FetchState<T>>({
    data: null,
    loading: true,
    error: null,
  });
  const [refetchIndex, setRefetchIndex] = useState(0);
  
  useEffect(() => {
    let cancelled = false;
    
    setState(prev => ({ ...prev, loading: true }));
    
    fetch(url)
      .then(res => res.json())
      .then(data => {
        if (!cancelled) {
          setState({ data, loading: false, error: null });
        }
      })
      .catch(error => {
        if (!cancelled) {
          setState({ data: null, loading: false, error });
        }
      });
    
    return () => {
      cancelled = true;
    };
  }, [url, refetchIndex]);
  
  const refetch = () => setRefetchIndex(i => i + 1);
  
  return { ...state, refetch };
}

describe('useFetch', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });
  
  test('fetches data successfully', async () => {
    const mockData = { id: 1, title: 'Test' };
    global.fetch = jest.fn(() =>
      Promise.resolve({
        json: () => Promise.resolve(mockData),
      })
    ) as jest.Mock;
    
    const { result } = renderHook(() => useFetch('/api/data'));
    
    expect(result.current.loading).toBe(true);
    expect(result.current.data).toBe(null);
    expect(result.current.error).toBe(null);
    
    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });
    
    expect(result.current.data).toEqual(mockData);
    expect(result.current.error).toBe(null);
  });
  
  test('handles fetch errors', async () => {
    const mockError = new Error('Network error');
    global.fetch = jest.fn(() => Promise.reject(mockError)) as jest.Mock;
    
    const { result } = renderHook(() => useFetch('/api/data'));
    
    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });
    
    expect(result.current.data).toBe(null);
    expect(result.current.error).toBe(mockError);
  });
  
  test('refetch triggers new request', async () => {
    const mockData1 = { id: 1, title: 'First' };
    const mockData2 = { id: 2, title: 'Second' };
    
    global.fetch = jest.fn()
      .mockResolvedValueOnce({ json: () => Promise.resolve(mockData1) })
      .mockResolvedValueOnce({ json: () => Promise.resolve(mockData2) }) as jest.Mock;
    
    const { result } = renderHook(() => useFetch('/api/data'));
    
    await waitFor(() => {
      expect(result.current.data).toEqual(mockData1);
    });
    
    act(() => {
      result.current.refetch();
    });
    
    expect(result.current.loading).toBe(true);
    
    await waitFor(() => {
      expect(result.current.data).toEqual(mockData2);
    });
    
    expect(global.fetch).toHaveBeenCalledTimes(2);
  });
  
  test('cancels request on unmount', async () => {
    global.fetch = jest.fn(() =>
      new Promise(resolve => setTimeout(() => resolve({ json: () => ({}) }), 1000))
    ) as jest.Mock;
    
    const { result, unmount } = renderHook(() => useFetch('/api/data'));
    
    expect(result.current.loading).toBe(true);
    
    unmount();
    
    // Wait to ensure no state updates after unmount
    await new Promise(resolve => setTimeout(resolve, 1100));
    
    // Should not throw "Can't perform a React state update on an unmounted component"
  });
});
```

## Example: Testing useDebounce

Testing a debounce hook with timers.

```typescript
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);
  
  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);
    
    return () => {
      clearTimeout(handler);
    };
  }, [value, delay]);
  
  return debouncedValue;
}

describe('useDebounce', () => {
  beforeEach(() => {
    jest.useFakeTimers();
  });
  
  afterEach(() => {
    jest.useRealTimers();
  });
  
  test('returns initial value immediately', () => {
    const { result } = renderHook(() => useDebounce('initial', 500));
    
    expect(result.current).toBe('initial');
  });
  
  test('debounces value changes', () => {
    const { result, rerender } = renderHook(
      ({ value, delay }) => useDebounce(value, delay),
      { initialProps: { value: 'initial', delay: 500 } }
    );
    
    expect(result.current).toBe('initial');
    
    rerender({ value: 'changed', delay: 500 });
    
    // Value should not change immediately
    expect(result.current).toBe('initial');
    
    // Advance timers
    act(() => {
      jest.advanceTimersByTime(500);
    });
    
    expect(result.current).toBe('changed');
  });
  
  test('cancels previous timeout on rapid changes', () => {
    const { result, rerender } = renderHook(
      ({ value, delay }) => useDebounce(value, delay),
      { initialProps: { value: 'first', delay: 500 } }
    );
    
    rerender({ value: 'second', delay: 500 });
    
    act(() => {
      jest.advanceTimersByTime(250);
    });
    
    rerender({ value: 'third', delay: 500 });
    
    act(() => {
      jest.advanceTimersByTime(250);
    });
    
    // Should still be initial
    expect(result.current).toBe('first');
    
    act(() => {
      jest.advanceTimersByTime(250);
    });
    
    // Now should be the latest value
    expect(result.current).toBe('third');
  });
});
```

## Example: Testing useLocalStorage

Complete localStorage hook testing.

```typescript
function useLocalStorage<T>(key: string, initialValue: T) {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.error(error);
      return initialValue;
    }
  });
  
  const setValue = (value: T | ((val: T) => T)) => {
    try {
      const valueToStore = value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.error(error);
    }
  };
  
  return [storedValue, setValue] as const;
}

describe('useLocalStorage', () => {
  beforeEach(() => {
    localStorage.clear();
  });
  
  test('initializes with default value', () => {
    const { result } = renderHook(() => useLocalStorage('test-key', 'default'));
    
    expect(result.current[0]).toBe('default');
  });
  
  test('initializes with stored value', () => {
    localStorage.setItem('test-key', JSON.stringify('stored'));
    
    const { result } = renderHook(() => useLocalStorage('test-key', 'default'));
    
    expect(result.current[0]).toBe('stored');
  });
  
  test('updates localStorage when value changes', () => {
    const { result } = renderHook(() => useLocalStorage('test-key', 'initial'));
    
    act(() => {
      result.current[1]('updated');
    });
    
    expect(result.current[0]).toBe('updated');
    expect(localStorage.getItem('test-key')).toBe(JSON.stringify('updated'));
  });
  
  test('works with complex objects', () => {
    const initialValue = { count: 0, name: 'Test' };
    const { result } = renderHook(() => useLocalStorage('test-key', initialValue));
    
    expect(result.current[0]).toEqual(initialValue);
    
    act(() => {
      result.current[1]({ count: 1, name: 'Updated' });
    });
    
    expect(result.current[0]).toEqual({ count: 1, name: 'Updated' });
    expect(JSON.parse(localStorage.getItem('test-key')!)).toEqual({
      count: 1,
      name: 'Updated',
    });
  });
  
  test('handles updater function', () => {
    const { result } = renderHook(() => useLocalStorage('test-key', 0));
    
    act(() => {
      result.current[1](prev => prev + 1);
    });
    
    expect(result.current[0]).toBe(1);
    
    act(() => {
      result.current[1](prev => prev + 5);
    });
    
    expect(result.current[0]).toBe(6);
  });
  
  test('handles localStorage errors gracefully', () => {
    const consoleErrorSpy = jest.spyOn(console, 'error').mockImplementation();
    
    // Mock localStorage.setItem to throw
    Storage.prototype.setItem = jest.fn(() => {
      throw new Error('QuotaExceededError');
    });
    
    const { result } = renderHook(() => useLocalStorage('test-key', 'initial'));
    
    act(() => {
      result.current[1]('updated');
    });
    
    expect(consoleErrorSpy).toHaveBeenCalled();
    
    consoleErrorSpy.mockRestore();
  });
});
```

## Common Mistakes

### 1. Not Using act() for State Updates

```typescript
// ❌ WRONG: State update without act()
test('counter (BAD)', () => {
  const { result } = renderHook(() => useCounter());
  result.current.increment(); // Warning!
  expect(result.current.count).toBe(1);
});

// ✅ CORRECT: Wrap in act()
test('counter (GOOD)', () => {
  const { result } = renderHook(() => useCounter());
  act(() => {
    result.current.increment();
  });
  expect(result.current.count).toBe(1);
});
```

### 2. Not Awaiting Async Operations

```typescript
// ❌ WRONG: Not awaiting
test('async hook (BAD)', () => {
  const { result } = renderHook(() => useFetch('/api/data'));
  expect(result.current.data).toBe(mockData); // Fails!
});

// ✅ CORRECT: Use waitFor
test('async hook (GOOD)', async () => {
  const { result } = renderHook(() => useFetch('/api/data'));
  await waitFor(() => {
    expect(result.current.data).toBe(mockData);
  });
});
```

### 3. Mutating result.current

```typescript
// ❌ WRONG: Mutating result
test('mutation (BAD)', () => {
  const { result } = renderHook(() => useCounter());
  result.current.count = 5; // Don't mutate!
});

// ✅ CORRECT: Use hook's API
test('mutation (GOOD)', () => {
  const { result } = renderHook(() => useCounter());
  act(() => {
    result.current.increment();
  });
});
```

### 4. Forgetting Cleanup in Tests

```typescript
// ❌ WRONG: Timers leak between tests
test('timer (BAD)', () => {
  jest.useFakeTimers();
  // ... test code ...
  // Forgot jest.useRealTimers()!
});

// ✅ CORRECT: Clean up after each test
afterEach(() => {
  jest.useRealTimers();
  jest.clearAllMocks();
});
```

## Best Practices

### 1. Use Descriptive Test Names

```typescript
// ✅ GOOD: Clear test names
test('useCounter initializes with provided initial value', () => {});
test('useCounter increments by 1 when increment is called', () => {});
test('useCounter resets to initial value when reset is called', () => {});
```

### 2. Test Hook Behavior, Not Implementation

```typescript
// ❌ BAD: Testing implementation
test('useCounter uses useState internally', () => {
  // Don't test internal implementation
});

// ✅ GOOD: Test observable behavior
test('useCounter returns incremented value after increment call', () => {
  const { result } = renderHook(() => useCounter());
  act(() => {
    result.current.increment();
  });
  expect(result.current.count).toBe(1);
});
```

### 3. Clean Up After Tests

```typescript
beforeEach(() => {
  localStorage.clear();
  jest.clearAllMocks();
});

afterEach(() => {
  jest.useRealTimers();
  jest.restoreAllMocks();
});
```

### 4. Use Proper Matchers

```typescript
// ✅ GOOD: Appropriate matchers
expect(result.current.count).toBe(5);
expect(result.current.items).toEqual(['a', 'b', 'c']);
expect(result.current.callback).toHaveBeenCalledTimes(2);
```

## Interview Questions

### Q1: What is renderHook and when do you use it?

**Answer:** `renderHook` is a utility from React Testing Library that allows you to test custom hooks in isolation without creating wrapper components. It returns `result.current` which contains the hook's return value, and utilities like `rerender` and `unmount` for testing different scenarios.

### Q2: Why do you need act() when testing hooks?

**Answer:** `act()` ensures all React updates (state changes, effects) are processed and applied before making assertions. Without it, you may get warnings about state updates not being wrapped in act(), and assertions may run before the component has fully updated.

### Q3: How do you test async hooks?

**Answer:** Use `waitFor` from React Testing Library to wait for async state updates. Mock async dependencies like fetch, and use `waitFor(() => expect(...))` to wait for the hook to finish loading and update its state. For timers, use `jest.useFakeTimers()` and `jest.advanceTimersByTime()`.

### Q4: How do you test hooks that use context?

**Answer:** Use the `wrapper` option in `renderHook` to provide context providers. Create a wrapper component that includes all necessary providers and pass it as the second argument: `renderHook(() => useMyHook(), { wrapper })`.

### Q5: What's the difference between rerender and unmount?

**Answer:** `rerender` allows you to test how a hook responds to prop changes by triggering a re-render with new props. `unmount` completely unmounts the component, which is useful for testing cleanup functions in `useEffect` to ensure resources are properly released.

### Q6: How do you test hook cleanup functions?

**Answer:** Call `unmount()` returned from `renderHook` and verify that cleanup occurred (event listeners removed, timers cleared, etc.). You can spy on functions like `removeEventListener` or `clearInterval` to verify cleanup was called.

## Key Takeaways

1. Use `renderHook` to test custom hooks in isolation
2. Access hook return value via `result.current`
3. Wrap state updates in `act()` to avoid warnings
4. Use `rerender` to test hooks with changing props
5. Use `waitFor` for async operations
6. Provide context via `wrapper` option
7. Mock external dependencies (fetch, localStorage, etc.)
8. Test `useEffect` cleanup by calling `unmount()`
9. Use fake timers for testing intervals and debouncing
10. Test behavior, not implementation details
11. Clean up mocks and timers after each test
12. Write descriptive test names

## Resources

- [React Testing Library - Hooks](https://testing-library.com/docs/react-testing-library/api/#renderhook)
- [Kent C. Dodds - How to Test Custom React Hooks](https://kentcdodds.com/blog/how-to-test-custom-react-hooks)
- [Testing Library - Async Utilities](https://testing-library.com/docs/dom-testing-library/api-async/)
- [Jest - Timer Mocks](https://jestjs.io/docs/timer-mocks)
- [Testing Library - Setup](https://testing-library.com/docs/react-testing-library/setup)
