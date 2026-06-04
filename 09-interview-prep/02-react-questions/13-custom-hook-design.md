# Custom Hook Design

## Rules of Hooks (Why They Exist)

1. **Only call hooks at the top level** — no conditionals, loops, or early returns before hooks
2. **Only call hooks from React functions** — function components or other custom hooks

Reason: React stores hook state in a linked list per fiber, indexed by call order. Conditional calls would desync the indices between renders.

---

## Naming Convention

Always start with `use`. This isn't just convention — it's how React's linter knows to enforce Rules of Hooks inside the function:

```js
// ✅ Named correctly — linter enforces hook rules inside
function useWindowSize() { ... }

// ❌ Wrong — linter won't check hook rules inside
function getWindowSize() { ... }
```

---

## Anatomy of a Well-Designed Hook

```js
// useLocalStorage.ts
function useLocalStorage<T>(key: string, initialValue: T) {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setValue = useCallback((value: T | ((prev: T) => T)) => {
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

  return [storedValue, setValue] as const;
}
```

Good hook characteristics:
- Returns stable references with `useCallback`/`useMemo` where appropriate
- Handles errors internally (or exposes error state)
- `as const` for tuple return types (TypeScript)

---

## Composition Patterns

### Hooks Calling Hooks

```js
function useUser(id: string) {
  return useQuery({ queryKey: ['user', id], queryFn: () => fetchUser(id) });
}

function useUserPermissions(id: string) {
  const { data: user } = useUser(id);    // composed hook
  return useMemo(() => computePermissions(user), [user]);
}
```

### Extracting Event Handler Logic

```js
// Before — logic mixed into component
function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    if (!query) return;
    setLoading(true);
    searchAPI(query).then(r => { setResults(r); setLoading(false); });
  }, [query]);
  // ...
}

// After — logic in a hook
function useSearch() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    if (!query) return;
    let cancelled = false;
    setLoading(true);
    searchAPI(query).then(r => {
      if (!cancelled) { setResults(r); setLoading(false); }
    });
    return () => { cancelled = true; };
  }, [query]);

  return { query, setQuery, results, loading };
}
```

---

## Return Value Conventions

**Object** — when you have many named values (flexible, order doesn't matter):
```js
const { data, loading, error, refetch } = useQuery(...);
```

**Tuple** — when pairing a value with a setter/toggle (mirrors `useState`):
```js
const [isOpen, setIsOpen] = useToggle(false);
const [value, setValue] = useLocalStorage('key', 'default');
```

---

## Anti-Patterns

### Over-Abstraction
```js
// ❌ Too generic — this is just useState with extra steps
function useBoolean(initial) {
  const [v, setV] = useState(initial);
  return { value: v, setTrue: () => setV(true), setFalse: () => setV(false) };
}

// ✅ Only abstract when there's non-obvious logic
function useDisclosure() {
  const [isOpen, setIsOpen] = useState(false);
  const open = useCallback(() => setIsOpen(true), []);
  const close = useCallback(() => setIsOpen(false), []);
  const toggle = useCallback(() => setIsOpen(v => !v), []);
  return { isOpen, open, close, toggle }; // named actions are the value
}
```

### Accessing State from Outside
```js
// ❌ Breaks encapsulation
const globalRef = { current: null };
function useBadHook() {
  globalRef.current = useRef();
}
```

### Missing Cleanup
```js
// ❌ Memory leak
function useBadSubscription(id) {
  useEffect(() => {
    subscribe(id, handler);
    // Missing: return () => unsubscribe(id, handler);
  }, [id]);
}
```

---

## Common Interview Questions

**Q: What makes a good custom hook vs a regular utility function?**
A custom hook is appropriate when you need to use other hooks inside it (useState, useEffect, useRef, etc.). If your logic is pure functions with no hooks, a plain utility function is better.

**Q: How do you avoid stale closures in custom hooks?**
Use `useCallback` with proper dependency arrays, or use `useRef` to store the latest value of a callback without it being a dependency:
```js
const callbackRef = useRef(callback);
callbackRef.current = callback;
// Use callbackRef.current() to always call the latest version
```

**Q: Can custom hooks share state between components?**
By default, each component using the hook gets its own state instance. To share state, hoist it to a common ancestor or use a store (Context, Zustand, etc.).
