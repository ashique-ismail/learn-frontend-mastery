# React Hooks - Interview Questions

## The Idea

**In plain English:** React Hooks are special functions that let you add memory and behaviour to a reusable piece of UI (called a component) without writing a full class. "Memory" here means the component can remember a value — like a score or a username — and update the screen when that value changes.

**Real-world analogy:** Think of a vending machine at school. The machine tracks how many of each snack is left, reacts when you press a button, and cleans up (locks the dispenser door) when the transaction is done. Each of those jobs is handled by a separate built-in mechanism inside the machine, not bolted on from the outside.

- The snack count display = `useState` (stores and shows a value that can change)
- The sensor that reacts when you press a button = `useEffect` (runs code in response to something changing)
- The lock that engages after the transaction = the cleanup function returned by `useEffect` (tidies up when the job is done)

---

## Table of Contents

- [Common Questions](#common-questions)
- [Advanced Questions](#advanced-questions)
- [Custom Hooks](#custom-hooks)
- [What Interviewers Look For](#what-interviewers-look-for)
- [Key Takeaways](#key-takeaways)

## Common Questions

### Question 1: useState - How Does it Work?

**Expected Answer:**

```javascript
// Basic usage
const [count, setCount] = useState(0);

// Functional update (when new state depends on previous)
setCount(prevCount => prevCount + 1);

// Lazy initialization (expensive computation)
const [data, setData] = useState(() => {
  return expensiveComputation();
});

// Multiple state variables
const [name, setName] = useState('');
const [age, setAge] = useState(0);

// Object state (must spread to update)
const [user, setUser] = useState({ name: '', email: '' });
setUser(prev => ({ ...prev, name: 'John' }));

// Common mistake: Direct mutation
// setUser(user.name = 'John'); // WRONG
setUser({ ...user, name: 'John' }); // RIGHT
```

### Question 2: useEffect - Side Effects and Cleanup

**Expected Answer:**

```javascript
// Runs after every render
useEffect(() => {
  console.log('Effect ran');
});

// Runs only on mount
useEffect(() => {
  console.log('Mounted');
}, []);

// Runs when dependencies change
useEffect(() => {
  fetchData(userId);
}, [userId]);

// Cleanup function
useEffect(() => {
  const subscription = subscribeToData();
  
  return () => {
    subscription.unsubscribe(); // Cleanup
  };
}, []);

// Multiple effects for separation of concerns
useEffect(() => {
  // Effect 1: Fetch data
}, [userId]);

useEffect(() => {
  // Effect 2: Set up listener
}, []);

// Common mistake: Missing dependencies
useEffect(() => {
  fetchData(userId); // userId not in deps array
}, []); // WRONG - stale closure

// Fix with ESLint plugin
useEffect(() => {
  fetchData(userId);
}, [userId, fetchData]); // Include all dependencies
```

### Question 3: useContext - Global State

**Expected Answer:**

```javascript
// Create context
const ThemeContext = createContext('light');

// Provider
function App() {
  const [theme, setTheme] = useState('light');
  
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <Component />
    </ThemeContext.Provider>
  );
}

// Consumer with useContext
function Component() {
  const { theme, setTheme } = useContext(ThemeContext);
  
  return (
    <div className={theme}>
      <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
        Toggle Theme
      </button>
    </div>
  );
}

// Multiple contexts
function MultiContextComponent() {
  const theme = useContext(ThemeContext);
  const user = useContext(UserContext);
  const settings = useContext(SettingsContext);
  
  return <div>{/* Use contexts */}</div>;
}

// Avoid re-renders: Split contexts
const ThemeContext = createContext();
const ThemeUpdateContext = createContext();

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');
  
  return (
    <ThemeContext.Provider value={theme}>
      <ThemeUpdateContext.Provider value={setTheme}>
        {children}
      </ThemeUpdateContext.Provider>
    </ThemeContext.Provider>
  );
}
```

### Question 4: useReducer - Complex State Logic

**Expected Answer:**

```javascript
// Reducer function
function reducer(state, action) {
  switch (action.type) {
    case 'increment':
      return { count: state.count + 1 };
    case 'decrement':
      return { count: state.count - 1 };
    case 'reset':
      return { count: 0 };
    default:
      throw new Error(`Unknown action: ${action.type}`);
  }
}

// Usage
function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0 });
  
  return (
    <div>
      Count: {state.count}
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
      <button onClick={() => dispatch({ type: 'reset' })}>Reset</button>
    </div>
  );
}

// With payload
function todoReducer(state, action) {
  switch (action.type) {
    case 'add':
      return [...state, { id: Date.now(), text: action.payload }];
    case 'delete':
      return state.filter(todo => todo.id !== action.payload);
    case 'toggle':
      return state.map(todo =>
        todo.id === action.payload
          ? { ...todo, done: !todo.done }
          : todo
      );
    default:
      return state;
  }
}

// Lazy initialization
function init(initialCount) {
  return { count: initialCount };
}

const [state, dispatch] = useReducer(reducer, initialCount, init);
```

### Question 5: useCallback and useMemo

**Expected Answer:**

```javascript
// useCallback - memoize function
function Parent() {
  const [count, setCount] = useState(0);
  
  // Without useCallback: new function every render
  const handleClick = () => {
    console.log('Clicked');
  };
  
  // With useCallback: same function reference
  const memoizedCallback = useCallback(() => {
    console.log('Clicked', count);
  }, [count]); // Recreate when count changes
  
  return <Child onClick={memoizedCallback} />;
}

const Child = React.memo(({ onClick }) => {
  console.log('Child rendered');
  return <button onClick={onClick}>Click</button>;
});

// useMemo - memoize computed value
function ExpensiveComponent({ data }) {
  // Recomputes every render
  const result = expensiveCalculation(data);
  
  // Memoized: Only recomputes when data changes
  const memoizedResult = useMemo(() => {
    return expensiveCalculation(data);
  }, [data]);
  
  return <div>{memoizedResult}</div>;
}

// When to use
// useCallback: Passing callbacks to optimized children
// useMemo: Expensive computations, referential equality

// Don't overuse!
// Bad: Memoizing everything
const simple = useMemo(() => a + b, [a, b]); // Unnecessary

// Good: Actual expensive operation
const filtered = useMemo(() => {
  return largeArray
    .filter(item => item.active)
    .map(item => heavyTransform(item));
}, [largeArray]);
```

### Question 6: useRef - Mutable Values and DOM Access

**Expected Answer:**

```javascript
// DOM access
function TextInput() {
  const inputRef = useRef(null);
  
  const focus = () => {
    inputRef.current.focus();
  };
  
  return (
    <div>
      <input ref={inputRef} />
      <button onClick={focus}>Focus Input</button>
    </div>
  );
}

// Mutable value (doesn't trigger re-render)
function Timer() {
  const [count, setCount] = useState(0);
  const intervalRef = useRef();
  
  useEffect(() => {
    intervalRef.current = setInterval(() => {
      setCount(c => c + 1);
    }, 1000);
    
    return () => clearInterval(intervalRef.current);
  }, []);
  
  const stop = () => {
    clearInterval(intervalRef.current);
  };
  
  return <div>{count} <button onClick={stop}>Stop</button></div>;
}

// Previous value tracking
function usePrevious(value) {
  const ref = useRef();
  
  useEffect(() => {
    ref.current = value;
  });
  
  return ref.current;
}

// useRef vs useState
// useRef: Doesn't trigger re-render when changed
// useState: Triggers re-render when changed

function Example() {
  const [state, setState] = useState(0); // Triggers re-render
  const ref = useRef(0); // Doesn't trigger re-render
  
  const updateState = () => setState(s => s + 1); // Re-renders
  const updateRef = () => ref.current += 1; // No re-render
}
```

## Advanced Questions

### Question 7: useLayoutEffect vs useEffect

**Expected Answer:**

```javascript
// useEffect: Runs after paint (async)
useEffect(() => {
  // Runs after browser paints
  console.log('useEffect');
}, []);

// useLayoutEffect: Runs before paint (sync)
useLayoutEffect(() => {
  // Runs before browser paints
  // Blocks visual updates
  console.log('useLayoutEffect');
}, []);

// When to use useLayoutEffect
function Tooltip() {
  const [coords, setCoords] = useState({ x: 0, y: 0 });
  const ref = useRef();
  
  useLayoutEffect(() => {
    // Measure DOM before paint
    const rect = ref.current.getBoundingClientRect();
    setCoords({ x: rect.x, y: rect.y });
  }, []);
  
  return <div ref={ref} style={{ left: coords.x, top: coords.y }}>
    Tooltip
  </div>;
}

// Use cases for useLayoutEffect:
// - Measuring DOM
// - Synchronous DOM mutations
// - Preventing visual flicker

// Prefer useEffect unless you need synchronous behavior
```

### Question 8: Rules of Hooks

**Expected Answer:**

```javascript
// Rule 1: Only call hooks at top level
function Component() {
  // WRONG: Conditional hook
  if (condition) {
    useState(0); // Error!
  }
  
  // WRONG: Hook in loop
  for (let i = 0; i < 10; i++) {
    useState(i); // Error!
  }
  
  // RIGHT: Always call in same order
  const [count, setCount] = useState(0);
  const [name, setName] = useState('');
  
  if (condition) {
    // Use state here
  }
}

// Rule 2: Only call hooks in React functions
function regularFunction() {
  useState(0); // Error! Not a React function
}

function Component() {
  useState(0); // OK: Function component
}

function useCustomHook() {
  useState(0); // OK: Custom hook
}

class ClassComponent extends React.Component {
  componentDidMount() {
    useState(0); // Error! Class component
  }
}

// Why these rules exist:
// React tracks hooks by call order
// Breaking order breaks React's hook tracking
```

## Custom Hooks

### Question 9: Creating Custom Hooks

**Expected Answer:**

```javascript
// Custom hook for fetching data
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    const fetchData = async () => {
      try {
        setLoading(true);
        const response = await fetch(url);
        const json = await response.json();
        setData(json);
      } catch (err) {
        setError(err);
      } finally {
        setLoading(false);
      }
    };
    
    fetchData();
  }, [url]);
  
  return { data, loading, error };
}

// Usage
function Component() {
  const { data, loading, error } = useFetch('/api/users');
  
  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  return <div>{JSON.stringify(data)}</div>;
}

// Custom hook for local storage
function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      return initialValue;
    }
  });
  
  const setStoredValue = (value) => {
    try {
      setValue(value);
      window.localStorage.setItem(key, JSON.stringify(value));
    } catch (error) {
      console.error(error);
    }
  };
  
  return [value, setStoredValue];
}

// Custom hook for previous value
function usePrevious(value) {
  const ref = useRef();
  
  useEffect(() => {
    ref.current = value;
  }, [value]);
  
  return ref.current;
}

// Custom hook for debounce
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
      // API call
      searchAPI(debouncedSearchTerm);
    }
  }, [debouncedSearchTerm]);
  
  return <input value={searchTerm} onChange={e => setSearchTerm(e.target.value)} />;
}
```

### Question 10: Common Hook Patterns

**Expected Answer:**

```javascript
// Pattern 1: useToggle
function useToggle(initialValue = false) {
  const [value, setValue] = useState(initialValue);
  const toggle = useCallback(() => setValue(v => !v), []);
  return [value, toggle];
}

// Pattern 2: useAsync
function useAsync(asyncFunction, immediate = true) {
  const [status, setStatus] = useState('idle');
  const [value, setValue] = useState(null);
  const [error, setError] = useState(null);
  
  const execute = useCallback(async (...args) => {
    setStatus('pending');
    setValue(null);
    setError(null);
    
    try {
      const response = await asyncFunction(...args);
      setValue(response);
      setStatus('success');
    } catch (error) {
      setError(error);
      setStatus('error');
    }
  }, [asyncFunction]);
  
  useEffect(() => {
    if (immediate) {
      execute();
    }
  }, [execute, immediate]);
  
  return { execute, status, value, error };
}

// Pattern 3: useEventListener
function useEventListener(eventName, handler, element = window) {
  const savedHandler = useRef();
  
  useEffect(() => {
    savedHandler.current = handler;
  }, [handler]);
  
  useEffect(() => {
    const eventListener = (event) => savedHandler.current(event);
    element.addEventListener(eventName, eventListener);
    
    return () => {
      element.removeEventListener(eventName, eventListener);
    };
  }, [eventName, element]);
}

// Pattern 4: useOnClickOutside
function useOnClickOutside(ref, handler) {
  useEffect(() => {
    const listener = (event) => {
      if (!ref.current || ref.current.contains(event.target)) {
        return;
      }
      handler(event);
    };
    
    document.addEventListener('mousedown', listener);
    document.addEventListener('touchstart', listener);
    
    return () => {
      document.removeEventListener('mousedown', listener);
      document.removeEventListener('touchstart', listener);
    };
  }, [ref, handler]);
}
```

## What Interviewers Look For

### 1. Core Hooks Mastery
- useState with functional updates
- useEffect with proper dependencies
- useContext for global state
- useReducer for complex state

### 2. Performance Understanding
- useCallback for memoized callbacks
- useMemo for expensive computations
- When to optimize, when not to

### 3. Advanced Usage
- useRef for mutable values
- useLayoutEffect timing
- Custom hooks creation
- Hook composition

### 4. Best Practices
- Rules of Hooks
- Dependency arrays
- Cleanup functions
- Hook naming (use prefix)

### 5. Common Patterns
- Data fetching
- Event listeners
- Local storage
- Previous values

## Key Takeaways

### Essential Hooks
1. **useState**: Component state
2. **useEffect**: Side effects and lifecycle
3. **useContext**: Consume context
4. **useReducer**: Complex state logic
5. **useCallback**: Memoize functions
6. **useMemo**: Memoize values
7. **useRef**: Mutable values, DOM access

### Rules of Hooks
1. **Top level only**: No conditionals, loops
2. **React functions only**: Components and custom hooks
3. **Same order**: Consistent across renders
4. **Name with "use"**: Custom hooks start with "use"

### Best Practices
1. **Functional updates**: When state depends on previous
2. **Dependency arrays**: Include all dependencies
3. **Cleanup functions**: Always clean up effects
4. **Separate concerns**: Multiple effects for different purposes
5. **ESLint plugin**: Use exhaustive-deps rule

### Interview Success Tips
1. **Explain useState closure**: How it captures state
2. **Dependency arrays**: Why they're needed
3. **Cleanup importance**: Memory leaks without it
4. **Custom hooks**: Show you can create abstractions
5. **Performance**: Discuss useCallback/useMemo appropriately
6. **Real examples**: Hooks you've built

Remember: Hooks enable functional components to have state and lifecycle features. Master the basics (useState, useEffect), understand performance hooks (useCallback, useMemo), and know when to create custom hooks for reusable logic.
