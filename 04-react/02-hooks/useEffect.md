# useEffect - Side Effects and Lifecycle

## Overview

`useEffect` is React's Hook for performing side effects in functional components. It combines the functionality of `componentDidMount`, `componentDidUpdate`, and `componentWillUnmount` from class components into a single unified API. Understanding `useEffect` is crucial for managing subscriptions, data fetching, DOM manipulation, and other side effects in modern React applications.

## Table of Contents

1. [Basic Syntax and Concepts](#basic-syntax-and-concepts)
2. [Effect Timing and Execution](#effect-timing-and-execution)
3. [Dependencies Array](#dependencies-array)
4. [Cleanup Functions](#cleanup-functions)
5. [Common Use Cases](#common-use-cases)
6. [Advanced Patterns](#advanced-patterns)
7. [Best Practices](#best-practices)
8. [Common Pitfalls](#common-pitfalls)

## Basic Syntax and Concepts

### What is a Side Effect?

Side effects are operations that affect something outside the component's scope:

```jsx
import { useEffect } from 'react';

function Component() {
  // This is NOT a side effect - pure calculation
  const doubled = count * 2;
  
  // These ARE side effects:
  useEffect(() => {
    // DOM manipulation
    document.title = `Count: ${count}`;
    
    // Data fetching
    fetch('/api/data').then(handleResponse);
    
    // Subscriptions
    const subscription = dataSource.subscribe(handleData);
    
    // Timers
    const timer = setTimeout(() => {}, 1000);
    
    // Local storage
    localStorage.setItem('key', value);
  });
}
```

### Basic Effect Syntax

```jsx
import { useState, useEffect } from 'react';

function BasicEffect() {
  const [count, setCount] = useState(0);
  
  // Effect runs after every render
  useEffect(() => {
    console.log('Effect ran!');
    document.title = `Count: ${count}`;
  });
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
    </div>
  );
}
```

### Mental Model

```jsx
function Component() {
  // 1. React renders the component
  const [state, setState] = useState(initialValue);
  
  // 2. JSX is returned and DOM is updated
  
  // 3. THEN effects run (after paint)
  useEffect(() => {
    console.log('Effect runs AFTER render');
  });
  
  return <div>{state}</div>;
}

// Execution order:
// 1. Component function executes
// 2. Return JSX
// 3. Browser paints screen
// 4. Effects execute
```

## Effect Timing and Execution

### Effect Execution Flow

```jsx
function EffectTiming() {
  const [count, setCount] = useState(0);
  
  console.log('1. Render phase');
  
  useEffect(() => {
    console.log('3. Effect runs (after paint)');
    
    return () => {
      console.log('2. Cleanup (before next effect or unmount)');
    };
  });
  
  console.log('1b. Still render phase');
  
  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  );
}

// First render:
// "1. Render phase"
// "1b. Still render phase"
// [Browser paints]
// "3. Effect runs (after paint)"

// On update:
// "1. Render phase"
// "1b. Still render phase"
// [Browser paints]
// "2. Cleanup (before next effect or unmount)"
// "3. Effect runs (after paint)"
```

### useEffect vs useLayoutEffect

```jsx
import { useEffect, useLayoutEffect, useState } from 'react';

function EffectComparison() {
  const [color, setColor] = useState('red');
  
  // Runs AFTER browser paint (asynchronous)
  useEffect(() => {
    console.log('useEffect runs');
    // User might see a flash if this changes DOM
  });
  
  // Runs BEFORE browser paint (synchronous)
  useLayoutEffect(() => {
    console.log('useLayoutEffect runs first');
    // Use for DOM measurements or preventing visual flashes
  });
  
  return <div style={{ color }}>{color}</div>;
}

// Use useLayoutEffect when:
function TooltipPosition() {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  const tooltipRef = useRef(null);
  
  useLayoutEffect(() => {
    // Measure DOM before paint to avoid flicker
    const rect = tooltipRef.current.getBoundingClientRect();
    
    if (rect.right > window.innerWidth) {
      setPosition({ x: window.innerWidth - rect.width, y: rect.y });
    }
  }, []);
  
  return (
    <div 
      ref={tooltipRef} 
      style={{ position: 'absolute', left: position.x, top: position.y }}
    >
      Tooltip
    </div>
  );
}
```

## Dependencies Array

### No Dependencies (Every Render)

```jsx
function RunsEveryRender() {
  const [count, setCount] = useState(0);
  const [text, setText] = useState('');
  
  // Runs after EVERY render
  useEffect(() => {
    console.log('Runs every time anything changes');
  }); // No dependencies array
  
  return (
    <>
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
      <input value={text} onChange={(e) => setText(e.target.value)} />
    </>
  );
}
```

### Empty Dependencies (Mount Only)

```jsx
function RunsOnce() {
  const [data, setData] = useState(null);
  
  // Runs only once after initial render (like componentDidMount)
  useEffect(() => {
    console.log('Runs only on mount');
    
    fetch('/api/data')
      .then(res => res.json())
      .then(setData);
    
    return () => {
      console.log('Cleanup only on unmount');
    };
  }, []); // Empty array = no dependencies
  
  return <div>{data ? JSON.stringify(data) : 'Loading...'}</div>;
}
```

### Specific Dependencies

```jsx
function RunsOnSpecificChanges() {
  const [count, setCount] = useState(0);
  const [userId, setUserId] = useState(1);
  const [theme, setTheme] = useState('light');
  
  // Runs when count changes
  useEffect(() => {
    document.title = `Count: ${count}`;
  }, [count]); // Only re-run if count changes
  
  // Runs when userId changes
  useEffect(() => {
    fetch(`/api/user/${userId}`)
      .then(res => res.json())
      .then(console.log);
  }, [userId]); // Only re-run if userId changes
  
  // Multiple dependencies
  useEffect(() => {
    console.log(`Count: ${count}, Theme: ${theme}`);
  }, [count, theme]); // Re-run if either changes
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
      <button onClick={() => setUserId(userId + 1)}>User: {userId}</button>
      <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
        Theme: {theme}
      </button>
    </div>
  );
}
```

### Dependency Rules and Exhaustive Deps

```jsx
// ❌ BAD: Missing dependencies
function MissingDependencies() {
  const [count, setCount] = useState(0);
  const [multiplier, setMultiplier] = useState(2);
  
  useEffect(() => {
    // Uses 'multiplier' but doesn't list it in deps
    console.log(count * multiplier);
  }, [count]); // ⚠️ ESLint warning: missing 'multiplier'
}

// ✅ GOOD: All dependencies included
function AllDependencies() {
  const [count, setCount] = useState(0);
  const [multiplier, setMultiplier] = useState(2);
  
  useEffect(() => {
    console.log(count * multiplier);
  }, [count, multiplier]); // ✅ All used values listed
}

// ✅ GOOD: Functions in dependencies
function FunctionDependencies() {
  const [count, setCount] = useState(0);
  
  // Wrap in useCallback to avoid recreating every render
  const logCount = useCallback(() => {
    console.log(count);
  }, [count]);
  
  useEffect(() => {
    logCount();
  }, [logCount]); // Function is stable between renders
}

// ✅ GOOD: Using functional updates to avoid dependencies
function FunctionalUpdates() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const timer = setInterval(() => {
      // Use functional update instead of depending on 'count'
      setCount(c => c + 1);
    }, 1000);
    
    return () => clearInterval(timer);
  }, []); // No dependencies needed!
  
  return <div>{count}</div>;
}
```

## Cleanup Functions

### Why Cleanup is Important

```jsx
import { useState, useEffect } from 'react';

// ❌ BAD: Memory leak - subscription never cleaned up
function MemoryLeak() {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    const subscription = dataSource.subscribe(setData);
    // Missing cleanup - subscription persists after unmount!
  }, []);
  
  return <div>{data}</div>;
}

// ✅ GOOD: Proper cleanup
function ProperCleanup() {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    const subscription = dataSource.subscribe(setData);
    
    // Cleanup function
    return () => {
      subscription.unsubscribe();
    };
  }, []);
  
  return <div>{data}</div>;
}
```

### Common Cleanup Patterns

```jsx
// Event listeners
function EventListenerCleanup() {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  
  useEffect(() => {
    const handleMouseMove = (e) => {
      setPosition({ x: e.clientX, y: e.clientY });
    };
    
    window.addEventListener('mousemove', handleMouseMove);
    
    return () => {
      window.removeEventListener('mousemove', handleMouseMove);
    };
  }, []);
  
  return <div>Mouse: {position.x}, {position.y}</div>;
}

// Timers
function TimerCleanup() {
  const [seconds, setSeconds] = useState(0);
  
  useEffect(() => {
    const interval = setInterval(() => {
      setSeconds(s => s + 1);
    }, 1000);
    
    return () => {
      clearInterval(interval);
    };
  }, []);
  
  return <div>Seconds: {seconds}</div>;
}

// Async operations (cancellation)
function AsyncCleanup() {
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    let cancelled = false;
    
    async function fetchData() {
      try {
        const response = await fetch('/api/data');
        const result = await response.json();
        
        // Only update if not cancelled
        if (!cancelled) {
          setData(result);
        }
      } catch (err) {
        if (!cancelled) {
          setError(err);
        }
      }
    }
    
    fetchData();
    
    return () => {
      cancelled = true;
    };
  }, []);
  
  if (error) return <div>Error: {error.message}</div>;
  if (!data) return <div>Loading...</div>;
  return <div>{JSON.stringify(data)}</div>;
}

// AbortController for fetch
function AbortControllerCleanup() {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    const controller = new AbortController();
    
    fetch('/api/data', { signal: controller.signal })
      .then(res => res.json())
      .then(setData)
      .catch(err => {
        if (err.name !== 'AbortError') {
          console.error(err);
        }
      });
    
    return () => {
      controller.abort();
    };
  }, []);
  
  return <div>{data ? JSON.stringify(data) : 'Loading...'}</div>;
}
```

## Common Use Cases

### Data Fetching

```jsx
function DataFetching({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    let cancelled = false;
    
    async function fetchUser() {
      try {
        setLoading(true);
        setError(null);
        
        const response = await fetch(`/api/users/${userId}`);
        if (!response.ok) throw new Error('Failed to fetch');
        
        const data = await response.json();
        
        if (!cancelled) {
          setUser(data);
          setLoading(false);
        }
      } catch (err) {
        if (!cancelled) {
          setError(err.message);
          setLoading(false);
        }
      }
    }
    
    fetchUser();
    
    return () => {
      cancelled = true;
    };
  }, [userId]); // Re-fetch when userId changes
  
  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  return <div>User: {user.name}</div>;
}
```

### DOM Manipulation

```jsx
function DOMManipulation() {
  const [isDark, setIsDark] = useState(false);
  
  useEffect(() => {
    // Manipulate document outside React
    document.body.className = isDark ? 'dark-mode' : 'light-mode';
    
    // Cleanup
    return () => {
      document.body.className = '';
    };
  }, [isDark]);
  
  return (
    <button onClick={() => setIsDark(!isDark)}>
      Toggle {isDark ? 'Light' : 'Dark'} Mode
    </button>
  );
}

// Dynamic script loading
function ScriptLoader({ src }) {
  const [loaded, setLoaded] = useState(false);
  
  useEffect(() => {
    const script = document.createElement('script');
    script.src = src;
    script.async = true;
    
    script.onload = () => setLoaded(true);
    
    document.body.appendChild(script);
    
    return () => {
      document.body.removeChild(script);
    };
  }, [src]);
  
  return loaded ? <div>Script loaded!</div> : <div>Loading script...</div>;
}
```

### Subscriptions

```jsx
// WebSocket connection
function WebSocketComponent() {
  const [messages, setMessages] = useState([]);
  const [connected, setConnected] = useState(false);
  
  useEffect(() => {
    const ws = new WebSocket('ws://localhost:8080');
    
    ws.onopen = () => {
      setConnected(true);
      console.log('Connected');
    };
    
    ws.onmessage = (event) => {
      setMessages(prev => [...prev, event.data]);
    };
    
    ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };
    
    ws.onclose = () => {
      setConnected(false);
      console.log('Disconnected');
    };
    
    // Cleanup: close connection
    return () => {
      ws.close();
    };
  }, []);
  
  return (
    <div>
      <div>Status: {connected ? 'Connected' : 'Disconnected'}</div>
      <ul>
        {messages.map((msg, i) => (
          <li key={i}>{msg}</li>
        ))}
      </ul>
    </div>
  );
}

// Observable subscription
function ObservableSubscription({ observable }) {
  const [value, setValue] = useState(null);
  
  useEffect(() => {
    const subscription = observable.subscribe({
      next: (val) => setValue(val),
      error: (err) => console.error(err),
      complete: () => console.log('Complete')
    });
    
    return () => {
      subscription.unsubscribe();
    };
  }, [observable]);
  
  return <div>Value: {value}</div>;
}
```

### Local Storage Sync

```jsx
function LocalStorageSync({ storageKey }) {
  const [value, setValue] = useState(() => {
    // Initialize from localStorage
    const saved = localStorage.getItem(storageKey);
    return saved ? JSON.parse(saved) : '';
  });
  
  // Sync to localStorage when value changes
  useEffect(() => {
    localStorage.setItem(storageKey, JSON.stringify(value));
  }, [value, storageKey]);
  
  // Listen for changes in other tabs
  useEffect(() => {
    const handleStorageChange = (e) => {
      if (e.key === storageKey && e.newValue) {
        setValue(JSON.parse(e.newValue));
      }
    };
    
    window.addEventListener('storage', handleStorageChange);
    
    return () => {
      window.removeEventListener('storage', handleStorageChange);
    };
  }, [storageKey]);
  
  return (
    <input
      value={value}
      onChange={(e) => setValue(e.target.value)}
      placeholder="Type something..."
    />
  );
}
```

## Advanced Patterns

### Custom useEffect Hook Patterns

```jsx
// useEffectOnce - run only on mount
function useEffectOnce(effect) {
  useEffect(effect, []); // eslint-disable-line react-hooks/exhaustive-deps
}

// useUpdateEffect - skip first render
function useUpdateEffect(effect, deps) {
  const isFirstRender = useRef(true);
  
  useEffect(() => {
    if (isFirstRender.current) {
      isFirstRender.current = false;
      return;
    }
    
    return effect();
  }, deps); // eslint-disable-line react-hooks/exhaustive-deps
}

// useDebounceEffect - debounced effect
function useDebounceEffect(effect, deps, delay = 500) {
  useEffect(() => {
    const handler = setTimeout(() => {
      effect();
    }, delay);
    
    return () => {
      clearTimeout(handler);
    };
  }, [...deps, delay]); // eslint-disable-line react-hooks/exhaustive-deps
}

// Usage
function SearchComponent() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  
  useDebounceEffect(() => {
    if (query) {
      fetch(`/api/search?q=${query}`)
        .then(res => res.json())
        .then(setResults);
    }
  }, [query], 500);
  
  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <ul>{results.map(r => <li key={r.id}>{r.name}</li>)}</ul>
    </>
  );
}
```

### Effect Dependencies with Objects and Arrays

```jsx
// ❌ BAD: Object/array literal causes infinite loop
function BadObjectDep() {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    fetchData({ page: 1, limit: 10 }); // New object every render!
  }, [{ page: 1, limit: 10 }]); // New reference every render = infinite loop!
}

// ✅ GOOD: Stable object reference
function GoodObjectDep() {
  const [data, setData] = useState(null);
  const [page, setPage] = useState(1);
  const limit = 10;
  
  useEffect(() => {
    fetchData({ page, limit });
  }, [page, limit]); // Primitive values are compared by value
}

// ✅ GOOD: useMemo for stable reference
function MemoizedObjectDep() {
  const [page, setPage] = useState(1);
  
  const options = useMemo(() => ({
    page,
    limit: 10
  }), [page]);
  
  useEffect(() => {
    fetchData(options);
  }, [options]); // Stable reference when page doesn't change
}

// Deep comparison for complex objects
import { useEffect, useRef } from 'react';
import isEqual from 'lodash/isEqual';

function useDeepCompareEffect(effect, deps) {
  const ref = useRef(undefined);
  
  if (!isEqual(deps, ref.current)) {
    ref.current = deps;
  }
  
  useEffect(effect, [ref.current]); // eslint-disable-line react-hooks/exhaustive-deps
}

function DeepCompareExample({ user }) {
  useDeepCompareEffect(() => {
    console.log('User changed:', user);
  }, [user]); // Deep compares user object
  
  return <div>{user.name}</div>;
}
```

### Conditional Effects

```jsx
function ConditionalEffects({ shouldFetch, userId }) {
  const [data, setData] = useState(null);
  
  // Pattern 1: Early return
  useEffect(() => {
    if (!shouldFetch) return;
    
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(setData);
  }, [shouldFetch, userId]);
  
  // Pattern 2: Separate effects for different conditions
  useEffect(() => {
    if (userId) {
      fetch(`/api/users/${userId}`)
        .then(res => res.json())
        .then(setData);
    }
  }, [userId]);
  
  useEffect(() => {
    if (!userId) {
      setData(null);
    }
  }, [userId]);
  
  return <div>{data?.name || 'No user'}</div>;
}
```

### Effect Composition

```jsx
// Combining multiple effects
function MultipleEffects() {
  const [count, setCount] = useState(0);
  const [user, setUser] = useState(null);
  
  // Effect 1: Update document title
  useEffect(() => {
    document.title = `Count: ${count}`;
  }, [count]);
  
  // Effect 2: Fetch user data
  useEffect(() => {
    fetch('/api/user')
      .then(res => res.json())
      .then(setUser);
  }, []);
  
  // Effect 3: Log analytics
  useEffect(() => {
    analytics.track('count_changed', { count });
  }, [count]);
  
  return (
    <div>
      <p>Count: {count}</p>
      <p>User: {user?.name}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}

// Extract to custom hooks
function useDocumentTitle(title) {
  useEffect(() => {
    document.title = title;
  }, [title]);
}

function useUser() {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    fetch('/api/user')
      .then(res => res.json())
      .then(setUser);
  }, []);
  
  return user;
}

function useAnalytics(event, data) {
  useEffect(() => {
    analytics.track(event, data);
  }, [event, JSON.stringify(data)]); // Serialize object for comparison
}

// Cleaner component
function CleanerComponent() {
  const [count, setCount] = useState(0);
  const user = useUser();
  
  useDocumentTitle(`Count: ${count}`);
  useAnalytics('count_changed', { count });
  
  return (
    <div>
      <p>Count: {count}</p>
      <p>User: {user?.name}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}
```

## Best Practices

### 1. Keep Effects Focused

```jsx
// ❌ BAD: One effect doing too much
function BadMultiPurposeEffect({ userId, theme }) {
  useEffect(() => {
    // Fetching data
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(setUser);
    
    // Updating theme
    document.body.className = theme;
    
    // Setting up subscription
    const sub = dataSource.subscribe(handleData);
    
    // Timer
    const timer = setInterval(() => {}, 1000);
    
    return () => {
      sub.unsubscribe();
      clearInterval(timer);
    };
  }, [userId, theme]); // Effect re-runs when either changes
}

// ✅ GOOD: Separate, focused effects
function GoodSeparateEffects({ userId, theme }) {
  // Effect 1: Data fetching
  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(setUser);
  }, [userId]);
  
  // Effect 2: Theme
  useEffect(() => {
    document.body.className = theme;
    return () => {
      document.body.className = '';
    };
  }, [theme]);
  
  // Effect 3: Subscription
  useEffect(() => {
    const sub = dataSource.subscribe(handleData);
    return () => sub.unsubscribe();
  }, []);
  
  // Effect 4: Timer
  useEffect(() => {
    const timer = setInterval(() => {}, 1000);
    return () => clearInterval(timer);
  }, []);
}
```

### 2. Avoid Setting State from Props

```jsx
// ❌ BAD: Syncing state with props
function BadPropSync({ initialValue }) {
  const [value, setValue] = useState(initialValue);
  
  useEffect(() => {
    setValue(initialValue); // Dangerous pattern
  }, [initialValue]);
}

// ✅ GOOD: Use key to reset component
function GoodKeyReset({ userId }) {
  return <UserProfile key={userId} userId={userId} />;
}

// ✅ GOOD: Derive state instead
function GoodDerivedState({ items }) {
  // No effect needed - derive on each render
  const sortedItems = useMemo(() => {
    return [...items].sort((a, b) => a.name.localeCompare(b.name));
  }, [items]);
  
  return <ul>{sortedItems.map(item => <li key={item.id}>{item.name}</li>)}</ul>;
}

// ✅ GOOD: Fully controlled component
function GoodControlled({ value, onChange }) {
  // No internal state - parent controls everything
  return (
    <input
      value={value}
      onChange={(e) => onChange(e.target.value)}
    />
  );
}
```

### 3. Handle Race Conditions

```jsx
// ❌ BAD: Race condition
function BadRaceCondition({ userId }) {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    // If userId changes quickly, responses might arrive out of order
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(setUser); // Might set stale data
  }, [userId]);
}

// ✅ GOOD: Cancel previous request
function GoodCancellation({ userId }) {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    let cancelled = false;
    
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => {
        if (!cancelled) {
          setUser(data);
        }
      });
    
    return () => {
      cancelled = true;
    };
  }, [userId]);
}

// ✅ BETTER: Use AbortController
function BestAbortController({ userId }) {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    const controller = new AbortController();
    
    fetch(`/api/users/${userId}`, {
      signal: controller.signal
    })
      .then(res => res.json())
      .then(setUser)
      .catch(err => {
        if (err.name !== 'AbortError') {
          console.error(err);
        }
      });
    
    return () => {
      controller.abort();
    };
  }, [userId]);
}
```

### 4. Extract Custom Hooks

```jsx
// Custom hook for data fetching
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    const controller = new AbortController();
    
    async function fetchData() {
      try {
        setLoading(true);
        const response = await fetch(url, { signal: controller.signal });
        const result = await response.json();
        setData(result);
      } catch (err) {
        if (err.name !== 'AbortError') {
          setError(err);
        }
      } finally {
        setLoading(false);
      }
    }
    
    fetchData();
    
    return () => controller.abort();
  }, [url]);
  
  return { data, loading, error };
}

// Usage
function UserProfile({ userId }) {
  const { data: user, loading, error } = useFetch(`/api/users/${userId}`);
  
  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  return <div>{user.name}</div>;
}
```

## Common Pitfalls

### 1. Infinite Loops

```jsx
// ❌ BAD: Infinite loop
function InfiniteLoop() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    setCount(count + 1); // Changes state -> re-render -> effect runs -> changes state...
  }); // No dependency array!
}

// ❌ BAD: Object dependency
function ObjectDependency() {
  const [user, setUser] = useState({ name: 'John' });
  
  useEffect(() => {
    console.log(user);
  }, [user]); // Re-runs on every render if user object is recreated
}

// ✅ GOOD: Proper dependencies
function FixedLoop() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    if (count < 10) {
      setCount(count + 1);
    }
  }, [count]); // Include dependency
}
```

### 2. Stale Closures

```jsx
// ❌ BAD: Stale closure
function StaleCounter() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const timer = setInterval(() => {
      setCount(count + 1); // Always uses initial count value (0)
    }, 1000);
    
    return () => clearInterval(timer);
  }, []); // Empty deps = closure captures initial count
  
  return <div>{count}</div>; // Stuck at 1
}

// ✅ GOOD: Functional update
function FreshCounter() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const timer = setInterval(() => {
      setCount(c => c + 1); // Uses current count
    }, 1000);
    
    return () => clearInterval(timer);
  }, []); // No dependencies needed
  
  return <div>{count}</div>; // Counts correctly
}
```

### 3. Missing Cleanup

```jsx
// ❌ BAD: No cleanup
function MissingCleanup() {
  useEffect(() => {
    const timer = setInterval(() => {
      console.log('Running');
    }, 1000);
    // Timer keeps running after unmount!
  }, []);
}

// ✅ GOOD: Proper cleanup
function ProperCleanup() {
  useEffect(() => {
    const timer = setInterval(() => {
      console.log('Running');
    }, 1000);
    
    return () => {
      clearInterval(timer);
    };
  }, []);
}
```

### 4. Effect Dependencies Issues

```jsx
// ❌ BAD: Missing function dependency
function MissingFunctionDep() {
  const [count, setCount] = useState(0);
  
  const logCount = () => {
    console.log(count);
  };
  
  useEffect(() => {
    logCount(); // Uses logCount but doesn't list it
  }, []); // ⚠️ logCount changes every render
}

// ✅ GOOD: Include function in deps and memoize
function GoodFunctionDep() {
  const [count, setCount] = useState(0);
  
  const logCount = useCallback(() => {
    console.log(count);
  }, [count]);
  
  useEffect(() => {
    logCount();
  }, [logCount]); // ✅ Stable reference
}

// ✅ BETTER: Move function inside effect
function BestFunctionDep() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const logCount = () => {
      console.log(count);
    };
    
    logCount();
  }, [count]); // ✅ Only depends on what it uses
}
```

## Summary

### Key Takeaways

1. **Purpose**: Use `useEffect` for side effects (data fetching, subscriptions, DOM manipulation)
2. **Timing**: Effects run after render and paint
3. **Dependencies**: Always include all used values in dependency array
4. **Cleanup**: Return cleanup function for subscriptions, timers, listeners
5. **Separation**: Split unrelated effects into separate `useEffect` calls
6. **Custom Hooks**: Extract reusable effect logic into custom hooks

### Quick Reference

```jsx
// Run on every render
useEffect(() => {
  // effect
});

// Run once on mount
useEffect(() => {
  // effect
  return () => {
    // cleanup on unmount
  };
}, []);

// Run when dependencies change
useEffect(() => {
  // effect
  return () => {
    // cleanup before next effect
  };
}, [dep1, dep2]);

// Synchronous effect (before paint)
useLayoutEffect(() => {
  // effect
}, [deps]);
```

### ESLint Rule

Enable `eslint-plugin-react-hooks` to catch dependency issues:

```json
{
  "plugins": ["react-hooks"],
  "rules": {
    "react-hooks/rules-of-hooks": "error",
    "react-hooks/exhaustive-deps": "warn"
  }
}
```

`useEffect` is fundamental to React functional components and understanding its nuances is essential for building robust applications.
