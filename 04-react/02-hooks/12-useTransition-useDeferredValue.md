# useTransition and useDeferredValue

## The Idea

**In plain English:** `useTransition` and `useDeferredValue` are two React tools that let you tell your app which updates need to happen right now and which ones can wait a moment — so the page always feels fast and responsive, even when it has a lot of work to do.

**Real-world analogy:** Imagine you are a restaurant cashier taking orders during a lunch rush. A customer steps up and you immediately acknowledge them and start writing their order on your notepad (urgent). Meanwhile, sending that order ticket back to the kitchen to be cooked takes time, so the kitchen works on it in the background without stopping you from greeting the next customer (non-urgent).

- The cashier acknowledging the customer = updating the input field immediately
- The kitchen cooking the meal = running the expensive filtering or heavy computation
- The order ticket going to the kitchen = `startTransition` / `useDeferredValue` marking work as lower priority

---

## Overview

`useTransition` and `useDeferredValue` are React 18+ hooks that enable **concurrent rendering** features. They allow you to mark certain updates as **non-urgent**, keeping the UI responsive during expensive operations.

**Key Concept**: Not all state updates are equally important. Some (like typing) must be instant. Others (like filtering a large list) can be delayed.

## The Problem They Solve

### Without Concurrent Features

```jsx
function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  
  const handleChange = (e) => {
    const value = e.target.value;
    setQuery(value);  // UI update
    
    // Expensive computation
    const filtered = hugeList.filter(item => 
      item.name.toLowerCase().includes(value.toLowerCase())
    );
    setResults(filtered);  // Blocks UI!
  };
  
  // Input feels laggy because filtering blocks rendering
  return (
    <>
      <input value={query} onChange={handleChange} />
      <ResultsList results={results} />
    </>
  );
}
```

**Problem**: Both state updates have equal priority. The expensive filtering operation blocks the input from updating smoothly.

## useTransition

### Basic Syntax

```jsx
const [isPending, startTransition] = useTransition();
```

- **isPending**: Boolean indicating if transition is in progress
- **startTransition**: Function to wrap non-urgent updates

### Basic Example

```jsx
function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();
  
  const handleChange = (e) => {
    const value = e.target.value;
    
    // Urgent: Update input immediately
    setQuery(value);
    
    // Non-urgent: Defer expensive filtering
    startTransition(() => {
      const filtered = hugeList.filter(item => 
        item.name.toLowerCase().includes(value.toLowerCase())
      );
      setResults(filtered);
    });
  };
  
  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending && <Spinner />}
      <ResultsList results={results} />
    </>
  );
}
```

**What happens**:
1. Input updates immediately (stays responsive)
2. Filtering happens in background
3. `isPending` is true while filtering
4. Results update when ready

### How It Works

```jsx
// Without transition: Both updates are urgent
setState1(x);
setState2(y);  // Blocks until complete
// Both must finish before render

// With transition: Different priorities
setState1(x);  // Urgent - renders immediately
startTransition(() => {
  setState2(y);  // Non-urgent - can be interrupted
});
// React can render urgent updates first
```

## useDeferredValue

### Basic Syntax

```jsx
const deferredValue = useDeferredValue(value);
```

- **value**: The value you want to defer
- **deferredValue**: A deferred version that "lags behind"

### Basic Example

```jsx
function SearchPage() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  
  // query updates immediately
  // deferredQuery updates with lower priority
  
  return (
    <>
      <input 
        value={query} 
        onChange={(e) => setQuery(e.target.value)} 
      />
      <ResultsList query={deferredQuery} />
    </>
  );
}

function ResultsList({ query }) {
  // This component re-renders with lower priority
  const results = useMemo(() => 
    hugeList.filter(item => 
      item.name.toLowerCase().includes(query.toLowerCase())
    ),
    [query]
  );
  
  return results.map(item => <Item key={item.id} {...item} />);
}
```

**What happens**:
1. User types: `query` updates immediately
2. Input shows new value right away
3. `deferredQuery` updates with delay
4. Results update when React has time

## useTransition vs useDeferredValue

### When to Use Each

```jsx
// useTransition: You control the state update
function Component() {
  const [isPending, startTransition] = useTransition();
  
  const handleClick = () => {
    startTransition(() => {
      // You explicitly wrap the state update
      setState(newValue);
    });
  };
}

// useDeferredValue: You receive the value as prop/state
function Component({ value }) {
  // You don't control when value changes
  const deferredValue = useDeferredValue(value);
  
  // Use deferredValue for expensive operations
}
```

### Comparison

| Feature | useTransition | useDeferredValue |
|---------|---------------|------------------|
| Control | You wrap updates | React defers value |
| isPending | Yes | No (but you can derive it) |
| Use case | Own state updates | Props/external values |
| Flexibility | More control | Simpler |

## Practical Examples

### Example 1: Tabs with Slow Content

```jsx
function TabContainer() {
  const [activeTab, setActiveTab] = useState('home');
  const [isPending, startTransition] = useTransition();
  
  const handleTabClick = (tab) => {
    startTransition(() => {
      setActiveTab(tab);
    });
  };
  
  return (
    <>
      <div className="tabs">
        <button onClick={() => handleTabClick('home')}>Home</button>
        <button onClick={() => handleTabClick('profile')}>Profile</button>
        <button onClick={() => handleTabClick('settings')}>Settings</button>
      </div>
      
      {isPending && <div className="tab-loading">Loading...</div>}
      
      <div style={{ opacity: isPending ? 0.7 : 1 }}>
        {activeTab === 'home' && <HomePage />}
        {activeTab === 'profile' && <ProfilePage />}
        {activeTab === 'settings' && <SettingsPage />}
      </div>
    </>
  );
}
```

**Why this works**: Tab button stays responsive. Expensive tab content renders when ready.

### Example 2: Autocomplete with Deferred Value

```jsx
function Autocomplete({ suggestions }) {
  const [input, setInput] = useState('');
  const deferredInput = useDeferredValue(input);
  
  // Filtered with current input (can be stale)
  const filtered = useMemo(() => {
    return suggestions.filter(s => 
      s.toLowerCase().includes(deferredInput.toLowerCase())
    );
  }, [deferredInput, suggestions]);
  
  const isStale = input !== deferredInput;
  
  return (
    <>
      <input 
        value={input}
        onChange={(e) => setInput(e.target.value)}
      />
      
      <div style={{ opacity: isStale ? 0.6 : 1 }}>
        {filtered.map(item => (
          <div key={item}>{item}</div>
        ))}
      </div>
    </>
  );
}
```

### Example 3: List Filtering with Both Hooks

```jsx
function FilterableList({ items }) {
  const [filter, setFilter] = useState('');
  const [sortBy, setSortBy] = useState('name');
  const [isPending, startTransition] = useTransition();
  
  // Defer filter but not sort (sort is quick)
  const deferredFilter = useDeferredValue(filter);
  
  const handleSortChange = (newSort) => {
    // Wrap in transition for smooth experience
    startTransition(() => {
      setSortBy(newSort);
    });
  };
  
  const processedItems = useMemo(() => {
    // Filter with deferred value
    let result = items.filter(item =>
      item.name.toLowerCase().includes(deferredFilter.toLowerCase())
    );
    
    // Sort
    result.sort((a, b) => a[sortBy].localeCompare(b[sortBy]));
    
    return result;
  }, [items, deferredFilter, sortBy]);
  
  const isFiltering = filter !== deferredFilter;
  
  return (
    <>
      <input
        value={filter}
        onChange={(e) => setFilter(e.target.value)}
        placeholder="Filter..."
      />
      
      <select value={sortBy} onChange={(e) => handleSortChange(e.target.value)}>
        <option value="name">Name</option>
        <option value="date">Date</option>
        <option value="size">Size</option>
      </select>
      
      {(isPending || isFiltering) && <Spinner />}
      
      <div style={{ opacity: isPending || isFiltering ? 0.7 : 1 }}>
        {processedItems.map(item => (
          <Item key={item.id} {...item} />
        ))}
      </div>
    </>
  );
}
```

### Example 4: Dashboard with Live Updates

```jsx
function Dashboard() {
  const [refreshKey, setRefreshKey] = useState(0);
  const [isPending, startTransition] = useTransition();
  
  const handleRefresh = () => {
    startTransition(() => {
      // Non-blocking refresh
      setRefreshKey(k => k + 1);
    });
  };
  
  return (
    <>
      <button onClick={handleRefresh} disabled={isPending}>
        {isPending ? 'Refreshing...' : 'Refresh'}
      </button>
      
      <div style={{ opacity: isPending ? 0.7 : 1 }}>
        <ExpensiveChart key={refreshKey} />
        <ExpensiveTable key={refreshKey} />
        <ExpensiveStats key={refreshKey} />
      </div>
    </>
  );
}
```

### Example 5: Incremental Search Results

```jsx
function SearchResults({ query }) {
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;
  
  // Fetch based on deferred query
  const { data, isLoading } = useQuery({
    queryKey: ['search', deferredQuery],
    queryFn: () => searchAPI(deferredQuery),
    enabled: deferredQuery.length > 0
  });
  
  return (
    <div>
      {isStale && <div className="badge">Updating...</div>}
      
      <div style={{ 
        opacity: isStale ? 0.6 : 1,
        transition: 'opacity 200ms'
      }}>
        {isLoading ? (
          <Spinner />
        ) : (
          <ResultsList results={data} />
        )}
      </div>
    </div>
  );
}
```

## Advanced Patterns

### Pattern 1: Nested Transitions

```jsx
function ComplexApp() {
  const [tab, setTab] = useState('overview');
  const [isPending, startTransition] = useTransition();
  
  const handleTabChange = (newTab) => {
    startTransition(() => {
      setTab(newTab);
    });
  };
  
  return (
    <>
      <Tabs activeTab={tab} onTabChange={handleTabChange} />
      
      {isPending && <GlobalSpinner />}
      
      {/* Each tab can have its own transitions */}
      <TabContent tab={tab} />
    </>
  );
}

function TabContent({ tab }) {
  const [subView, setSubView] = useState('main');
  const [isPending, startTransition] = useTransition();
  
  const handleSubViewChange = (view) => {
    startTransition(() => {
      setSubView(view);
    });
  };
  
  return (
    <>
      {isPending && <LocalSpinner />}
      {/* Content */}
    </>
  );
}
```

### Pattern 2: Optimistic UI with Transitions

```jsx
function TodoList() {
  const [todos, setTodos] = useState([]);
  const [isPending, startTransition] = useTransition();
  
  const addTodo = async (text) => {
    // Optimistically add todo
    const tempId = Date.now();
    setTodos(prev => [...prev, { id: tempId, text, pending: true }]);
    
    try {
      const newTodo = await api.createTodo(text);
      
      // Replace temp with real todo (non-urgent)
      startTransition(() => {
        setTodos(prev => prev.map(todo =>
          todo.id === tempId ? { ...newTodo, pending: false } : todo
        ));
      });
    } catch (error) {
      // Remove temp todo on error
      setTodos(prev => prev.filter(todo => todo.id !== tempId));
    }
  };
  
  return (
    <>
      <AddTodoForm onAdd={addTodo} />
      <div style={{ opacity: isPending ? 0.8 : 1 }}>
        {todos.map(todo => (
          <TodoItem key={todo.id} {...todo} />
        ))}
      </div>
    </>
  );
}
```

### Pattern 3: Deferred Expensive Renders

```jsx
function DataGrid({ data }) {
  const [sortConfig, setSortConfig] = useState({ key: 'name', dir: 'asc' });
  const deferredSortConfig = useDeferredValue(sortConfig);
  
  // Sort with deferred config
  const sortedData = useMemo(() => {
    const sorted = [...data].sort((a, b) => {
      const aVal = a[deferredSortConfig.key];
      const bVal = b[deferredSortConfig.key];
      const modifier = deferredSortConfig.dir === 'asc' ? 1 : -1;
      return aVal > bVal ? modifier : -modifier;
    });
    return sorted;
  }, [data, deferredSortConfig]);
  
  const isSorting = sortConfig !== deferredSortConfig;
  
  return (
    <>
      <SortControls value={sortConfig} onChange={setSortConfig} />
      {isSorting && <SortingIndicator />}
      <VirtualizedTable data={sortedData} dimmed={isSorting} />
    </>
  );
}
```

## TypeScript

```tsx
import { useTransition, useDeferredValue } from 'react';

interface SearchProps {
  items: Array<{ id: string; name: string }>;
}

function Search({ items }: SearchProps) {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();
  const deferredQuery = useDeferredValue(query);
  
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    startTransition(() => {
      setQuery(e.target.value);
    });
  };
  
  const filtered = useMemo(
    () => items.filter(item => 
      item.name.toLowerCase().includes(deferredQuery.toLowerCase())
    ),
    [items, deferredQuery]
  );
  
  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending && <span>Loading...</span>}
      <ul>
        {filtered.map(item => (
          <li key={item.id}>{item.name}</li>
        ))}
      </ul>
    </>
  );
}
```

## Common Mistakes

### Mistake 1: Wrapping Everything in Transitions

```jsx
// Bad: Unnecessary transition
function Counter() {
  const [count, setCount] = useState(0);
  const [isPending, startTransition] = useTransition();
  
  const increment = () => {
    startTransition(() => {
      setCount(c => c + 1);  // This is already fast!
    });
  };
  
  return <button onClick={increment}>{count}</button>;
}

// Good: Only wrap expensive updates
function Counter() {
  const [count, setCount] = useState(0);
  
  const increment = () => {
    setCount(c => c + 1);  // Fast update, no transition needed
  };
  
  return <button onClick={increment}>{count}</button>;
}
```

### Mistake 2: Not Using useMemo with useDeferredValue

```jsx
// Bad: Computation runs every render
function Search({ items }) {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  
  // This runs on EVERY render!
  const filtered = items.filter(item => 
    item.name.includes(deferredQuery)
  );
  
  return <ResultsList results={filtered} />;
}

// Good: Memoize the computation
function Search({ items }) {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  
  const filtered = useMemo(
    () => items.filter(item => item.name.includes(deferredQuery)),
    [items, deferredQuery]
  );
  
  return <ResultsList results={filtered} />;
}
```

### Mistake 3: Confusing with debounce/throttle

```jsx
// These are NOT the same!

// Debounce: Delays execution until user stops typing
const debouncedSearch = debounce((query) => {
  setResults(search(query));
}, 300);

// useDeferredValue: Executes immediately but with lower priority
const deferredQuery = useDeferredValue(query);
// Computation happens right away, just doesn't block UI

// Use case:
// - Debounce: Reduce API calls, save computation
// - useDeferredValue: Keep UI responsive during heavy computation
```

### Mistake 4: Expecting Immediate Visual Feedback

```jsx
// Misunderstanding: Thinking isPending is instant
function BadExample() {
  const [isPending, startTransition] = useTransition();
  
  const handleClick = () => {
    startTransition(() => {
      // If this is fast, isPending might stay false!
      setState(x);
    });
  };
  
  // isPending might never show if update is quick
  return isPending ? <Spinner /> : <Content />;
}

// Better: Always show some feedback
function GoodExample() {
  const [isPending, startTransition] = useTransition();
  const [isClicking, setIsClicking] = useState(false);
  
  const handleClick = () => {
    setIsClicking(true);
    startTransition(() => {
      setState(x);
      setIsClicking(false);
    });
  };
  
  return (isPending || isClicking) ? <Spinner /> : <Content />;
}
```

## Performance Considerations

### When Transitions Help

```jsx
// ✓ Good use case: Heavy computation
const filtered = hugeList.filter(item => expensiveCheck(item));

// ✓ Good use case: Large list rendering
const items = data.map(item => <ExpensiveComponent key={item.id} {...item} />);

// ✓ Good use case: Route transitions
startTransition(() => {
  navigate('/dashboard');  // Loads heavy components
});
```

### When Transitions Don't Help

```jsx
// ✗ Bad: Simple state updates
startTransition(() => {
  setCount(c => c + 1);
});

// ✗ Bad: Already async operations
startTransition(() => {
  // API call is already async!
  fetch('/api/data').then(data => setData(data));
});

// ✗ Bad: Controlled inputs
// Never defer controlled input values
const deferredValue = useDeferredValue(inputValue);  // Don't do this
return <input value={deferredValue} />;  // Input will lag!
```

## Integration with Suspense

```jsx
function SearchPage() {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();
  const deferredQuery = useDeferredValue(query);
  
  return (
    <>
      <input
        value={query}
        onChange={(e) => {
          startTransition(() => {
            setQuery(e.target.value);
          });
        }}
      />
      
      {isPending && <Spinner />}
      
      <Suspense fallback={<ResultsSkeleton />}>
        <Results query={deferredQuery} />
      </Suspense>
    </>
  );
}

function Results({ query }) {
  // This can suspend without blocking input
  const data = use(fetchResults(query));
  return <ResultsList results={data} />;
}
```

## Browser Support

Both hooks require:
- React 18+
- Concurrent mode (enabled by default with createRoot)

```jsx
// Enable concurrent features
import { createRoot } from 'react-dom/client';

const root = createRoot(document.getElementById('root'));
root.render(<App />);
```

## Resources

- [React Docs: useTransition](https://react.dev/reference/react/useTransition)
- [React Docs: useDeferredValue](https://react.dev/reference/react/useDeferredValue)
- [Concurrent Rendering](https://react.dev/blog/2022/03/29/react-v18#what-is-concurrent-react)
- [New Suspense SSR Architecture](https://github.com/reactwg/react-18/discussions/37)
