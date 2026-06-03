# useSyncExternalStore

## Overview

`useSyncExternalStore` is a React 18+ hook designed to **subscribe to external stores** in a way that's safe with concurrent rendering and SSR. It prevents "tearing" (inconsistent UI due to interrupted renders) and ensures server/client consistency.

```jsx
const state = useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot?);
```

## Why It Exists

### The Problem: Tearing in Concurrent React

In React 18+, rendering can be interrupted and resumed. This creates a problem with external stores:

```jsx
// External store (not React state)
let count = 0;
const listeners = new Set();

const store = {
  getCount: () => count,
  increment: () => {
    count++;
    listeners.forEach(fn => fn());
  },
  subscribe: (fn) => {
    listeners.add(fn);
    return () => listeners.delete(fn);
  }
};

// Unsafe component (DON'T DO THIS)
function Counter() {
  const [count, setCount] = useState(store.getCount());
  
  useEffect(() => {
    return store.subscribe(() => {
      setCount(store.getCount());
    });
  }, []);
  
  return <div>{count}</div>;
}

// Problem: During concurrent rendering, React might:
// 1. Start rendering with count=0
// 2. Get interrupted
// 3. Store updates to count=1
// 4. Resume rendering with count=1
// 5. Now some components show 0, others show 1 → TEARING!
```

### The Solution: useSyncExternalStore

```jsx
function Counter() {
  const count = useSyncExternalStore(
    store.subscribe,
    store.getCount
  );
  
  return <div>{count}</div>;
}

// React guarantees all components see the same snapshot
// No tearing possible!
```

## Syntax

```jsx
const snapshot = useSyncExternalStore(
  subscribe,        // Function to subscribe to store
  getSnapshot,      // Function to get current snapshot
  getServerSnapshot? // Optional: snapshot for SSR
);
```

### Parameters

**subscribe**: `(onStoreChange: () => void) => () => void`
- Called once when component mounts
- Should subscribe to store and call `onStoreChange` when store changes
- Must return cleanup function

**getSnapshot**: `() => Snapshot`
- Returns current snapshot of store
- Called during render
- Must return immutable value
- If return value changes (by `Object.is`), component re-renders

**getServerSnapshot**: `() => Snapshot` (optional)
- Used during SSR
- Should return initial snapshot
- Must match server-rendered HTML

## Basic Examples

### Example 1: Simple Counter Store

```jsx
// Store implementation
function createStore(initialState) {
  let state = initialState;
  const listeners = new Set();
  
  return {
    getState: () => state,
    setState: (newState) => {
      state = newState;
      listeners.forEach(fn => fn());
    },
    subscribe: (listener) => {
      listeners.add(listener);
      return () => listeners.delete(listener);
    }
  };
}

const counterStore = createStore(0);

// Component using the store
function Counter() {
  const count = useSyncExternalStore(
    counterStore.subscribe,
    counterStore.getState
  );
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => counterStore.setState(count + 1)}>
        Increment
      </button>
    </div>
  );
}
```

### Example 2: Browser API (Window Size)

```jsx
function getWindowSize() {
  return {
    width: window.innerWidth,
    height: window.innerHeight
  };
}

function subscribe(callback) {
  window.addEventListener('resize', callback);
  return () => window.removeEventListener('resize', callback);
}

function useWindowSize() {
  const size = useSyncExternalStore(
    subscribe,
    getWindowSize,
    () => ({ width: 0, height: 0 }) // SSR fallback
  );
  
  return size;
}

// Usage
function Component() {
  const { width, height } = useWindowSize();
  return <div>Window: {width} × {height}</div>;
}
```

### Example 3: Online Status

```jsx
function getOnlineStatus() {
  return navigator.onLine;
}

function subscribe(callback) {
  window.addEventListener('online', callback);
  window.addEventListener('offline', callback);
  
  return () => {
    window.removeEventListener('online', callback);
    window.removeEventListener('offline', callback);
  };
}

function useOnlineStatus() {
  const isOnline = useSyncExternalStore(
    subscribe,
    getOnlineStatus,
    () => true // Assume online during SSR
  );
  
  return isOnline;
}

// Usage
function StatusIndicator() {
  const isOnline = useOnlineStatus();
  
  return (
    <div className={isOnline ? 'online' : 'offline'}>
      {isOnline ? '🟢 Online' : '🔴 Offline'}
    </div>
  );
}
```

## Advanced Examples

### Example 4: Redux-Like Store

```jsx
function createReduxLikeStore(reducer, initialState) {
  let state = initialState;
  const listeners = new Set();
  
  const getState = () => state;
  
  const dispatch = (action) => {
    state = reducer(state, action);
    listeners.forEach(fn => fn());
  };
  
  const subscribe = (listener) => {
    listeners.add(listener);
    return () => listeners.delete(listener);
  };
  
  return { getState, dispatch, subscribe };
}

// Reducer
function counterReducer(state = 0, action) {
  switch (action.type) {
    case 'INCREMENT': return state + 1;
    case 'DECREMENT': return state - 1;
    case 'RESET': return 0;
    default: return state;
  }
}

const store = createReduxLikeStore(counterReducer, 0);

// Hook to use store
function useStore() {
  const state = useSyncExternalStore(
    store.subscribe,
    store.getState
  );
  
  return [state, store.dispatch];
}

// Usage
function Counter() {
  const [count, dispatch] = useStore();
  
  return (
    <>
      <p>{count}</p>
      <button onClick={() => dispatch({ type: 'INCREMENT' })}>+</button>
      <button onClick={() => dispatch({ type: 'DECREMENT' })}>-</button>
      <button onClick={() => dispatch({ type: 'RESET' })}>Reset</button>
    </>
  );
}
```

### Example 5: Store with Selectors

```jsx
function createStore(initialState) {
  let state = initialState;
  const listeners = new Set();
  
  return {
    getState: () => state,
    setState: (partial) => {
      state = { ...state, ...partial };
      listeners.forEach(fn => fn());
    },
    subscribe: (listener) => {
      listeners.add(listener);
      return () => listeners.delete(listener);
    }
  };
}

const store = createStore({
  user: { name: 'John', age: 30 },
  todos: [],
  theme: 'light'
});

// Hook with selector
function useStoreSelector(selector) {
  const snapshot = useSyncExternalStore(
    store.subscribe,
    () => selector(store.getState())
  );
  
  return snapshot;
}

// Usage
function UserName() {
  // Only re-renders when user.name changes
  const name = useStoreSelector(state => state.user.name);
  return <div>Hello, {name}</div>;
}

function TodoCount() {
  // Only re-renders when todos.length changes
  const count = useStoreSelector(state => state.todos.length);
  return <div>{count} todos</div>;
}
```

### Example 6: Subscription with Selector

```jsx
function useStoreWithSelector(store, selector) {
  // Wrap selector to maintain referential stability
  const getSnapshot = useCallback(
    () => selector(store.getState()),
    [store, selector]
  );
  
  const snapshot = useSyncExternalStore(
    store.subscribe,
    getSnapshot
  );
  
  return snapshot;
}

// Better: Memoize selector
function useSelector(selector) {
  const memoizedSelector = useCallback(selector, []);
  
  return useSyncExternalStore(
    store.subscribe,
    () => memoizedSelector(store.getState())
  );
}
```

### Example 7: URL State

```jsx
function getCurrentURL() {
  return window.location.href;
}

function subscribe(callback) {
  window.addEventListener('popstate', callback);
  window.addEventListener('hashchange', callback);
  
  return () => {
    window.removeEventListener('popstate', callback);
    window.removeEventListener('hashchange', callback);
  };
}

function useURL() {
  const url = useSyncExternalStore(
    subscribe,
    getCurrentURL,
    () => '' // No URL during SSR
  );
  
  return url;
}

// Usage
function URLDisplay() {
  const url = useURL();
  return <div>Current URL: {url}</div>;
}
```

### Example 8: Local Storage Sync

```jsx
function createLocalStorageStore(key, initialValue) {
  const listeners = new Set();
  
  const getSnapshot = () => {
    try {
      const item = localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  };
  
  const setValue = (value) => {
    try {
      localStorage.setItem(key, JSON.stringify(value));
      listeners.forEach(fn => fn());
    } catch (error) {
      console.error('Failed to save to localStorage', error);
    }
  };
  
  const subscribe = (listener) => {
    listeners.add(listener);
    
    // Also listen to storage events (for cross-tab sync)
    const storageListener = (e) => {
      if (e.key === key) {
        listener();
      }
    };
    window.addEventListener('storage', storageListener);
    
    return () => {
      listeners.delete(listener);
      window.removeEventListener('storage', storageListener);
    };
  };
  
  return { getSnapshot, setValue, subscribe };
}

function useLocalStorage(key, initialValue) {
  const store = useMemo(
    () => createLocalStorageStore(key, initialValue),
    [key, initialValue]
  );
  
  const value = useSyncExternalStore(
    store.subscribe,
    store.getSnapshot,
    () => initialValue // SSR fallback
  );
  
  return [value, store.setValue];
}

// Usage
function ThemeToggle() {
  const [theme, setTheme] = useLocalStorage('theme', 'light');
  
  return (
    <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
      Current: {theme}
    </button>
  );
}
```

## SSR Considerations

### Why getServerSnapshot Is Needed

```jsx
// Without getServerSnapshot
function BrokenComponent() {
  const width = useSyncExternalStore(
    subscribe,
    () => window.innerWidth  // Error on server! No window object
  );
  
  return <div>{width}px</div>;
}

// With getServerSnapshot
function WorkingComponent() {
  const width = useSyncExternalStore(
    subscribe,
    () => window.innerWidth,
    () => 1200  // Default width for SSR
  );
  
  return <div>{width}px</div>;
}
```

### Hydration Mismatch Warning

```jsx
// Server renders with getServerSnapshot
// <div>1200px</div>

// Client hydrates with getSnapshot
// <div>800px</div>  ← Mismatch!

// React will show warning but fix it after hydration
```

### Best Practice: Suppress Hydration Warning

```jsx
function useClientWidth() {
  const width = useSyncExternalStore(
    subscribe,
    () => window.innerWidth,
    () => null  // null during SSR
  );
  
  // Don't render width on first client render
  if (width === null) {
    return null;
  }
  
  return <div suppressHydrationWarning>{width}px</div>;
}
```

## TypeScript

```tsx
import { useSyncExternalStore } from 'react';

interface Store<T> {
  getState: () => T;
  setState: (value: T | ((prev: T) => T)) => void;
  subscribe: (listener: () => void) => () => void;
}

function createStore<T>(initialState: T): Store<T> {
  let state = initialState;
  const listeners = new Set<() => void>();
  
  return {
    getState: () => state,
    setState: (value) => {
      state = typeof value === 'function' 
        ? (value as (prev: T) => T)(state) 
        : value;
      listeners.forEach(fn => fn());
    },
    subscribe: (listener) => {
      listeners.add(listener);
      return () => listeners.delete(listener);
    }
  };
}

// Typed hook
function useStore<T>(store: Store<T>): T {
  return useSyncExternalStore(
    store.subscribe,
    store.getState
  );
}

// With selector
function useStoreSelector<T, U>(
  store: Store<T>,
  selector: (state: T) => U
): U {
  return useSyncExternalStore(
    store.subscribe,
    () => selector(store.getState())
  );
}
```

## Common Patterns

### Pattern 1: Memoized Snapshot

```jsx
function useOptimizedStore(store, selector) {
  // Memoize to avoid creating new snapshot every render
  const getSnapshot = useMemo(
    () => () => selector(store.getState()),
    [store, selector]
  );
  
  return useSyncExternalStore(
    store.subscribe,
    getSnapshot
  );
}
```

### Pattern 2: Conditional Subscription

```jsx
function useConditionalStore(store, shouldSubscribe) {
  const subscribe = useMemo(
    () => shouldSubscribe 
      ? store.subscribe 
      : () => () => {},  // No-op subscription
    [store, shouldSubscribe]
  );
  
  return useSyncExternalStore(
    subscribe,
    store.getState
  );
}
```

### Pattern 3: Multiple Stores

```jsx
function useMultipleStores() {
  const userStore = useSyncExternalStore(
    userStoreObj.subscribe,
    userStoreObj.getState
  );
  
  const todoStore = useSyncExternalStore(
    todoStoreObj.subscribe,
    todoStoreObj.getState
  );
  
  return { user: userStore, todos: todoStore };
}
```

## Performance Considerations

### Shallow Comparison Problem

```jsx
// Bad: New object every time
function BadComponent() {
  const size = useSyncExternalStore(
    subscribe,
    () => ({ 
      width: window.innerWidth,
      height: window.innerHeight 
    })  // New object every render!
  );
  
  // Re-renders on every parent render, even if size unchanged
}

// Good: Cache object
let cachedSize = null;
function getSize() {
  const newSize = {
    width: window.innerWidth,
    height: window.innerHeight
  };
  
  if (!cachedSize || 
      cachedSize.width !== newSize.width || 
      cachedSize.height !== newSize.height) {
    cachedSize = newSize;
  }
  
  return cachedSize;
}

function GoodComponent() {
  const size = useSyncExternalStore(subscribe, getSize);
  // Only re-renders when size actually changes
}
```

### Selector Optimization

```jsx
// Bad: Expensive selector
function ExpensiveComponent() {
  const result = useSyncExternalStore(
    store.subscribe,
    () => {
      // This runs on EVERY render!
      return store.getState().items
        .filter(x => x.active)
        .map(x => x.value)
        .reduce((a, b) => a + b, 0);
    }
  );
}

// Good: Memoize in store
const selectTotal = createSelector(
  state => state.items,
  items => items
    .filter(x => x.active)
    .map(x => x.value)
    .reduce((a, b) => a + b, 0)
);

function OptimizedComponent() {
  const result = useSyncExternalStore(
    store.subscribe,
    () => selectTotal(store.getState())
  );
}
```

## Common Mistakes

### Mistake 1: Not Returning Cleanup

```jsx
// Bad: No cleanup
function BadComponent() {
  const value = useSyncExternalStore(
    (callback) => {
      store.subscribe(callback);
      // Missing return!
    },
    store.getState
  );
}

// Good: Return cleanup
function GoodComponent() {
  const value = useSyncExternalStore(
    (callback) => {
      const unsubscribe = store.subscribe(callback);
      return unsubscribe;
    },
    store.getState
  );
}
```

### Mistake 2: Mutating Snapshot

```jsx
// Bad: Mutating snapshot
const getSnapshot = () => {
  const state = store.getState();
  state.newProperty = 'value';  // Mutation!
  return state;
};

// Good: Return immutable snapshot
const getSnapshot = () => {
  return store.getState();  // Don't modify
};
```

### Mistake 3: Async getSnapshot

```jsx
// Bad: Async snapshot
const value = useSyncExternalStore(
  subscribe,
  async () => await fetchData()  // Can't be async!
);

// Good: Sync snapshot
const value = useSyncExternalStore(
  subscribe,
  () => cache.get('data') ?? null  // Must be synchronous
);
```

## When to Use

### Good Use Cases

1. **Browser APIs**: window size, online status, media queries
2. **Third-party stores**: Redux, MobX, Zustand (if not using their hooks)
3. **External subscriptions**: WebSocket data, EventEmitter
4. **Cross-tab sync**: localStorage, BroadcastChannel
5. **Custom stores**: When building your own state management

### When NOT to Use

1. **React state**: Use useState/useReducer
2. **Props/context**: Use regular React patterns
3. **Derived state**: Use useMemo
4. **Server data**: Use data fetching libraries (React Query, SWR)

## Resources

- [React Docs: useSyncExternalStore](https://react.dev/reference/react/useSyncExternalStore)
- [useSyncExternalStore First Look](https://github.com/reactwg/react-18/discussions/86)
- [Preventing Tearing](https://github.com/reactwg/react-18/discussions/69)
- [use-sync-external-store Shim](https://www.npmjs.com/package/use-sync-external-store) (for React 17)
