# useEffect Lifecycle

## Overview

useEffect is React's primary mechanism for handling side effects in function components. While its API appears simple, understanding its execution timing, cleanup behavior, and dependency system is crucial for writing correct and performant React applications.

This guide explores the complete lifecycle of useEffect, from when it's scheduled to when it executes, how cleanup works, dependency comparison mechanics, and common pitfalls.

## Effect Execution Timeline

Understanding when effects run relative to rendering and painting is essential:

```javascript
// Example 1: Complete render cycle with useEffect
function Component() {
  console.log('1. Render function executes');
  
  useEffect(() => {
    console.log('4. Effect executes');
    return () => console.log('5. Cleanup (on next effect or unmount)');
  });
  
  return <div>Component</div>;
}

// Timeline:
// 1. Render function executes (component renders)
// 2. React commits changes to DOM (mutation phase)
// 3. Browser paints screen
// 4. Effect executes (async, after paint)
// 5. On next render or unmount, cleanup runs before effect
```

```javascript
// Example 2: Detailed timing comparison
function TimingDemo() {
  const [count, setCount] = useState(0);
  
  console.log('A. Render starts');
  
  useLayoutEffect(() => {
    console.log('C. useLayoutEffect (before browser paint)');
    return () => console.log('C-cleanup');
  });
  
  useEffect(() => {
    console.log('D. useEffect (after browser paint)');
    return () => console.log('D-cleanup');
  });
  
  console.log('B. Render ends');
  
  return (
    <button onClick={() => setCount(count + 1)}>
      {count}
    </button>
  );
}

// First render output:
// A. Render starts
// B. Render ends
// [DOM mutation happens]
// C. useLayoutEffect (before browser paint)
// [Browser paints]
// D. useEffect (after browser paint)

// On update (click button):
// A. Render starts
// B. Render ends
// [DOM mutation happens]
// C-cleanup
// C. useLayoutEffect (before browser paint)
// [Browser paints]
// D-cleanup
// D. useEffect (after browser paint)
```

## Mount Phase

The first time a component renders:

```javascript
// Example 3: Mount behavior
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    console.log('Effect runs after initial render');
    
    fetchUser(userId).then(data => {
      setUser(data);
    });
    
    // No cleanup needed yet
  }, [userId]);
  
  if (!user) return <div>Loading...</div>;
  return <div>{user.name}</div>;
}

// Mount sequence:
// 1. Initial render (user = null)
// 2. Returns "Loading..."
// 3. Commits to DOM
// 4. Browser paints
// 5. Effect executes
// 6. fetchUser starts
// 7. Promise resolves
// 8. setUser triggers re-render
```

```javascript
// Example 4: Multiple effects on mount
function ComplexComponent() {
  useEffect(() => {
    console.log('Effect 1');
    return () => console.log('Cleanup 1');
  }, []);
  
  useEffect(() => {
    console.log('Effect 2');
    return () => console.log('Cleanup 2');
  }, []);
  
  useEffect(() => {
    console.log('Effect 3');
    return () => console.log('Cleanup 3');
  }, []);
  
  return <div>Component</div>;
}

// On mount:
// Effect 1
// Effect 2
// Effect 3

// On unmount:
// Cleanup 1
// Cleanup 2
// Cleanup 3
```

## Update Phase

When dependencies change:

```javascript
// Example 5: Update behavior
function SearchResults({ query }) {
  const [results, setResults] = useState([]);
  
  useEffect(() => {
    console.log(`Effect runs for query: ${query}`);
    
    const controller = new AbortController();
    
    fetch(`/api/search?q=${query}`, { signal: controller.signal })
      .then(res => res.json())
      .then(data => setResults(data))
      .catch(err => {
        if (err.name !== 'AbortError') {
          console.error(err);
        }
      });
    
    return () => {
      console.log(`Cleanup for query: ${query}`);
      controller.abort();
    };
  }, [query]);
  
  return <div>{results.length} results</div>;
}

// User types "react" → "reactjs" quickly:
// 
// First render (query="react"):
// Effect runs for query: react
// 
// Second render (query="reactjs"):
// Cleanup for query: react  (aborts first request)
// Effect runs for query: reactjs
```

```javascript
// Example 6: Selective effect execution
function MultiDependency({ userId, theme }) {
  const [user, setUser] = useState(null);
  
  // Effect 1: Only runs when userId changes
  useEffect(() => {
    console.log('Fetching user');
    fetchUser(userId).then(setUser);
  }, [userId]);
  
  // Effect 2: Only runs when theme changes
  useEffect(() => {
    console.log('Applying theme');
    document.body.className = theme;
  }, [theme]);
  
  // Effect 3: Runs when either changes
  useEffect(() => {
    console.log('Analytics event');
    trackEvent('view-profile', { userId, theme });
  }, [userId, theme]);
  
  return <div>{user?.name}</div>;
}

// Scenario: userId changes but theme doesn't
// Effect 1: RUNS (userId changed)
// Effect 2: SKIPS (theme unchanged)
// Effect 3: RUNS (userId changed)
```

## Cleanup Phase

Cleanup functions prevent memory leaks and stale effects:

```javascript
// Example 7: Subscription cleanup
function RealtimeData({ channelId }) {
  const [messages, setMessages] = useState([]);
  
  useEffect(() => {
    const subscription = subscribeToChannel(channelId, message => {
      setMessages(prev => [...prev, message]);
    });
    
    // Cleanup runs:
    // 1. Before next effect (if channelId changes)
    // 2. On component unmount
    return () => {
      subscription.unsubscribe();
    };
  }, [channelId]);
  
  return <MessageList messages={messages} />;
}

// Timeline when channelId changes from "general" to "random":
// 1. Component re-renders with channelId="random"
// 2. Commits to DOM
// 3. Cleanup runs (unsubscribes from "general")
// 4. New effect runs (subscribes to "random")
```

```javascript
// Example 8: Timer cleanup
function AutoSaveForm({ onSave }) {
  const [formData, setFormData] = useState({});
  
  useEffect(() => {
    // Set up auto-save timer
    const timerId = setInterval(() => {
      onSave(formData);
    }, 5000);
    
    // Clear timer on cleanup
    return () => {
      clearInterval(timerId);
    };
  }, [formData, onSave]);
  
  return <form>...</form>;
}

// Every time formData changes:
// 1. Cleanup runs (clears old timer)
// 2. New effect runs (sets new timer with current formData)
```

```javascript
// Example 9: Event listener cleanup
function WindowSize() {
  const [size, setSize] = useState({
    width: window.innerWidth,
    height: window.innerHeight
  });
  
  useEffect(() => {
    const handleResize = () => {
      setSize({
        width: window.innerWidth,
        height: window.innerHeight
      });
    };
    
    window.addEventListener('resize', handleResize);
    
    // Cleanup removes listener
    return () => {
      window.removeEventListener('resize', handleResize);
    };
  }, []); // Empty deps - only mount/unmount
  
  return <div>{size.width} x {size.height}</div>;
}
```

## Dependency Array Mechanics

How React compares dependencies:

```javascript
// Example 10: Object.is comparison
function DependencyComparison() {
  const [count, setCount] = useState(0);
  const [user] = useState({ name: 'John' });
  
  useEffect(() => {
    // This runs every render!
    console.log('Effect runs');
  }, [user]);
  
  // Problem: user reference never changes
  // Object.is(prevUser, user) is always true
  // But if user was recreated each render, it would be false
}

// Example 11: Primitive vs reference comparison
function ComparisonDemo() {
  const [count, setCount] = useState(0);
  const [obj, setObj] = useState({ value: 0 });
  
  useEffect(() => {
    console.log('Effect 1');
  }, [count]); // Primitive - compares value
  
  useEffect(() => {
    console.log('Effect 2');
  }, [obj]); // Object - compares reference
  
  const handleUpdate = () => {
    // This doesn't trigger Effect 2
    obj.value = 1; // Mutate object
    
    // This DOES trigger Effect 2
    setObj({ ...obj }); // New reference
  };
}

// Example 12: Dependency comparison implementation
function areHookInputsEqual(nextDeps, prevDeps) {
  // Handle null case
  if (prevDeps === null) return false;
  
  // Compare each dependency using Object.is
  for (let i = 0; i < prevDeps.length; i++) {
    // Object.is is like === but handles NaN and +0/-0 correctly
    if (Object.is(nextDeps[i], prevDeps[i])) {
      continue;
    }
    return false;
  }
  return true;
}

// Object.is behavior:
Object.is(5, 5);           // true
Object.is('hi', 'hi');     // true
Object.is(NaN, NaN);       // true (unlike ===)
Object.is(+0, -0);         // false (unlike ===)
Object.is({}, {});         // false (different references)

const obj = {};
Object.is(obj, obj);       // true (same reference)
```

## Empty Dependency Array

```javascript
// Example 13: Run once pattern
function Analytics() {
  useEffect(() => {
    // Runs only on mount
    initializeAnalytics();
    
    return () => {
      // Runs only on unmount
      cleanupAnalytics();
    };
  }, []); // Empty array - no dependencies
  
  return null;
}

// Example 14: Stale closure trap
function StaleClosureDemo() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const interval = setInterval(() => {
      // BUG: Always logs 0 (stale closure)
      console.log(count);
      
      // BUG: Always sets count to 1
      setCount(count + 1);
    }, 1000);
    
    return () => clearInterval(interval);
  }, []); // Empty deps - captures initial count (0)
  
  return <div>{count}</div>;
}

// Example 15: Fix with functional update
function FixedStaleClosureDemo() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const interval = setInterval(() => {
      // Use functional update to get current state
      setCount(c => {
        console.log(c); // Always current
        return c + 1;
      });
    }, 1000);
    
    return () => clearInterval(interval);
  }, []); // Safe now
  
  return <div>{count}</div>;
}
```

## No Dependency Array

Omitting the array runs the effect after every render:

```javascript
// Example 16: Effect on every render
function EveryRenderEffect() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    // Runs after EVERY render
    console.log('Rendered with count:', count);
    // Usually a mistake - causes performance issues
  }); // No dependency array
  
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}

// Example 17: When no deps is intentional
function Logger({ children }) {
  useEffect(() => {
    // Intentionally log every render
    console.log('Component rendered');
  });
  
  return children;
}
```

## Effect Dependencies Best Practices

```javascript
// Example 18: Include all dependencies
function SearchComponent() {
  const [query, setQuery] = useState('');
  const [filter, setFilter] = useState('all');
  
  // WRONG: Missing filter dependency
  useEffect(() => {
    fetchResults(query, filter);
  }, [query]); // ESLint warning!
  
  // RIGHT: All dependencies included
  useEffect(() => {
    fetchResults(query, filter);
  }, [query, filter]);
}

// Example 19: Extracting constants
function ConstantExtraction() {
  const API_URL = 'https://api.example.com'; // Constant - not a dependency
  
  useEffect(() => {
    fetch(API_URL).then(/* ... */);
  }, []); // Safe - API_URL never changes
}

// Example 20: Functions as dependencies
function FunctionDependencies({ onUpdate }) {
  const [data, setData] = useState(null);
  
  // PROBLEM: onUpdate might be new function each render
  useEffect(() => {
    fetchData().then(result => {
      setData(result);
      onUpdate(result);
    });
  }, [onUpdate]); // Effect runs if onUpdate reference changes
  
  // SOLUTION 1: Wrap parent's callback in useCallback
  // SOLUTION 2: Use ref to always have latest callback
  // SOLUTION 3: Move onUpdate call to event handler instead
}

// Example 21: useCallback to stabilize dependencies
function StableDependency() {
  const [count, setCount] = useState(0);
  
  // Memoize function to keep reference stable
  const logCount = useCallback(() => {
    console.log('Count:', count);
  }, [count]);
  
  useEffect(() => {
    const timer = setInterval(logCount, 1000);
    return () => clearInterval(timer);
  }, [logCount]); // Only re-run when logCount changes
}
```

## Advanced Patterns

```javascript
// Example 22: Custom cleanup logic
function DataFetcher({ id }) {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    let cancelled = false;
    
    async function fetchData() {
      const result = await fetch(`/api/data/${id}`);
      const json = await result.json();
      
      // Check if we should still update
      if (!cancelled) {
        setData(json);
      }
    }
    
    fetchData();
    
    return () => {
      cancelled = true; // Set flag
    };
  }, [id]);
  
  return <div>{data?.name}</div>;
}

// Example 23: Effect dependencies with objects
function ObjectDependency({ config }) {
  // PROBLEM: config is a new object every render
  useEffect(() => {
    applyConfig(config);
  }, [config]); // Runs every render
  
  // SOLUTION 1: Depend on specific properties
  useEffect(() => {
    applyConfig(config);
  }, [config.url, config.method]); // Only re-run if these change
  
  // SOLUTION 2: Use useMemo in parent
  // const config = useMemo(() => ({ url, method }), [url, method]);
}

// Example 24: Ref for latest value without re-running effect
function LatestValuePattern() {
  const [count, setCount] = useState(0);
  const countRef = useRef(count);
  
  // Keep ref in sync
  useEffect(() => {
    countRef.current = count;
  });
  
  useEffect(() => {
    const interval = setInterval(() => {
      // Always reads latest count without re-running effect
      console.log('Count:', countRef.current);
    }, 1000);
    
    return () => clearInterval(interval);
  }, []); // Empty deps safe now
  
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

## Common Patterns and Pitfalls

```javascript
// Example 25: Infinite loop
function InfiniteLoop() {
  const [data, setData] = useState([]);
  
  useEffect(() => {
    // INFINITE LOOP!
    setData([...data, 'item']); // Updates state
    // State update triggers re-render
    // Re-render triggers effect
    // Effect updates state... forever
  }, [data]);
}

// Example 26: Missing cleanup causing memory leak
function MemoryLeak() {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    const subscription = subscribeToData(newData => {
      setData(newData);
    });
    
    // MISSING: return () => subscription.unsubscribe();
    
    // When component unmounts, subscription still calls setData
    // Causes "Can't perform a React state update on an unmounted component"
  }, []);
}

// Example 27: Race condition
function RaceCondition({ userId }) {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    // Race condition: responses may arrive out of order
    fetchUser(userId).then(data => {
      setUser(data); // Might be for old userId!
    });
  }, [userId]);
  
  // FIX: Add cleanup
  useEffect(() => {
    let cancelled = false;
    
    fetchUser(userId).then(data => {
      if (!cancelled) {
        setUser(data);
      }
    });
    
    return () => {
      cancelled = true;
    };
  }, [userId]);
}
```

## Common Misconceptions

1. **"useEffect runs after render"** - Partially true. It runs after commit and after browser paint, not immediately after the component function returns.

2. **"Cleanup runs when the component unmounts"** - Partially true. It also runs before the effect executes again (when dependencies change).

3. **"Empty deps array means it runs once"** - True for the effect, but the cleanup still runs on unmount.

4. **"I don't need to include X in dependencies"** - Usually false. ESLint's exhaustive-deps rule is almost always right.

5. **"useEffect is for componentDidMount"** - Wrong mental model. useEffect is for synchronizing with external systems, not mimicking lifecycle methods.

## Performance Implications

1. **Effect Timing** - useEffect doesn't block painting, making it ideal for non-visual side effects. Use useLayoutEffect only when you need to read layout and synchronously re-render.

2. **Cleanup Cost** - Cleanup functions run synchronously during commit. Expensive cleanup can block the UI.

3. **Dependency Granularity** - Too many dependencies cause unnecessary effect re-runs. Too few cause stale closures.

4. **Effect Count** - Many effects in one component are fine. They run in order and are easier to reason about than one giant effect.

## Interview Questions

1. **Q: When exactly does useEffect run?**
   A: After the render phase completes, React commits changes to the DOM, the browser paints, then useEffect callbacks run asynchronously. On updates, cleanup runs before the new effect, after the browser has painted.

2. **Q: What's the difference between useEffect and useLayoutEffect?**
   A: useLayoutEffect runs synchronously after DOM mutations but before the browser paints, making it suitable for reading layout and synchronously re-rendering. useEffect runs asynchronously after paint, making it better for most side effects.

3. **Q: Why is the cleanup function important?**
   A: Cleanup prevents memory leaks and bugs. It unsubscribes from subscriptions, cancels pending requests, clears timers, and removes event listeners. Without cleanup, these operations would continue even after the component unmounts or dependencies change.

4. **Q: How does React compare dependencies?**
   A: React uses Object.is() to compare each dependency with its previous value. This is a shallow comparison—primitives are compared by value, objects and arrays by reference. All dependencies must be equal for the effect to be skipped.

5. **Q: What is a stale closure in useEffect?**
   A: A stale closure occurs when an effect with empty dependencies captures values from the initial render. Those captured values never update, even when the actual state/props change. Fix it with functional updates, refs, or proper dependencies.

6. **Q: Why do you need to include functions in the dependency array?**
   A: If a function is defined in the component body, it's recreated each render. If the effect depends on that function, it needs to be in the deps array or memoized with useCallback. Otherwise, you might have stale closures or miss updates.

7. **Q: Can you have multiple useEffect calls in one component?**
   A: Yes, and it's encouraged! Separate effects by concern rather than grouping unrelated logic. React runs them in order, and separate effects are easier to understand and maintain.

8. **Q: What causes an infinite loop with useEffect?**
   A: Updating state that's in the dependency array without proper conditions creates an infinite loop: effect runs → updates state → triggers re-render → effect runs again. Ensure state updates are conditional or use refs for values that shouldn't trigger effects.

## Key Takeaways

1. useEffect runs asynchronously after browser paint, not immediately after render
2. Cleanup functions run before the next effect and on unmount
3. Dependencies are compared with Object.is() (shallow, reference equality)
4. Empty dependency array runs effect once (mount) and cleanup once (unmount)
5. Missing dependencies cause stale closures; include all reactive values
6. Always clean up subscriptions, timers, and async operations
7. Separate concerns into multiple effects rather than one giant effect
8. Understanding effect timing is crucial for avoiding bugs and memory leaks

## Resources

- [useEffect Complete Guide](https://overreacted.io/a-complete-guide-to-useeffect/)
- [React useEffect Documentation](https://react.dev/reference/react/useEffect)
- [Synchronizing with Effects](https://react.dev/learn/synchronizing-with-effects)
- [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect)
- [React Hooks Source Code](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.js)
