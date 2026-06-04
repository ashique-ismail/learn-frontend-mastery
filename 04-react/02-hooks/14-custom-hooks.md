# Custom Hooks - Reusable Logic Patterns

## Table of Contents
- [Introduction](#introduction)
- [Basic Concepts](#basic-concepts)
- [Creating Custom Hooks](#creating-custom-hooks)
- [Common Patterns](#common-patterns)
- [Advanced Custom Hooks](#advanced-custom-hooks)
- [Testing Custom Hooks](#testing-custom-hooks)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Real-World Examples](#real-world-examples)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Custom Hooks are JavaScript functions that use React Hooks and allow you to extract and reuse stateful logic across multiple components. They follow the naming convention of starting with "use" and can call other Hooks.

### Why Custom Hooks?

```javascript
// ❌ Without Custom Hooks: Duplicate logic
function UserProfile() {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch('/api/user')
      .then(res => res.json())
      .then(setUser)
      .catch(setError)
      .finally(() => setLoading(false));
  }, []);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  return <div>{user.name}</div>;
}

function UserPosts() {
  const [posts, setPosts] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch('/api/posts')
      .then(res => res.json())
      .then(setPosts)
      .catch(setError)
      .finally(() => setLoading(false));
  }, []);

  // Same loading/error logic repeated!
}

// ✅ With Custom Hook: Reusable logic
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch(url)
      .then(res => res.json())
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [url]);

  return { data, loading, error };
}

function UserProfile() {
  const { data: user, loading, error } = useFetch('/api/user');
  
  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  return <div>{user.name}</div>;
}

function UserPosts() {
  const { data: posts, loading, error } = useFetch('/api/posts');
  // Clean and reusable!
}
```

## Basic Concepts

### Rules of Custom Hooks

1. **Name must start with "use"**
2. **Can call other Hooks**
3. **Must be called at top level**
4. **Can return anything** (values, functions, objects, arrays)

```javascript
// ✅ VALID: Starts with "use"
function useCounter(initialValue = 0) {
  const [count, setCount] = useState(initialValue);
  return [count, setCount];
}

// ❌ INVALID: Doesn't start with "use"
function counter(initialValue = 0) {
  const [count, setCount] = useState(initialValue); // ❌ Hook call outside "use" function
  return [count, setCount];
}

// ✅ VALID: Calls other hooks
function useAuth() {
  const user = useContext(AuthContext);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    // Can use multiple hooks
  }, []);
  
  return { user, loading };
}
```

### Basic Structure

```javascript
function useCustomHook(parameters) {
  // 1. Declare state
  const [state, setState] = useState(initialValue);
  
  // 2. Define effects
  useEffect(() => {
    // Side effects
    return () => {
      // Cleanup
    };
  }, [dependencies]);
  
  // 3. Define helper functions
  const helperFunction = useCallback(() => {
    // Logic
  }, [dependencies]);
  
  // 4. Return values/functions
  return {
    state,
    helperFunction
  };
}
```

## Creating Custom Hooks

### Simple State Hook

```javascript
function useToggle(initialValue = false) {
  const [value, setValue] = useState(initialValue);
  
  const toggle = useCallback(() => {
    setValue(v => !v);
  }, []);
  
  const setTrue = useCallback(() => {
    setValue(true);
  }, []);
  
  const setFalse = useCallback(() => {
    setValue(false);
  }, []);
  
  return [value, { toggle, setTrue, setFalse }];
}

// Usage
function Component() {
  const [isOpen, { toggle, setTrue, setFalse }] = useToggle(false);
  
  return (
    <div>
      <p>Is open: {isOpen ? 'Yes' : 'No'}</p>
      <button onClick={toggle}>Toggle</button>
      <button onClick={setTrue}>Open</button>
      <button onClick={setFalse}>Close</button>
    </div>
  );
}
```

### Effect-based Hook

```javascript
function useDocumentTitle(title) {
  useEffect(() => {
    const previousTitle = document.title;
    document.title = title;
    
    return () => {
      document.title = previousTitle;
    };
  }, [title]);
}

// Usage
function PageComponent() {
  useDocumentTitle('My Page Title');
  
  return <div>Content</div>;
}
```

### Complex State Hook

```javascript
function useLocalStorage(key, initialValue) {
  // Get initial value from localStorage
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.error(error);
      return initialValue;
    }
  });

  // Update localStorage when value changes
  const setValue = useCallback((value) => {
    try {
      const valueToStore = value instanceof Function 
        ? value(storedValue) 
        : value;
      
      setStoredValue(valueToStore);
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.error(error);
    }
  }, [key, storedValue]);

  // Remove from localStorage
  const removeValue = useCallback(() => {
    try {
      window.localStorage.removeItem(key);
      setStoredValue(initialValue);
    } catch (error) {
      console.error(error);
    }
  }, [key, initialValue]);

  return [storedValue, setValue, removeValue];
}

// Usage
function Settings() {
  const [theme, setTheme, removeTheme] = useLocalStorage('theme', 'light');
  
  return (
    <div>
      <p>Current theme: {theme}</p>
      <button onClick={() => setTheme('dark')}>Dark</button>
      <button onClick={() => setTheme('light')}>Light</button>
      <button onClick={removeTheme}>Reset</button>
    </div>
  );
}
```

## Common Patterns

### Data Fetching Hook

```javascript
function useFetch(url, options = {}) {
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    let isCancelled = false;

    const fetchData = async () => {
      setLoading(true);
      setError(null);

      try {
        const response = await fetch(url, options);
        
        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }
        
        const json = await response.json();
        
        if (!isCancelled) {
          setData(json);
        }
      } catch (error) {
        if (!isCancelled) {
          setError(error);
        }
      } finally {
        if (!isCancelled) {
          setLoading(false);
        }
      }
    };

    fetchData();

    return () => {
      isCancelled = true;
    };
  }, [url, JSON.stringify(options)]);

  return { data, error, loading };
}

// Usage
function UserList() {
  const { data, error, loading } = useFetch('/api/users');

  if (loading) return <Spinner />;
  if (error) return <Error message={error.message} />;
  
  return (
    <ul>
      {data.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

### Form Handling Hook

```javascript
function useForm(initialValues, validate) {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState({});
  const [touched, setTouched] = useState({});
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleChange = useCallback((name, value) => {
    setValues(prev => ({ ...prev, [name]: value }));
    
    // Clear error when user starts typing
    if (errors[name]) {
      setErrors(prev => ({ ...prev, [name]: undefined }));
    }
  }, [errors]);

  const handleBlur = useCallback((name) => {
    setTouched(prev => ({ ...prev, [name]: true }));
    
    if (validate) {
      const fieldErrors = validate({ ...values });
      setErrors(fieldErrors);
    }
  }, [values, validate]);

  const handleSubmit = useCallback(async (onSubmit) => {
    return async (e) => {
      e?.preventDefault();
      
      // Mark all fields as touched
      const touchedFields = Object.keys(values).reduce((acc, key) => {
        acc[key] = true;
        return acc;
      }, {});
      setTouched(touchedFields);

      // Validate
      if (validate) {
        const validationErrors = validate(values);
        setErrors(validationErrors);
        
        if (Object.keys(validationErrors).length > 0) {
          return;
        }
      }

      setIsSubmitting(true);
      try {
        await onSubmit(values);
      } catch (error) {
        setErrors({ submit: error.message });
      } finally {
        setIsSubmitting(false);
      }
    };
  }, [values, validate]);

  const reset = useCallback(() => {
    setValues(initialValues);
    setErrors({});
    setTouched({});
    setIsSubmitting(false);
  }, [initialValues]);

  return {
    values,
    errors,
    touched,
    isSubmitting,
    handleChange,
    handleBlur,
    handleSubmit,
    reset
  };
}

// Usage
function LoginForm() {
  const validate = (values) => {
    const errors = {};
    if (!values.email) {
      errors.email = 'Required';
    } else if (!/\S+@\S+\.\S+/.test(values.email)) {
      errors.email = 'Invalid email';
    }
    if (!values.password) {
      errors.password = 'Required';
    }
    return errors;
  };

  const {
    values,
    errors,
    touched,
    isSubmitting,
    handleChange,
    handleBlur,
    handleSubmit
  } = useForm({ email: '', password: '' }, validate);

  const onSubmit = async (values) => {
    await login(values);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input
        name="email"
        value={values.email}
        onChange={(e) => handleChange('email', e.target.value)}
        onBlur={() => handleBlur('email')}
      />
      {touched.email && errors.email && <span>{errors.email}</span>}
      
      <input
        type="password"
        name="password"
        value={values.password}
        onChange={(e) => handleChange('password', e.target.value)}
        onBlur={() => handleBlur('password')}
      />
      {touched.password && errors.password && <span>{errors.password}</span>}
      
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Submitting...' : 'Submit'}
      </button>
    </form>
  );
}
```

### Event Listener Hook

```javascript
function useEventListener(eventName, handler, element = window) {
  const savedHandler = useRef(handler);

  useEffect(() => {
    savedHandler.current = handler;
  }, [handler]);

  useEffect(() => {
    const isSupported = element && element.addEventListener;
    if (!isSupported) return;

    const eventListener = (event) => savedHandler.current(event);
    element.addEventListener(eventName, eventListener);

    return () => {
      element.removeEventListener(eventName, eventListener);
    };
  }, [eventName, element]);
}

// Usage
function Component() {
  const [coords, setCoords] = useState({ x: 0, y: 0 });

  useEventListener('mousemove', (e) => {
    setCoords({ x: e.clientX, y: e.clientY });
  });

  return (
    <div>
      Mouse position: {coords.x}, {coords.y}
    </div>
  );
}
```

### Window Size Hook

```javascript
function useWindowSize() {
  const [windowSize, setWindowSize] = useState({
    width: undefined,
    height: undefined
  });

  useEffect(() => {
    function handleResize() {
      setWindowSize({
        width: window.innerWidth,
        height: window.innerHeight
      });
    }

    window.addEventListener('resize', handleResize);
    handleResize(); // Call initially

    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return windowSize;
}

// Usage
function ResponsiveComponent() {
  const { width } = useWindowSize();

  return (
    <div>
      {width < 768 ? (
        <MobileView />
      ) : (
        <DesktopView />
      )}
    </div>
  );
}
```

## Advanced Custom Hooks

### Debounce Hook

```javascript
function useDebounce(value, delay) {
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

// Usage
function SearchComponent() {
  const [searchTerm, setSearchTerm] = useState('');
  const debouncedSearchTerm = useDebounce(searchTerm, 500);

  useEffect(() => {
    if (debouncedSearchTerm) {
      // API call with debounced term
      searchAPI(debouncedSearchTerm);
    }
  }, [debouncedSearchTerm]);

  return (
    <input
      value={searchTerm}
      onChange={(e) => setSearchTerm(e.target.value)}
      placeholder="Search..."
    />
  );
}
```

### Previous Value Hook

```javascript
function usePrevious(value) {
  const ref = useRef();

  useEffect(() => {
    ref.current = value;
  }, [value]);

  return ref.current;
}

// Usage
function Counter() {
  const [count, setCount] = useState(0);
  const prevCount = usePrevious(count);

  return (
    <div>
      <p>Current: {count}</p>
      <p>Previous: {prevCount}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}
```

### Async Hook with Cancellation

```javascript
function useAsync(asyncFunction, immediate = true) {
  const [status, setStatus] = useState('idle');
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);

  const execute = useCallback((...params) => {
    setStatus('pending');
    setData(null);
    setError(null);

    return asyncFunction(...params)
      .then((response) => {
        setData(response);
        setStatus('success');
      })
      .catch((error) => {
        setError(error);
        setStatus('error');
      });
  }, [asyncFunction]);

  useEffect(() => {
    if (immediate) {
      execute();
    }
  }, [execute, immediate]);

  return { execute, status, data, error };
}

// Usage
function UserProfile({ userId }) {
  const fetchUser = useCallback(
    () => fetch(`/api/users/${userId}`).then(res => res.json()),
    [userId]
  );

  const { data: user, status, error } = useAsync(fetchUser, true);

  if (status === 'pending') return <Spinner />;
  if (status === 'error') return <Error message={error.message} />;
  if (status === 'success') return <UserCard user={user} />;
  
  return null;
}
```

### Intersection Observer Hook

```javascript
function useIntersectionObserver(
  elementRef,
  { threshold = 0, root = null, rootMargin = '0%' } = {}
) {
  const [entry, setEntry] = useState(null);

  useEffect(() => {
    const element = elementRef.current;
    if (!element) return;

    const observer = new IntersectionObserver(
      ([entry]) => setEntry(entry),
      { threshold, root, rootMargin }
    );

    observer.observe(element);

    return () => {
      observer.disconnect();
    };
  }, [elementRef, threshold, root, rootMargin]);

  return entry;
}

// Usage
function LazyImage({ src, alt }) {
  const imageRef = useRef(null);
  const entry = useIntersectionObserver(imageRef, { threshold: 0.1 });
  const isVisible = entry?.isIntersecting;

  return (
    <div ref={imageRef}>
      {isVisible ? (
        <img src={src} alt={alt} />
      ) : (
        <div className="placeholder">Loading...</div>
      )}
    </div>
  );
}
```

### Media Query Hook

```javascript
function useMediaQuery(query) {
  const [matches, setMatches] = useState(false);

  useEffect(() => {
    const media = window.matchMedia(query);
    
    if (media.matches !== matches) {
      setMatches(media.matches);
    }

    const listener = () => setMatches(media.matches);
    media.addEventListener('change', listener);

    return () => media.removeEventListener('change', listener);
  }, [matches, query]);

  return matches;
}

// Usage
function ResponsiveComponent() {
  const isMobile = useMediaQuery('(max-width: 768px)');
  const isTablet = useMediaQuery('(min-width: 769px) and (max-width: 1024px)');
  const isDesktop = useMediaQuery('(min-width: 1025px)');

  return (
    <div>
      {isMobile && <MobileLayout />}
      {isTablet && <TabletLayout />}
      {isDesktop && <DesktopLayout />}
    </div>
  );
}
```

## Testing Custom Hooks

### Using React Testing Library

```javascript
import { renderHook, act } from '@testing-library/react';
import { useCounter } from './useCounter';

describe('useCounter', () => {
  it('should initialize with default value', () => {
    const { result } = renderHook(() => useCounter());
    expect(result.current.count).toBe(0);
  });

  it('should initialize with provided value', () => {
    const { result } = renderHook(() => useCounter(10));
    expect(result.current.count).toBe(10);
  });

  it('should increment counter', () => {
    const { result } = renderHook(() => useCounter());
    
    act(() => {
      result.current.increment();
    });
    
    expect(result.current.count).toBe(1);
  });

  it('should decrement counter', () => {
    const { result } = renderHook(() => useCounter(5));
    
    act(() => {
      result.current.decrement();
    });
    
    expect(result.current.count).toBe(4);
  });

  it('should reset counter', () => {
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
```

### Testing Async Hooks

```javascript
import { renderHook, waitFor } from '@testing-library/react';
import { useFetch } from './useFetch';

describe('useFetch', () => {
  beforeEach(() => {
    global.fetch = jest.fn();
  });

  it('should fetch data successfully', async () => {
    const mockData = { id: 1, name: 'John' };
    global.fetch.mockResolvedValueOnce({
      ok: true,
      json: async () => mockData
    });

    const { result } = renderHook(() => useFetch('/api/user'));

    expect(result.current.loading).toBe(true);
    expect(result.current.data).toBe(null);

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(result.current.data).toEqual(mockData);
    expect(result.current.error).toBe(null);
  });

  it('should handle errors', async () => {
    global.fetch.mockRejectedValueOnce(new Error('Network error'));

    const { result } = renderHook(() => useFetch('/api/user'));

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(result.current.error).toBeTruthy();
    expect(result.current.data).toBe(null);
  });
});
```

## Common Mistakes

### 1. Not Following Naming Convention

```javascript
// ❌ WRONG: Doesn't start with "use"
function counter() {
  const [count, setCount] = useState(0);
  return [count, setCount];
}

// ✅ CORRECT: Starts with "use"
function useCounter() {
  const [count, setCount] = useState(0);
  return [count, setCount];
}
```

### 2. Conditionally Calling Hooks

```javascript
// ❌ WRONG: Conditional hook call
function useData(shouldFetch) {
  if (shouldFetch) {
    const [data, setData] = useState(null); // ❌ Breaks rules
  }
}

// ✅ CORRECT: Always call hooks
function useData(shouldFetch) {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    if (shouldFetch) {
      fetchData().then(setData);
    }
  }, [shouldFetch]);
  
  return data;
}
```

### 3. Not Cleaning Up Side Effects

```javascript
// ❌ WRONG: No cleanup
function useInterval(callback, delay) {
  useEffect(() => {
    const id = setInterval(callback, delay);
    // Missing cleanup!
  }, [callback, delay]);
}

// ✅ CORRECT: Proper cleanup
function useInterval(callback, delay) {
  useEffect(() => {
    const id = setInterval(callback, delay);
    
    return () => {
      clearInterval(id);
    };
  }, [callback, delay]);
}
```

### 4. Stale Closures

```javascript
// ❌ WRONG: Stale closure
function useCounter() {
  const [count, setCount] = useState(0);
  
  const increment = () => {
    setCount(count + 1); // Uses stale count
  };
  
  return { count, increment };
}

// ✅ CORRECT: Functional update
function useCounter() {
  const [count, setCount] = useState(0);
  
  const increment = useCallback(() => {
    setCount(c => c + 1); // Always uses current count
  }, []);
  
  return { count, increment };
}
```

## Best Practices

### 1. Use TypeScript for Better DX

```typescript
interface UseToggleReturn {
  value: boolean;
  toggle: () => void;
  setTrue: () => void;
  setFalse: () => void;
}

function useToggle(initialValue: boolean = false): UseToggleReturn {
  const [value, setValue] = useState(initialValue);
  
  const toggle = useCallback(() => setValue(v => !v), []);
  const setTrue = useCallback(() => setValue(true), []);
  const setFalse = useCallback(() => setValue(false), []);
  
  return { value, toggle, setTrue, setFalse };
}
```

### 2. Document Your Hooks

```javascript
/**
 * Custom hook for debouncing a value
 * 
 * @param {any} value - The value to debounce
 * @param {number} delay - Delay in milliseconds
 * @returns {any} - The debounced value
 * 
 * @example
 * const debouncedSearchTerm = useDebounce(searchTerm, 500);
 */
function useDebounce(value, delay) {
  // Implementation
}
```

### 3. Return Consistent Data Structures

```javascript
// ✅ GOOD: Consistent object return
function useFetch(url) {
  return { data, loading, error };
}

// ✅ GOOD: Consistent array return (like useState)
function useToggle(initial) {
  return [value, { toggle, setTrue, setFalse }];
}

// ❌ BAD: Inconsistent returns
function useData(url) {
  if (error) return error;
  if (loading) return null;
  return data; // Inconsistent!
}
```

### 4. Keep Hooks Focused

```javascript
// ❌ BAD: Too much responsibility
function useEverything() {
  const user = useUser();
  const posts = usePosts();
  const comments = useComments();
  const likes = useLikes();
  // Too much!
}

// ✅ GOOD: Single responsibility
function useUser() {
  // Just user logic
}

function usePosts() {
  // Just posts logic
}
```

## Real-World Examples

### Complete Authentication Hook

```javascript
function useAuth() {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const token = localStorage.getItem('authToken');
    if (token) {
      verifyToken(token)
        .then(setUser)
        .catch((err) => {
          localStorage.removeItem('authToken');
          setError(err);
        })
        .finally(() => setLoading(false));
    } else {
      setLoading(false);
    }
  }, []);

  const login = useCallback(async (credentials) => {
    setLoading(true);
    setError(null);
    
    try {
      const { user, token } = await loginAPI(credentials);
      localStorage.setItem('authToken', token);
      setUser(user);
      return user;
    } catch (err) {
      setError(err);
      throw err;
    } finally {
      setLoading(false);
    }
  }, []);

  const logout = useCallback(async () => {
    setLoading(true);
    
    try {
      await logoutAPI();
      localStorage.removeItem('authToken');
      setUser(null);
    } catch (err) {
      setError(err);
    } finally {
      setLoading(false);
    }
  }, []);

  const updateUser = useCallback((updates) => {
    setUser(prev => ({ ...prev, ...updates }));
  }, []);

  return {
    user,
    loading,
    error,
    login,
    logout,
    updateUser,
    isAuthenticated: !!user
  };
}
```

### Pagination Hook

```javascript
function usePagination(data, itemsPerPage = 10) {
  const [currentPage, setCurrentPage] = useState(1);

  const maxPage = Math.ceil(data.length / itemsPerPage);

  const currentData = useMemo(() => {
    const begin = (currentPage - 1) * itemsPerPage;
    const end = begin + itemsPerPage;
    return data.slice(begin, end);
  }, [data, currentPage, itemsPerPage]);

  const next = useCallback(() => {
    setCurrentPage(page => Math.min(page + 1, maxPage));
  }, [maxPage]);

  const prev = useCallback(() => {
    setCurrentPage(page => Math.max(page - 1, 1));
  }, []);

  const jump = useCallback((page) => {
    const pageNumber = Math.max(1, Math.min(page, maxPage));
    setCurrentPage(pageNumber);
  }, [maxPage]);

  return {
    currentData,
    currentPage,
    maxPage,
    next,
    prev,
    jump,
    hasNext: currentPage < maxPage,
    hasPrev: currentPage > 1
  };
}

// Usage
function UserList({ users }) {
  const {
    currentData,
    currentPage,
    maxPage,
    next,
    prev,
    hasNext,
    hasPrev
  } = usePagination(users, 20);

  return (
    <div>
      <ul>
        {currentData.map(user => (
          <li key={user.id}>{user.name}</li>
        ))}
      </ul>
      
      <div className="pagination">
        <button onClick={prev} disabled={!hasPrev}>
          Previous
        </button>
        <span>Page {currentPage} of {maxPage}</span>
        <button onClick={next} disabled={!hasNext}>
          Next
        </button>
      </div>
    </div>
  );
}
```

## Interview Questions

### Q1: What are the rules for creating custom hooks?

**Answer:**
1. Name must start with "use"
2. Can only be called at the top level (not in loops/conditions)
3. Can call other hooks
4. Can return anything (values, functions, objects)

### Q2: Why should custom hooks start with "use"?

**Answer:** 
The "use" prefix allows React and linting tools to identify hooks and enforce their rules. It's a convention that signals the function uses hooks internally and should follow hook rules.

### Q3: How do you test custom hooks?

**Answer:**
Use `@testing-library/react-hooks`:

```javascript
import { renderHook, act } from '@testing-library/react';

const { result } = renderHook(() => useCounter());

act(() => {
  result.current.increment();
});

expect(result.current.count).toBe(1);
```

## Key Takeaways

1. **Custom hooks extract reusable logic** - Share stateful logic between components
2. **Must follow naming convention** - Start with "use"
3. **Can call other hooks** - Build on top of React's built-in hooks
4. **Return flexible data structures** - Objects, arrays, or values
5. **Enable composition** - Combine multiple hooks for complex logic
6. **Improve testability** - Logic separated from components
7. **Type them properly** - TypeScript provides better DX

## Resources

### Official Documentation
- [Building Your Own Hooks](https://react.dev/learn/reusing-logic-with-custom-hooks)
- [Rules of Hooks](https://react.dev/warnings/invalid-hook-call-warning)

### Articles
- [How to Create Custom Hooks](https://www.robinwieruch.de/react-custom-hook)
- [Custom Hook Patterns](https://kentcdodds.com/blog/react-hooks-whats-going-to-happen-to-render-props)

### Tools
- [@testing-library/react-hooks](https://github.com/testing-library/react-hooks-testing-library)

### Hook Collections
- [usehooks.com](https://usehooks.com/)
- [useHooks(🐠)](https://usehooks-ts.com/)
- [react-use](https://github.com/streamich/react-use)

---

**Next Steps:** Explore advanced patterns like compound components, render props, and higher-order components to complement your custom hooks knowledge.
