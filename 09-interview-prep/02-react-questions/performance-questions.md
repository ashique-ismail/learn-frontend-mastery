# React Performance Interview Questions

## Overview

Performance questions test your ability to identify bottlenecks, optimize renders, and build fast React applications. Understanding memoization, code splitting, and profiling is essential for senior positions.

---

## Core Performance Questions

### Q1: Explain React.memo, useMemo, and useCallback. When would you use each?

**Answer:**

These are React's three main memoization tools. Each serves a different purpose and is often misunderstood.

**React.memo (Component Memoization):**

Prevents component re-renders when props haven't changed.

```javascript
// Without memo: Re-renders every time parent renders
function ExpensiveComponent({ data, onUpdate }) {
  console.log('Rendering ExpensiveComponent');
  // Expensive calculations or rendering
  return <div>{processData(data)}</div>;
}

// With memo: Only re-renders when props change
const ExpensiveComponent = React.memo(function ExpensiveComponent({ data, onUpdate }) {
  console.log('Rendering ExpensiveComponent');
  return <div>{processData(data)}</div>;
});

// Usage:
function Parent() {
  const [count, setCount] = useState(0);
  const [data, setData] = useState({ value: 100 });
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      <ExpensiveComponent data={data} />
    </div>
  );
}

// Without memo: ExpensiveComponent renders when count changes
// With memo: ExpensiveComponent only renders when data changes
```

**Custom Comparison Function:**

```javascript
const ExpensiveComponent = React.memo(
  function ExpensiveComponent({ user, config }) {
    return <div>{user.name}</div>;
  },
  (prevProps, nextProps) => {
    // Return true if props are equal (SKIP render)
    // Return false if props changed (DO render)
    return (
      prevProps.user.id === nextProps.user.id &&
      prevProps.config.theme === nextProps.config.theme
    );
  }
);
```

**useMemo (Value Memoization):**

Caches computed values to avoid expensive recalculations.

```javascript
function DataTable({ items, filterText }) {
  // Without useMemo: Filters on EVERY render (expensive!)
  const filteredItems = items.filter(item =>
    item.name.toLowerCase().includes(filterText.toLowerCase())
  );
  
  // With useMemo: Only filters when dependencies change
  const filteredItems = useMemo(() => {
    console.log('Filtering items...');
    return items.filter(item =>
      item.name.toLowerCase().includes(filterText.toLowerCase())
    );
  }, [items, filterText]);
  
  return (
    <ul>
      {filteredItems.map(item => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  );
}

// Real-world example: Expensive calculation
function Chart({ data }) {
  const chartConfig = useMemo(() => {
    console.log('Calculating chart config...');
    return {
      series: processTimeSeries(data),
      xAxis: calculateXAxis(data),
      yAxis: calculateYAxis(data),
      annotations: generateAnnotations(data)
    };
  }, [data]); // Only recalculate when data changes
  
  return <ChartLibrary config={chartConfig} />;
}
```

**useCallback (Function Memoization):**

Caches function references to prevent child re-renders.

```javascript
// Without useCallback: New function every render
function Parent() {
  const [count, setCount] = useState(0);
  
  // New function created on every render
  const handleClick = () => {
    console.log('Clicked');
  };
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      <ExpensiveChild onClick={handleClick} />
    </div>
  );
}

const ExpensiveChild = React.memo(function ExpensiveChild({ onClick }) {
  console.log('Child render');
  return <button onClick={onClick}>Click me</button>;
});

// Problem: Even with React.memo, ExpensiveChild renders on every count change
// Because handleClick is a new function each time!

// With useCallback: Stable function reference
function Parent() {
  const [count, setCount] = useState(0);
  
  // Same function reference unless dependencies change
  const handleClick = useCallback(() => {
    console.log('Clicked');
  }, []); // No dependencies = never changes
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      <ExpensiveChild onClick={handleClick} />
    </div>
  );
}

// Now ExpensiveChild only renders when onClick actually changes!
```

**With Dependencies:**

```javascript
function SearchComponent({ apiEndpoint }) {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  
  // Function recreated when query or apiEndpoint changes
  const handleSearch = useCallback(async () => {
    const data = await fetch(`${apiEndpoint}?q=${query}`);
    setResults(await data.json());
  }, [query, apiEndpoint]);
  
  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <SearchButton onSearch={handleSearch} />
      <ResultsList results={results} />
    </div>
  );
}
```

**Decision Matrix:**

| Tool | Use Case | Example |
|------|----------|---------|
| React.memo | Prevent component re-render | Expensive component in list |
| useMemo | Cache expensive calculation | Filtering/sorting large arrays |
| useCallback | Stable function reference | Callbacks passed to memoized children |

**Common Pattern (All Three Together):**

```javascript
function TodoApp() {
  const [todos, setTodos] = useState([]);
  const [filter, setFilter] = useState('all');
  
  // useMemo: Cache filtered list
  const filteredTodos = useMemo(() => {
    switch (filter) {
      case 'active':
        return todos.filter(t => !t.completed);
      case 'completed':
        return todos.filter(t => t.completed);
      default:
        return todos;
    }
  }, [todos, filter]);
  
  // useCallback: Stable function for child
  const handleToggle = useCallback((id) => {
    setTodos(todos =>
      todos.map(t =>
        t.id === id ? { ...t, completed: !t.completed } : t
      )
    );
  }, []); // No dependencies needed with functional setState
  
  const handleDelete = useCallback((id) => {
    setTodos(todos => todos.filter(t => t.id !== id));
  }, []);
  
  return (
    <div>
      <FilterButtons filter={filter} setFilter={setFilter} />
      <TodoList
        todos={filteredTodos}
        onToggle={handleToggle}
        onDelete={handleDelete}
      />
    </div>
  );
}

// React.memo: Prevent re-render when todos haven't changed
const TodoList = React.memo(function TodoList({ todos, onToggle, onDelete }) {
  return (
    <ul>
      {todos.map(todo => (
        <TodoItem
          key={todo.id}
          todo={todo}
          onToggle={onToggle}
          onDelete={onDelete}
        />
      ))}
    </ul>
  );
});
```

**What Interviewers Look For:**
- Clear understanding of each tool's purpose
- Knowledge of when optimization is needed
- Awareness of the relationship between them
- Understanding of dependency arrays

**Red Flags:**
- Wrapping everything in useMemo/useCallback
- Not understanding why they're needed together
- Missing dependencies
- "I always use them for every function"

---

### Q2: What causes unnecessary re-renders and how do you prevent them?

**Answer:**

Understanding re-render causes is crucial for performance optimization.

**Common Causes:**

**1. Parent Re-renders:**

```javascript
// PROBLEM: Child renders when parent's unrelated state changes
function Parent() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Count: {count}
      </button>
      <ExpensiveChild />
    </div>
  );
}

function ExpensiveChild() {
  console.log('ExpensiveChild render');
  // Expensive rendering logic
  return <div>Static content</div>;
}

// SOLUTION 1: React.memo
const ExpensiveChild = React.memo(function ExpensiveChild() {
  console.log('ExpensiveChild render');
  return <div>Static content</div>;
});

// SOLUTION 2: Extract state
function Parent() {
  return (
    <div>
      <Counter />
      <ExpensiveChild />
    </div>
  );
}

function Counter() {
  const [count, setCount] = useState(0);
  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  );
}
```

**2. Inline Object/Array Creation:**

```javascript
// PROBLEM: New object every render breaks memoization
function Parent() {
  return (
    <Child
      user={{ name: 'Alice', age: 30 }}  // New object every render!
      items={[1, 2, 3]}  // New array every render!
    />
  );
}

const Child = React.memo(function Child({ user, items }) {
  console.log('Child render');
  return <div>{user.name}</div>;
});

// Child renders every time because props are "different" (new object)

// SOLUTION 1: Move static data outside component
const DEFAULT_USER = { name: 'Alice', age: 30 };
const DEFAULT_ITEMS = [1, 2, 3];

function Parent() {
  return <Child user={DEFAULT_USER} items={DEFAULT_ITEMS} />;
}

// SOLUTION 2: useMemo
function Parent() {
  const user = useMemo(() => ({ name: 'Alice', age: 30 }), []);
  const items = useMemo(() => [1, 2, 3], []);
  
  return <Child user={user} items={items} />;
}

// SOLUTION 3: useState for derived data
function Parent({ userId }) {
  const [user] = useState(() => fetchUser(userId));
  
  return <Child user={user} />;
}
```

**3. Inline Function Definitions:**

```javascript
// PROBLEM: New function every render
function Parent() {
  return (
    <Child
      onClick={() => console.log('clicked')}  // New function!
      onHover={() => console.log('hovered')}   // New function!
    />
  );
}

const Child = React.memo(function Child({ onClick, onHover }) {
  console.log('Child render');
  return <button onClick={onClick}>Click me</button>;
});

// SOLUTION: useCallback
function Parent() {
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []);
  
  const handleHover = useCallback(() => {
    console.log('hovered');
  }, []);
  
  return <Child onClick={handleClick} onHover={handleHover} />;
}
```

**4. Context Changes:**

```javascript
// PROBLEM: Context value changes trigger all consumers
function App() {
  const [user, setUser] = useState({ name: 'Alice' });
  const [theme, setTheme] = useState('light');
  
  // New object every render!
  const contextValue = { user, theme };
  
  return (
    <AppContext.Provider value={contextValue}>
      <Header />
      <Content />
      <Footer />
    </AppContext.Provider>
  );
}

// All three components re-render when either user OR theme changes!

// SOLUTION 1: Memoize context value
function App() {
  const [user, setUser] = useState({ name: 'Alice' });
  const [theme, setTheme] = useState('light');
  
  const contextValue = useMemo(
    () => ({ user, theme }),
    [user, theme]
  );
  
  return (
    <AppContext.Provider value={contextValue}>
      <Header />
      <Content />
      <Footer />
    </AppContext.Provider>
  );
}

// SOLUTION 2: Split contexts
function App() {
  const [user, setUser] = useState({ name: 'Alice' });
  const [theme, setTheme] = useState('light');
  
  return (
    <UserContext.Provider value={user}>
      <ThemeContext.Provider value={theme}>
        <Header />  {/* Only re-renders when theme changes */}
        <Content /> {/* Only re-renders when user changes */}
        <Footer />  {/* Only re-renders when theme changes */}
      </ThemeContext.Provider>
    </UserContext.Provider>
  );
}
```

**5. State Updates with Same Value:**

```javascript
// PROBLEM: setState with same value still triggers render
function Component() {
  const [count, setCount] = useState(0);
  
  // Clicking still triggers render even if count is already 0
  return (
    <button onClick={() => setCount(0)}>
      Reset
    </button>
  );
}

// React bails out, but component still renders once

// SOLUTION: Check before updating
function Component() {
  const [count, setCount] = useState(0);
  
  const handleReset = () => {
    if (count !== 0) {
      setCount(0);
    }
  };
  
  return <button onClick={handleReset}>Reset</button>;
}
```

**6. Incorrect Dependencies:**

```javascript
// PROBLEM: Missing dependencies cause stale closures
function Timer() {
  const [count, setCount] = useState(0);
  const [multiplier, setMultiplier] = useState(1);
  
  useEffect(() => {
    const timer = setInterval(() => {
      setCount(count + 1); // Uses stale count!
    }, 1000);
    
    return () => clearInterval(timer);
  }, []); // Missing count dependency!
  
  // Count gets stuck at 1
  
  return <div>{count * multiplier}</div>;
}

// SOLUTION 1: Functional setState
function Timer() {
  const [count, setCount] = useState(0);
  const [multiplier, setMultiplier] = useState(1);
  
  useEffect(() => {
    const timer = setInterval(() => {
      setCount(c => c + 1); // Always uses latest!
    }, 1000);
    
    return () => clearInterval(timer);
  }, []); // Safe without count
  
  return <div>{count * multiplier}</div>;
}

// SOLUTION 2: Include all dependencies
function Timer() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const timer = setInterval(() => {
      setCount(count + 1);
    }, 1000);
    
    return () => clearInterval(timer);
  }, [count]); // Includes count (restarts interval)
  
  return <div>{count}</div>;
}
```

**Diagnostic Tools:**

```javascript
// React DevTools Profiler
// Highlights components that rendered
// Shows why each component rendered

// Custom hook to detect unnecessary renders
function useWhyDidYouUpdate(name, props) {
  const previousProps = useRef();
  
  useEffect(() => {
    if (previousProps.current) {
      const allKeys = Object.keys({ ...previousProps.current, ...props });
      const changedProps = {};
      
      allKeys.forEach(key => {
        if (previousProps.current[key] !== props[key]) {
          changedProps[key] = {
            from: previousProps.current[key],
            to: props[key]
          };
        }
      });
      
      if (Object.keys(changedProps).length > 0) {
        console.log('[why-did-you-update]', name, changedProps);
      }
    }
    
    previousProps.current = props;
  });
}

// Usage:
function ExpensiveComponent(props) {
  useWhyDidYouUpdate('ExpensiveComponent', props);
  return <div>{/* ... */}</div>;
}
```

**What Interviewers Look For:**
- Identifying re-render causes
- Understanding referential equality
- Knowledge of optimization techniques
- Awareness of common pitfalls

---

### Q3: Explain code splitting and lazy loading in React

**Answer:**

Code splitting breaks your app into smaller chunks that load on demand, improving initial load time.

**React.lazy and Suspense:**

```javascript
// Before: All code loads upfront
import HeavyComponent from './HeavyComponent';
import AnotherHeavyComponent from './AnotherHeavyComponent';

function App() {
  return (
    <div>
      <HeavyComponent />
      <AnotherHeavyComponent />
    </div>
  );
}

// After: Components load on demand
import { lazy, Suspense } from 'react';

const HeavyComponent = lazy(() => import('./HeavyComponent'));
const AnotherHeavyComponent = lazy(() => import('./AnotherHeavyComponent'));

function App() {
  return (
    <div>
      <Suspense fallback={<div>Loading...</div>}>
        <HeavyComponent />
        <AnotherHeavyComponent />
      </Suspense>
    </div>
  );
}
```

**Route-based Code Splitting:**

```javascript
import { lazy, Suspense } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

// Lazy load route components
const Home = lazy(() => import('./pages/Home'));
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));
const Profile = lazy(() => import('./pages/Profile'));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<LoadingSpinner />}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/dashboard" element={<Dashboard />} />
          <Route path="/settings" element={<Settings />} />
          <Route path="/profile" element={<Profile />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}

// Initial bundle: ~50KB
// Each route: ~20-30KB loaded on demand
// Total saved on initial load: ~60-90KB
```

**Component-level Splitting:**

```javascript
// Heavy modal that's not always shown
const HeavyModal = lazy(() => import('./HeavyModal'));

function Page() {
  const [showModal, setShowModal] = useState(false);
  
  return (
    <div>
      <button onClick={() => setShowModal(true)}>
        Open Modal
      </button>
      
      {showModal && (
        <Suspense fallback={<ModalSkeleton />}>
          <HeavyModal onClose={() => setShowModal(false)} />
        </Suspense>
      )}
    </div>
  );
}

// Modal code only loads when user clicks button!
```

**Named Exports with Lazy Loading:**

```javascript
// If component uses named export:
// components/Charts.js
export function LineChart() { /* ... */ }
export function BarChart() { /* ... */ }

// Lazy load it:
const Charts = lazy(() => import('./components/Charts'));

// Use named export:
function Dashboard() {
  return (
    <Suspense fallback={<div>Loading chart...</div>}>
      <Charts.LineChart />
    </Suspense>
  );
}

// OR: Import specific export
const LineChart = lazy(() =>
  import('./components/Charts').then(module => ({
    default: module.LineChart
  }))
);
```

**Error Boundaries with Lazy Loading:**

```javascript
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }
  
  static getDerivedStateFromError(error) {
    return { hasError: true };
  }
  
  componentDidCatch(error, errorInfo) {
    console.error('Lazy loading error:', error, errorInfo);
  }
  
  render() {
    if (this.state.hasError) {
      return <div>Failed to load component. <button onClick={() => window.location.reload()}>Retry</button></div>;
    }
    
    return this.props.children;
  }
}

// Usage:
function App() {
  return (
    <ErrorBoundary>
      <Suspense fallback={<Loading />}>
        <LazyComponent />
      </Suspense>
    </ErrorBoundary>
  );
}
```

**Preloading Components:**

```javascript
const HeavyComponent = lazy(() => import('./HeavyComponent'));

function App() {
  const [show, setShow] = useState(false);
  
  // Preload on hover (before click)
  const handleMouseEnter = () => {
    // Triggers the import
    import('./HeavyComponent');
  };
  
  return (
    <div>
      <button
        onMouseEnter={handleMouseEnter}
        onClick={() => setShow(true)}
      >
        Show Component
      </button>
      
      {show && (
        <Suspense fallback={<div>Loading...</div>}>
          <HeavyComponent />
        </Suspense>
      )}
    </div>
  );
}

// Component loads on hover, ready when clicked!
```

**Library Code Splitting:**

```javascript
// Heavy library used in one component
function ChartPage() {
  const [data, setData] = useState([]);
  
  useEffect(() => {
    // Load library only when needed
    import('chart.js').then(Chart => {
      const chart = new Chart.Chart(/* ... */);
    });
  }, []);
  
  return <canvas id="chart" />;
}
```

**Webpack Magic Comments:**

```javascript
// Control chunk names and loading
const Dashboard = lazy(() =>
  import(
    /* webpackChunkName: "dashboard" */
    /* webpackPreload: true */
    './Dashboard'
  )
);

const AdminPanel = lazy(() =>
  import(
    /* webpackChunkName: "admin" */
    /* webpackPrefetch: true */
    './AdminPanel'
  )
);

// Generates: dashboard.chunk.js, admin.chunk.js
// Preload: Load immediately (high priority)
// Prefetch: Load during idle time (low priority)
```

**What Interviewers Look For:**
- Understanding of lazy() and Suspense
- Knowledge of when to split code
- Awareness of loading states
- Error handling strategies

---

## Key Takeaways

**Essential Concepts:**
1. React.memo prevents component re-renders
2. useMemo caches computed values
3. useCallback caches function references
4. Code splitting reduces initial bundle size
5. Lazy loading defers non-critical code

**Best Practices:**
- Profile before optimizing
- Use React DevTools Profiler
- Split code at route boundaries
- Handle loading and error states
- Memoize context values
- Avoid inline objects/functions in props

**Common Mistakes:**
- Over-optimizing (premature optimization)
- Not understanding why optimization is needed
- Missing dependencies in memoization hooks
- Forgetting error boundaries with lazy loading
- Creating new objects in render

**Interview Tips:**
- Explain when NOT to optimize
- Show profiling workflow
- Demonstrate trade-offs
- Provide real-world examples
- Discuss bundle size impact

**Red Flags to Avoid:**
- "I wrap everything in useMemo"
- "Performance is always the priority"
- Not profiling before optimizing
- Ignoring bundle size
- Breaking functionality for performance
