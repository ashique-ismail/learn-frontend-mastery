# React Lifecycle Interview Questions

## Overview

Component lifecycle questions test your understanding of how React components are created, updated, and destroyed. Modern React relies heavily on hooks like useEffect, but understanding the traditional lifecycle methods and how they map to hooks is crucial for interviews.

---

## Core Lifecycle Questions

### Q1: Explain the component lifecycle phases in React

**Answer:**

React components go through three main phases:

**1. Mounting Phase (Birth)**
- Component is created and inserted into the DOM
- Class methods: constructor → getDerivedStateFromProps → render → componentDidMount
- Hooks equivalent: useState/useReducer initialization → render → useEffect (empty deps)

**2. Updating Phase (Growth)**
- Component re-renders due to props or state changes
- Class methods: getDerivedStateFromProps → shouldComponentUpdate → render → getSnapshotBeforeUpdate → componentDidUpdate
- Hooks equivalent: render → useEffect (with dependencies)

**3. Unmounting Phase (Death)**
- Component is removed from the DOM
- Class method: componentWillUnmount
- Hooks equivalent: useEffect cleanup function

**Code Example:**

```javascript
// Class Component Lifecycle
class LifecycleDemo extends React.Component {
  constructor(props) {
    super(props);
    this.state = { count: 0 };
    console.log('1. Constructor');
  }

  static getDerivedStateFromProps(props, state) {
    console.log('2. getDerivedStateFromProps');
    return null;
  }

  componentDidMount() {
    console.log('4. componentDidMount');
    // API calls, subscriptions, timers
  }

  shouldComponentUpdate(nextProps, nextState) {
    console.log('5. shouldComponentUpdate');
    return true; // return false to prevent re-render
  }

  getSnapshotBeforeUpdate(prevProps, prevState) {
    console.log('7. getSnapshotBeforeUpdate');
    return null;
  }

  componentDidUpdate(prevProps, prevState, snapshot) {
    console.log('8. componentDidUpdate');
    // React to prop/state changes
  }

  componentWillUnmount() {
    console.log('9. componentWillUnmount');
    // Cleanup: remove listeners, cancel timers
  }

  render() {
    console.log('3. Render');
    return <div>{this.state.count}</div>;
  }
}

// Hooks Equivalent
function LifecycleDemoHooks() {
  const [count, setCount] = useState(0);
  
  // componentDidMount
  useEffect(() => {
    console.log('Component mounted');
    
    // componentWillUnmount
    return () => {
      console.log('Component will unmount');
    };
  }, []); // Empty deps = run once on mount
  
  // componentDidUpdate (for count changes)
  useEffect(() => {
    console.log('Count updated:', count);
  }, [count]); // Runs when count changes
  
  return <div>{count}</div>;
}
```

**What Interviewers Look For:**
- Understanding of all three phases
- Knowledge of when each method runs
- Ability to map class lifecycle to hooks
- Understanding of why certain methods were deprecated

**Follow-up:** Why were componentWillMount, componentWillReceiveProps, and componentWillUpdate deprecated?

---

### Q2: How does useEffect relate to lifecycle methods?

**Answer:**

useEffect is a unified API that replaces multiple lifecycle methods. Understanding the mapping is essential:

**1. ComponentDidMount Equivalent:**
```javascript
useEffect(() => {
  console.log('Mounted - runs once');
  fetchData();
}, []); // Empty dependency array
```

**2. ComponentDidUpdate Equivalent:**
```javascript
useEffect(() => {
  console.log('Updated - runs on every render');
  updateAnalytics();
}); // No dependency array

useEffect(() => {
  console.log('userId changed');
  fetchUserData(userId);
}, [userId]); // Specific dependencies
```

**3. ComponentWillUnmount Equivalent:**
```javascript
useEffect(() => {
  const timer = setInterval(() => tick(), 1000);
  
  // Cleanup function = componentWillUnmount
  return () => {
    clearInterval(timer);
  };
}, []);
```

**4. Combined Lifecycle Pattern:**
```javascript
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    // Mount + Update when userId changes
    let cancelled = false;
    
    async function loadUser() {
      const data = await fetchUser(userId);
      if (!cancelled) {
        setUser(data);
      }
    }
    
    loadUser();
    
    // Cleanup on unmount or before next effect
    return () => {
      cancelled = true;
    };
  }, [userId]);
  
  return <div>{user?.name}</div>;
}
```

**Key Differences:**

| Aspect | Class Lifecycle | useEffect |
|--------|----------------|-----------|
| API | Multiple methods | Single hook |
| Timing | Separate mount/update | Combined logic |
| Cleanup | Separate method | Return function |
| Dependencies | Manual comparison | Automatic tracking |
| Mental Model | Lifecycle events | Synchronization |

**Common Pitfall:**
```javascript
// WRONG: Missing dependency
useEffect(() => {
  console.log(props.userId);
}, []); // Should include props.userId

// CORRECT:
useEffect(() => {
  console.log(props.userId);
}, [props.userId]);
```

**What Interviewers Look For:**
- Deep understanding of dependency arrays
- Knowledge of cleanup patterns
- Ability to avoid infinite loops
- Understanding of effect timing

---

### Q3: What's the difference between useEffect and useLayoutEffect?

**Answer:**

Both hooks have the same API but different timing:

**useEffect (Most Common):**
- Runs AFTER browser paint (asynchronously)
- Non-blocking
- Use for: data fetching, subscriptions, logging
- Better for performance

**useLayoutEffect (Special Cases):**
- Runs BEFORE browser paint (synchronously)
- Blocks visual updates
- Use for: DOM measurements, preventing flicker
- Can hurt performance if overused

**Visual Timeline:**

```
Render → DOM Updates → useLayoutEffect → Browser Paint → useEffect
```

**Practical Example:**

```javascript
// Example 1: Preventing Flash of Incorrect Content
function Tooltip({ targetRef }) {
  const [position, setPosition] = useState({ top: 0, left: 0 });
  
  // useLayoutEffect prevents visible jump
  useLayoutEffect(() => {
    const rect = targetRef.current.getBoundingClientRect();
    setPosition({
      top: rect.bottom + 10,
      left: rect.left
    });
  }, [targetRef]);
  
  return (
    <div style={{ position: 'absolute', ...position }}>
      Tooltip content
    </div>
  );
}

// Example 2: Measuring DOM Elements
function AutoResizeTextarea() {
  const textareaRef = useRef(null);
  
  useLayoutEffect(() => {
    // Measure and adjust before paint
    const textarea = textareaRef.current;
    textarea.style.height = 'auto';
    textarea.style.height = textarea.scrollHeight + 'px';
  });
  
  return <textarea ref={textareaRef} />;
}

// Example 3: When useEffect is fine
function DataFetcher() {
  const [data, setData] = useState(null);
  
  // useEffect is perfect here - no visual flicker concern
  useEffect(() => {
    fetchData().then(setData);
  }, []);
  
  return <div>{data}</div>;
}
```

**Decision Tree:**

```
Do you need to measure/mutate DOM before paint?
├─ Yes → useLayoutEffect
│   └─ Examples: tooltips, animations, scroll position
└─ No → useEffect
    └─ Examples: data fetching, subscriptions, analytics
```

**Performance Consideration:**

```javascript
// BAD: Unnecessary useLayoutEffect hurts performance
function LoggingComponent() {
  useLayoutEffect(() => {
    console.log('Rendered'); // Blocks paint unnecessarily
  });
  
  // GOOD: Use useEffect for non-visual side effects
  useEffect(() => {
    console.log('Rendered'); // Non-blocking
  });
}
```

**What Interviewers Look For:**
- Understanding the timing difference
- Knowing when to use each
- Awareness of performance implications
- Real-world use case examples

---

### Q4: How do you handle cleanup in useEffect?

**Answer:**

Cleanup is crucial to prevent memory leaks, stale closures, and unexpected behavior. Every effect that sets up a resource should clean it up.

**Common Cleanup Scenarios:**

**1. Event Listeners:**
```javascript
function WindowSize() {
  const [size, setSize] = useState({ width: 0, height: 0 });
  
  useEffect(() => {
    function handleResize() {
      setSize({
        width: window.innerWidth,
        height: window.innerHeight
      });
    }
    
    // Setup
    window.addEventListener('resize', handleResize);
    handleResize(); // Initial call
    
    // Cleanup
    return () => {
      window.removeEventListener('resize', handleResize);
    };
  }, []);
  
  return <div>{size.width} x {size.height}</div>;
}
```

**2. Subscriptions:**
```javascript
function RealtimeData({ channelId }) {
  const [messages, setMessages] = useState([]);
  
  useEffect(() => {
    // Setup subscription
    const subscription = subscribeToChannel(channelId, (message) => {
      setMessages(prev => [...prev, message]);
    });
    
    // Cleanup subscription
    return () => {
      subscription.unsubscribe();
    };
  }, [channelId]);
  
  return <MessageList messages={messages} />;
}
```

**3. Timers:**
```javascript
function Countdown({ seconds }) {
  const [remaining, setRemaining] = useState(seconds);
  
  useEffect(() => {
    // Don't start if already done
    if (remaining <= 0) return;
    
    // Setup timer
    const timer = setTimeout(() => {
      setRemaining(remaining - 1);
    }, 1000);
    
    // Cleanup timer
    return () => {
      clearTimeout(timer);
    };
  }, [remaining]);
  
  return <div>{remaining}s</div>;
}
```

**4. Async Operations (Fetch Cancellation):**
```javascript
function UserData({ userId }) {
  const [user, setUser] = useState(null);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    // Flag to track if component is mounted
    let cancelled = false;
    
    async function loadUser() {
      try {
        const data = await fetchUser(userId);
        
        // Only update if not cancelled
        if (!cancelled) {
          setUser(data);
          setError(null);
        }
      } catch (err) {
        if (!cancelled) {
          setError(err);
        }
      }
    }
    
    loadUser();
    
    // Cleanup: prevent setState on unmounted component
    return () => {
      cancelled = true;
    };
  }, [userId]);
  
  if (error) return <div>Error: {error.message}</div>;
  if (!user) return <div>Loading...</div>;
  return <div>{user.name}</div>;
}

// Modern approach with AbortController
function UserDataModern({ userId }) {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    const abortController = new AbortController();
    
    fetch(`/api/users/${userId}`, {
      signal: abortController.signal
    })
      .then(res => res.json())
      .then(setUser)
      .catch(err => {
        if (err.name !== 'AbortError') {
          console.error(err);
        }
      });
    
    return () => {
      abortController.abort();
    };
  }, [userId]);
  
  return user ? <div>{user.name}</div> : <div>Loading...</div>;
}
```

**5. Complex Cleanup Example:**
```javascript
function ChatRoom({ roomId, userId }) {
  const [messages, setMessages] = useState([]);
  
  useEffect(() => {
    // Multiple setup operations
    const socket = createWebSocket(`/chat/${roomId}`);
    const heartbeat = setInterval(() => socket.ping(), 30000);
    
    socket.on('message', (msg) => {
      setMessages(prev => [...prev, msg]);
    });
    
    socket.emit('join', { roomId, userId });
    
    // Cleanup all resources
    return () => {
      socket.emit('leave', { roomId, userId });
      socket.off('message');
      socket.close();
      clearInterval(heartbeat);
    };
  }, [roomId, userId]);
  
  return <MessageList messages={messages} />;
}
```

**Cleanup Execution Rules:**

1. Cleanup runs BEFORE the next effect (when deps change)
2. Cleanup runs when component unmounts
3. Cleanup does NOT run on initial mount
4. Each effect instance has its own cleanup function

**What Interviewers Look For:**
- Always cleaning up subscriptions and listeners
- Understanding of when cleanup runs
- Handling async operations properly
- Preventing memory leaks

**Red Flags:**
- Forgetting to cleanup event listeners
- Not handling component unmount during async operations
- Memory leaks from uncancelled timers

---

### Q5: What are the rules of useEffect dependencies?

**Answer:**

The dependency array controls when effects run. Getting this wrong causes bugs, infinite loops, or stale closures.

**Core Rules:**

**1. Include All Values Used Inside Effect:**
```javascript
// WRONG: Missing dependency
function SearchResults({ query }) {
  const [results, setResults] = useState([]);
  
  useEffect(() => {
    fetchResults(query).then(setResults); // Uses query
  }, []); // But doesn't list it! BUG!
  
  return <div>{results.length} results</div>;
}

// CORRECT:
function SearchResults({ query }) {
  const [results, setResults] = useState([]);
  
  useEffect(() => {
    fetchResults(query).then(setResults);
  }, [query]); // Includes all dependencies
  
  return <div>{results.length} results</div>;
}
```

**2. Primitive vs Reference Dependencies:**
```javascript
// Primitive values (string, number, boolean) - safe
useEffect(() => {
  console.log(userId);
}, [userId]); // Re-runs when value changes

// Objects/Arrays - may cause infinite loops
function UserProfile({ user }) {
  useEffect(() => {
    console.log(user);
  }, [user]); // New object each render = infinite loop!
}

// SOLUTIONS:

// Option 1: Extract primitive values
function UserProfile({ user }) {
  useEffect(() => {
    console.log(user);
  }, [user.id, user.name]); // Primitives
}

// Option 2: Memoize the object
function Parent() {
  const user = useMemo(() => ({
    id: 123,
    name: 'John'
  }), []); // Stable reference
  
  return <UserProfile user={user} />;
}

// Option 3: Use deep comparison (careful!)
function useDeepEffect(callback, deps) {
  const ref = useRef(deps);
  
  if (!deepEqual(ref.current, deps)) {
    ref.current = deps;
  }
  
  useEffect(callback, [ref.current]);
}
```

**3. Function Dependencies:**
```javascript
// PROBLEM: Function recreated every render
function SearchComponent() {
  const [query, setQuery] = useState('');
  
  const handleSearch = () => {
    api.search(query); // Uses query from closure
  };
  
  useEffect(() => {
    handleSearch();
  }, [handleSearch]); // Infinite loop! Function changes every render
}

// SOLUTION 1: Move function inside effect
function SearchComponent() {
  const [query, setQuery] = useState('');
  
  useEffect(() => {
    const handleSearch = () => {
      api.search(query);
    };
    
    handleSearch();
  }, [query]); // Only depends on query
}

// SOLUTION 2: useCallback
function SearchComponent() {
  const [query, setQuery] = useState('');
  
  const handleSearch = useCallback(() => {
    api.search(query);
  }, [query]); // Stable reference when query unchanged
  
  useEffect(() => {
    handleSearch();
  }, [handleSearch]);
}

// SOLUTION 3: Functional setState (if only setting state)
function Counter() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const timer = setInterval(() => {
      setCount(c => c + 1); // Doesn't need count in deps
    }, 1000);
    
    return () => clearInterval(timer);
  }, []); // Empty deps array is safe
}
```

**4. No Dependencies vs Empty Dependencies:**
```javascript
// No deps = runs after EVERY render
useEffect(() => {
  console.log('Every render');
}); // Dangerous - can cause performance issues

// Empty deps = runs ONCE on mount
useEffect(() => {
  console.log('Once on mount');
}, []); // Safe for setup operations

// With deps = runs when deps change
useEffect(() => {
  console.log('When userId changes');
}, [userId]); // Most common pattern
```

**5. ESLint Plugin:**
```javascript
// Install eslint-plugin-react-hooks
// It will warn about missing dependencies

// Example warning:
useEffect(() => {
  doSomething(prop1, prop2);
}, [prop1]); // ESLint: missing dependency 'prop2'
```

**Common Patterns:**

```javascript
// Pattern 1: Debounced effect
function SearchInput() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  
  useEffect(() => {
    const timer = setTimeout(() => {
      search(query).then(setResults);
    }, 500);
    
    return () => clearTimeout(timer);
  }, [query]); // Debounces on query changes
  
  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}

// Pattern 2: Syncing with external store
function useWindowSize() {
  const [size, setSize] = useState({ width: 0, height: 0 });
  
  useEffect(() => {
    function updateSize() {
      setSize({ width: window.innerWidth, height: window.innerHeight });
    }
    
    updateSize();
    window.addEventListener('resize', updateSize);
    return () => window.removeEventListener('resize', updateSize);
  }, []); // No dependencies - syncs with window
  
  return size;
}

// Pattern 3: Conditional effect
function Notification({ message, duration }) {
  useEffect(() => {
    if (!message) return; // Guard clause
    
    const timer = setTimeout(() => {
      clearNotification();
    }, duration);
    
    return () => clearTimeout(timer);
  }, [message, duration]);
}
```

**What Interviewers Look For:**
- Understanding dependency array purpose
- Ability to identify missing dependencies
- Knowledge of common pitfalls
- Solutions for object/function dependencies

**Red Flags:**
- Disabling ESLint rules
- Empty dependencies when values are used
- Infinite loops from object dependencies

---

## Advanced Lifecycle Questions

### Q6: How do you implement componentDidMount with dependencies?

**Answer:**

Sometimes you need "mount-like" behavior but want to re-run when specific values change. This is a common pattern for data fetching.

```javascript
// Pattern 1: Mount + dependency updates
function UserPosts({ userId }) {
  const [posts, setPosts] = useState([]);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    setLoading(true);
    
    fetchUserPosts(userId)
      .then(setPosts)
      .finally(() => setLoading(false));
  }, [userId]); // Runs on mount AND when userId changes
  
  if (loading) return <Spinner />;
  return <PostList posts={posts} />;
}

// Pattern 2: Only on mount, ignore prop changes
function AnalyticsTracker({ pageId }) {
  const mountedPageId = useRef(pageId);
  
  useEffect(() => {
    // Only track the initial page
    analytics.pageView(mountedPageId.current);
  }, []); // Truly only on mount
}

// Pattern 3: Mount + selective updates
function DataFetcher({ userId, includeDetails }) {
  useEffect(() => {
    fetchData(userId);
  }, [userId]); // Only refetch when userId changes, ignore includeDetails
  
  // Separate effect for details
  useEffect(() => {
    if (includeDetails) {
      fetchDetails(userId);
    }
  }, [userId, includeDetails]);
}
```

---

### Q7: How do you prevent effects from running on initial mount?

**Answer:**

Sometimes you want an effect to run on updates but NOT on the initial mount.

```javascript
// Solution 1: Using useRef
function UpdateOnlyEffect() {
  const [count, setCount] = useState(0);
  const isFirstRender = useRef(true);
  
  useEffect(() => {
    if (isFirstRender.current) {
      isFirstRender.current = false;
      return; // Skip first run
    }
    
    console.log('Count updated:', count);
    // This only runs on updates, not mount
  }, [count]);
  
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// Solution 2: Custom hook
function useUpdateEffect(effect, deps) {
  const isFirstRender = useRef(true);
  
  useEffect(() => {
    if (isFirstRender.current) {
      isFirstRender.current = false;
      return;
    }
    
    return effect();
  }, deps);
}

// Usage
function Component() {
  const [value, setValue] = useState('');
  
  useUpdateEffect(() => {
    console.log('Value updated:', value);
  }, [value]);
}

// Solution 3: Previous value comparison
function PreviousValueEffect() {
  const [count, setCount] = useState(0);
  const prevCount = useRef(count);
  
  useEffect(() => {
    if (prevCount.current !== count) {
      console.log(`Changed from ${prevCount.current} to ${count}`);
      prevCount.current = count;
    }
  }, [count]);
}
```

---

### Q8: Explain getDerivedStateFromProps and its hooks equivalent

**Answer:**

getDerivedStateFromProps is rarely needed in modern React. Most use cases have better alternatives.

```javascript
// Class Component (OLD WAY)
class EmailInput extends React.Component {
  state = { email: this.props.defaultEmail };
  
  static getDerivedStateFromProps(props, state) {
    // Anti-pattern: Overwriting state on every render
    if (props.defaultEmail !== state.prevDefaultEmail) {
      return {
        email: props.defaultEmail,
        prevDefaultEmail: props.defaultEmail
      };
    }
    return null;
  }
  
  render() {
    return (
      <input
        value={this.state.email}
        onChange={e => this.setState({ email: e.target.value })}
      />
    );
  }
}

// BETTER APPROACH: Fully controlled
function EmailInputControlled({ value, onChange }) {
  return <input value={value} onChange={onChange} />;
}

// BETTER APPROACH: Fully uncontrolled with key
function EmailInputUncontrolled({ defaultEmail, userId }) {
  return <input key={userId} defaultValue={defaultEmail} />;
}

// WHEN NEEDED: Controlled with default
function EmailInputHybrid({ defaultEmail }) {
  const [email, setEmail] = useState(defaultEmail);
  
  // Reset when default changes
  useEffect(() => {
    setEmail(defaultEmail);
  }, [defaultEmail]);
  
  return <input value={email} onChange={e => setEmail(e.target.value)} />;
}

// BEST APPROACH: Key prop pattern
function UserForm({ userId, defaultData }) {
  // Key forces remount when user changes
  return <FormFields key={userId} defaultData={defaultData} />;
}

function FormFields({ defaultData }) {
  const [data, setData] = useState(defaultData);
  // No need for getDerivedStateFromProps!
  
  return <form>{/* fields */}</form>;
}
```

**When getDerivedStateFromProps was used:**
1. Syncing state with props (usually an anti-pattern)
2. Resetting state when props change (use key instead)
3. Computing values from props (use useMemo instead)

**Modern alternatives:**
- Fully controlled component
- Key prop to reset state
- useMemo for derived values
- useEffect for sync operations

---

## Key Takeaways

**Essential Concepts:**
1. Three lifecycle phases: Mount, Update, Unmount
2. useEffect replaces multiple lifecycle methods
3. Dependency arrays control effect execution
4. Cleanup prevents memory leaks
5. useLayoutEffect for DOM measurements

**Best Practices:**
- Always include dependencies (use ESLint)
- Clean up subscriptions and listeners
- Handle async operations properly
- Use key prop instead of getDerivedStateFromProps
- Prefer useEffect over useLayoutEffect unless needed

**Common Mistakes:**
- Missing dependencies causing stale closures
- Forgetting cleanup causing memory leaks
- Using objects/arrays in dependencies causing infinite loops
- Disabling ESLint warnings
- Overusing useLayoutEffect

**Interview Tips:**
- Explain lifecycle phases clearly
- Show cleanup examples
- Demonstrate dependency array understanding
- Map class lifecycle to hooks
- Provide real-world use cases

**Red Flags to Avoid:**
- "I don't use dependencies"
- "I disable ESLint warnings"
- Not knowing cleanup patterns
- Confusing mount and update phases
- Not understanding useEffect timing
