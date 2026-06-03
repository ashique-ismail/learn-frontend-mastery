# Concurrent Rendering in React 18+

## Table of Contents
- [Introduction](#introduction)
- [What is Concurrent Rendering?](#what-is-concurrent-rendering)
- [Core Concepts](#core-concepts)
- [startTransition API](#starttransition-api)
- [useTransition Hook](#usetransition-hook)
- [useDeferredValue Hook](#usedeferredvalue-hook)
- [Interruptible Rendering](#interruptible-rendering)
- [Urgent vs Non-Urgent Updates](#urgent-vs-non-urgent-updates)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Concurrent rendering is React 18's most significant architectural change, enabling React to work on multiple versions of the UI simultaneously and interrupt less important work to handle urgent updates. This fundamentally changes how React prioritizes and schedules updates.

## What is Concurrent Rendering?

Concurrent rendering allows React to prepare multiple versions of the UI at the same time. React can start rendering an update, pause in the middle if something more important comes up, and resume or abandon the work later.

**Key Characteristics:**
- Non-blocking rendering
- Interruptible updates
- Prioritized updates
- Improved perceived performance
- Better responsiveness for user interactions

**Before React 18 (Synchronous Rendering):**
```typescript
// Synchronous: blocks the main thread
function SearchResults() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    setQuery(value); // Blocks while updating
    
    // Heavy computation blocks UI
    const filtered = heavyFilterOperation(data, value);
    setResults(filtered); // Also blocks
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      <ResultsList results={results} />
    </>
  );
}
```

**With React 18 (Concurrent Rendering):**
```typescript
import { useState, useTransition } from 'react';

function SearchResults() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    setQuery(value); // Urgent: updates immediately

    // Non-urgent: can be interrupted
    startTransition(() => {
      const filtered = heavyFilterOperation(data, value);
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

## Core Concepts

### 1. Concurrent Features are Opt-In

Concurrent rendering is enabled by default in React 18, but concurrent features are opt-in:

```typescript
import { createRoot } from 'react-dom/client';

// Enables concurrent features
const root = createRoot(document.getElementById('root')!);
root.render(<App />);

// Old API (legacy mode, no concurrent features)
// ReactDOM.render(<App />, document.getElementById('root'));
```

### 2. Time Slicing

React breaks rendering work into small chunks and spreads it over multiple frames:

```typescript
function TimeSlicingDemo() {
  const [items, setItems] = useState<number[]>([]);
  const [isPending, startTransition] = useTransition();

  const handleClick = () => {
    startTransition(() => {
      // Generate 10,000 items - will be time-sliced
      setItems(Array.from({ length: 10000 }, (_, i) => i));
    });
  };

  return (
    <>
      <button onClick={handleClick}>
        Generate Items {isPending && '(Loading...)'}
      </button>
      <ul>
        {items.map(item => (
          <li key={item}>Item {item}</li>
        ))}
      </ul>
    </>
  );
}
```

### 3. Priority Levels

React 18 has multiple priority levels for updates:

```typescript
// Highest priority: Discrete user interactions
// - Clicks, keypresses, focus events
// - Must be synchronous and immediate

// High priority: Continuous user interactions
// - Hover, scroll events
// - Should be responsive but can batch

// Default priority: Data fetching, timers
// - Regular state updates
// - Can be deferred if needed

// Low priority: Expensive computations
// - Heavy rendering, complex calculations
// - Can be interrupted and resumed

// Idle priority: Background work
// - Analytics, logging
// - Only when browser is idle
```

## startTransition API

Mark state updates as non-urgent, allowing React to interrupt them for more important work.

### Basic Usage

```typescript
import { startTransition } from 'react';

function SearchPage() {
  const [searchTerm, setSearchTerm] = useState('');
  const [results, setResults] = useState<Result[]>([]);

  const handleSearch = (value: string) => {
    // Urgent: Keep input responsive
    setSearchTerm(value);

    // Non-urgent: Can be interrupted
    startTransition(() => {
      const filtered = expensiveSearch(value);
      setResults(filtered);
    });
  };

  return (
    <>
      <input 
        value={searchTerm}
        onChange={(e) => handleSearch(e.target.value)}
        placeholder="Search..."
      />
      <SearchResults results={results} />
    </>
  );
}
```

### Complex Tab Switching

```typescript
interface TabProps {
  activeTab: string;
  onTabChange: (tab: string) => void;
}

function Tabs({ activeTab, onTabChange }: TabProps) {
  const [isPending, startTransition] = useTransition();

  const handleClick = (tab: string) => {
    startTransition(() => {
      onTabChange(tab);
    });
  };

  return (
    <div className="tabs">
      {['Profile', 'Posts', 'Contact'].map(tab => (
        <button
          key={tab}
          onClick={() => handleClick(tab)}
          className={activeTab === tab ? 'active' : ''}
          disabled={isPending}
        >
          {tab}
        </button>
      ))}
    </div>
  );
}

function App() {
  const [activeTab, setActiveTab] = useState('Profile');

  return (
    <>
      <Tabs activeTab={activeTab} onTabChange={setActiveTab} />
      <Suspense fallback={<Spinner />}>
        {activeTab === 'Profile' && <ProfileTab />}
        {activeTab === 'Posts' && <PostsTab />}
        {activeTab === 'Contact' && <ContactTab />}
      </Suspense>
    </>
  );
}
```

### With Data Fetching

```typescript
function UserSearch() {
  const [query, setQuery] = useState('');
  const [users, setUsers] = useState<User[]>([]);
  const [isSearching, setIsSearching] = useState(false);

  const searchUsers = async (searchQuery: string) => {
    if (!searchQuery.trim()) {
      setUsers([]);
      return;
    }

    setIsSearching(true);

    try {
      const response = await fetch(`/api/users?q=${searchQuery}`);
      const data = await response.json();
      
      startTransition(() => {
        setUsers(data);
        setIsSearching(false);
      });
    } catch (error) {
      console.error('Search failed:', error);
      setIsSearching(false);
    }
  };

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    setQuery(value);
    searchUsers(value);
  };

  return (
    <>
      <input 
        value={query}
        onChange={handleChange}
        placeholder="Search users..."
      />
      {isSearching && <LoadingIndicator />}
      <UserList users={users} />
    </>
  );
}
```

## useTransition Hook

Provides a pending state while the transition is happening.

### Basic Pattern

```typescript
import { useState, useTransition } from 'react';

function FilteredList() {
  const [filter, setFilter] = useState('');
  const [items, setItems] = useState<Item[]>(largeDataset);
  const [isPending, startTransition] = useTransition();

  const handleFilterChange = (value: string) => {
    setFilter(value);

    startTransition(() => {
      const filtered = largeDataset.filter(item =>
        item.name.toLowerCase().includes(value.toLowerCase())
      );
      setItems(filtered);
    });
  };

  return (
    <>
      <input
        value={filter}
        onChange={(e) => handleFilterChange(e.target.value)}
      />
      {isPending && <InlineSpinner />}
      <ul style={{ opacity: isPending ? 0.5 : 1 }}>
        {items.map(item => (
          <li key={item.id}>{item.name}</li>
        ))}
      </ul>
    </>
  );
}
```

### Custom Hook with Transition

```typescript
function useFilteredData<T>(
  data: T[],
  filterFn: (item: T, query: string) => boolean
) {
  const [query, setQuery] = useState('');
  const [filtered, setFiltered] = useState<T[]>(data);
  const [isPending, startTransition] = useTransition();

  const updateFilter = (newQuery: string) => {
    setQuery(newQuery);

    startTransition(() => {
      const result = data.filter(item => filterFn(item, newQuery));
      setFiltered(result);
    });
  };

  return { query, filtered, isPending, updateFilter };
}

// Usage
function ProductList() {
  const products = useProducts();
  const { query, filtered, isPending, updateFilter } = useFilteredData(
    products,
    (product, query) => 
      product.name.toLowerCase().includes(query.toLowerCase())
  );

  return (
    <>
      <SearchInput value={query} onChange={updateFilter} />
      {isPending && <LoadingBar />}
      <Products products={filtered} dimmed={isPending} />
    </>
  );
}
```

### Multiple Transitions

```typescript
function Dashboard() {
  const [chartData, setChartData] = useState<ChartData>([]);
  const [tableData, setTableData] = useState<TableData>([]);
  const [isChartPending, startChartTransition] = useTransition();
  const [isTablePending, startTableTransition] = useTransition();

  const updateDateRange = (start: Date, end: Date) => {
    // Both transitions can happen independently
    startChartTransition(() => {
      const data = calculateChartData(start, end);
      setChartData(data);
    });

    startTableTransition(() => {
      const data = calculateTableData(start, end);
      setTableData(data);
    });
  };

  return (
    <>
      <DateRangePicker onChange={updateDateRange} />
      <Chart data={chartData} loading={isChartPending} />
      <Table data={tableData} loading={isTablePending} />
    </>
  );
}
```

## useDeferredValue Hook

Defer updating a value until more urgent updates have finished.

### Basic Usage

```typescript
import { useState, useDeferredValue } from 'react';

function SearchResults() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);

  // query updates immediately (keeps input responsive)
  // deferredQuery updates later (non-urgent)

  return (
    <>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
      />
      <ExpensiveResults query={deferredQuery} />
    </>
  );
}

function ExpensiveResults({ query }: { query: string }) {
  const results = useMemo(() => {
    // Expensive computation only uses deferred value
    return heavySearch(query);
  }, [query]);

  return <ResultsList results={results} />;
}
```

### With Loading State

```typescript
function ProductSearch() {
  const [searchTerm, setSearchTerm] = useState('');
  const deferredSearchTerm = useDeferredValue(searchTerm);
  const isStale = searchTerm !== deferredSearchTerm;

  const products = useMemo(
    () => searchProducts(deferredSearchTerm),
    [deferredSearchTerm]
  );

  return (
    <>
      <input
        value={searchTerm}
        onChange={(e) => setSearchTerm(e.target.value)}
        placeholder="Search products..."
      />
      <div style={{ opacity: isStale ? 0.5 : 1 }}>
        {isStale && <LoadingSpinner />}
        <ProductGrid products={products} />
      </div>
    </>
  );
}
```

### Debouncing with useDeferredValue

```typescript
function AutocompleteSearch() {
  const [input, setInput] = useState('');
  const deferredInput = useDeferredValue(input);

  // Suggestions only recalculate when deferredInput changes
  const suggestions = useMemo(() => {
    if (!deferredInput.trim()) return [];
    return getSuggestions(deferredInput);
  }, [deferredInput]);

  return (
    <>
      <input
        value={input}
        onChange={(e) => setInput(e.target.value)}
        placeholder="Type to search..."
      />
      {suggestions.length > 0 && (
        <SuggestionsList suggestions={suggestions} />
      )}
    </>
  );
}
```

### useDeferredValue vs useTransition

```typescript
// useDeferredValue: Defer a VALUE
function WithDeferredValue() {
  const [text, setText] = useState('');
  const deferredText = useDeferredValue(text);

  return (
    <>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <SlowList text={deferredText} />
    </>
  );
}

// useTransition: Defer a STATE UPDATE
function WithTransition() {
  const [text, setText] = useState('');
  const [displayText, setDisplayText] = useState('');
  const [isPending, startTransition] = useTransition();

  const handleChange = (value: string) => {
    setText(value);
    startTransition(() => {
      setDisplayText(value);
    });
  };

  return (
    <>
      <input value={text} onChange={(e) => handleChange(e.target.value)} />
      {isPending && <Spinner />}
      <SlowList text={displayText} />
    </>
  );
}

// When to use which:
// - useDeferredValue: Don't control the state update (props, third-party)
// - useTransition: Control the state update + need isPending
```

## Interruptible Rendering

React can pause rendering work and switch to more important tasks.

### How Interruption Works

```typescript
function InterruptibleList() {
  const [items, setItems] = useState<number[]>([]);
  const [isPending, startTransition] = useTransition();

  const generateItems = () => {
    startTransition(() => {
      // Rendering 50,000 items is interruptible
      setItems(Array.from({ length: 50000 }, (_, i) => i));
    });
  };

  // If user types while rendering, React will:
  // 1. Pause the list rendering
  // 2. Handle the input (urgent)
  // 3. Resume list rendering

  return (
    <>
      <input placeholder="Type here (won't block)" />
      <button onClick={generateItems}>
        Generate {isPending && '(Rendering...)'}
      </button>
      <ul>
        {items.map(item => (
          <li key={item}>Item {item}</li>
        ))}
      </ul>
    </>
  );
}
```

### Demonstrating Interruption

```typescript
function InterruptionDemo() {
  const [urgentText, setUrgentText] = useState('');
  const [slowText, setSlowText] = useState('');
  const [isPending, startTransition] = useTransition();

  const handleUrgentChange = (value: string) => {
    // Always responsive, never interrupted
    setUrgentText(value);
  };

  const handleSlowChange = (value: string) => {
    startTransition(() => {
      // Can be interrupted by urgent updates
      setSlowText(value);
    });
  };

  return (
    <>
      <div>
        <label>Urgent Input (always responsive):</label>
        <input
          value={urgentText}
          onChange={(e) => handleUrgentChange(e.target.value)}
        />
      </div>

      <div>
        <label>Slow Input (interruptible):</label>
        <input onChange={(e) => handleSlowChange(e.target.value)} />
        {isPending && <span> Updating...</span>}
      </div>

      <SlowComponent text={slowText} />
    </>
  );
}

function SlowComponent({ text }: { text: string }) {
  // Intentionally slow to demonstrate interruption
  const items = useMemo(() => {
    const arr = [];
    for (let i = 0; i < 10000; i++) {
      arr.push(`${text}-${i}`);
    }
    return arr;
  }, [text]);

  return <div>Rendered {items.length} items</div>;
}
```

## Urgent vs Non-Urgent Updates

Understanding which updates should be urgent vs non-urgent is crucial.

### Urgent Updates

```typescript
// Urgent: Direct user input
function UrgentExamples() {
  const [text, setText] = useState('');
  const [checked, setChecked] = useState(false);
  const [selected, setSelected] = useState('');

  return (
    <>
      {/* Input value MUST update immediately */}
      <input value={text} onChange={(e) => setText(e.target.value)} />

      {/* Checkbox MUST check immediately */}
      <input
        type="checkbox"
        checked={checked}
        onChange={(e) => setChecked(e.target.checked)}
      />

      {/* Selection MUST update immediately */}
      <select value={selected} onChange={(e) => setSelected(e.target.value)}>
        <option value="a">A</option>
        <option value="b">B</option>
      </select>
    </>
  );
}
```

### Non-Urgent Updates

```typescript
// Non-urgent: Derived UI updates
function NonUrgentExamples() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<Result[]>([]);
  const [isPending, startTransition] = useTransition();

  const handleSearch = (value: string) => {
    // Urgent: input value
    setQuery(value);

    // Non-urgent: search results
    startTransition(() => {
      const filtered = performSearch(value);
      setResults(filtered);
    });
  };

  return (
    <>
      <input value={query} onChange={(e) => handleSearch(e.target.value)} />
      <SearchResults results={results} pending={isPending} />
    </>
  );
}
```

### Decision Matrix

```typescript
interface UpdatePriorityGuide {
  // URGENT (Don't wrap in transition):
  urgent: {
    directInput: string; // Input, textarea values
    formControls: boolean; // Checkbox, radio, select
    focusState: boolean; // Focus, blur events
    hoverState: boolean; // Hover effects
    clickFeedback: boolean; // Button pressed states
  };

  // NON-URGENT (Wrap in transition):
  nonUrgent: {
    searchResults: any[]; // Filtered/searched data
    chartUpdates: any; // Heavy visualizations
    largeListRenders: any[]; // Long lists
    tabContent: React.ReactNode; // Tab panel switches
    sortedData: any[]; // Sorting operations
    paginationChanges: number; // Page switches
  };
}

// Example combining both
function SmartSearchInterface() {
  const [inputValue, setInputValue] = useState(''); // Urgent
  const [displayValue, setDisplayValue] = useState(''); // Non-urgent
  const [results, setResults] = useState<Result[]>([]); // Non-urgent
  const [isPending, startTransition] = useTransition();

  const handleChange = (value: string) => {
    // Urgent: Keep input responsive
    setInputValue(value);

    // Non-urgent: Update display and results
    startTransition(() => {
      setDisplayValue(value);
      const filtered = expensiveFilter(value);
      setResults(filtered);
    });
  };

  return (
    <>
      {/* Always responsive */}
      <input
        value={inputValue}
        onChange={(e) => handleChange(e.target.value)}
      />

      {/* Shows what's actually displayed */}
      <p>Searching for: {displayValue}</p>

      {/* Non-urgent UI updates */}
      {isPending ? (
        <LoadingSpinner />
      ) : (
        <ResultsList results={results} />
      )}
    </>
  );
}
```

## Common Mistakes

### 1. Making Everything a Transition

```typescript
// WRONG: Input value in transition
function BadExample() {
  const [text, setText] = useState('');
  const [isPending, startTransition] = useTransition();

  const handleChange = (value: string) => {
    startTransition(() => {
      setText(value); // Input will feel laggy!
    });
  };

  return <input value={text} onChange={(e) => handleChange(e.target.value)} />;
}

// CORRECT: Only derived state in transition
function GoodExample() {
  const [text, setText] = useState('');
  const [results, setResults] = useState<Result[]>([]);
  const [isPending, startTransition] = useTransition();

  const handleChange = (value: string) => {
    setText(value); // Urgent: immediate

    startTransition(() => {
      const filtered = filterResults(value);
      setResults(filtered); // Non-urgent
    });
  };

  return (
    <>
      <input value={text} onChange={(e) => handleChange(e.target.value)} />
      <Results data={results} pending={isPending} />
    </>
  );
}
```

### 2. Not Using isPending for UX

```typescript
// WRONG: No loading indicator
function BadUX() {
  const [, startTransition] = useTransition();
  const [tab, setTab] = useState('home');

  return (
    <>
      <button onClick={() => startTransition(() => setTab('profile'))}>
        Profile
      </button>
      <TabContent tab={tab} />
    </>
  );
}

// CORRECT: Show pending state
function GoodUX() {
  const [isPending, startTransition] = useTransition();
  const [tab, setTab] = useState('home');

  return (
    <>
      <button
        onClick={() => startTransition(() => setTab('profile'))}
        disabled={isPending}
      >
        Profile {isPending && <Spinner />}
      </button>
      <TabContent tab={tab} dimmed={isPending} />
    </>
  );
}
```

### 3. Mixing useDeferredValue and useTransition Incorrectly

```typescript
// WRONG: Using both unnecessarily
function Redundant() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  const [isPending, startTransition] = useTransition();

  const handleChange = (value: string) => {
    setQuery(value);
    startTransition(() => {
      // No additional state to update here!
    });
  };

  return (
    <>
      <input value={query} onChange={(e) => handleChange(e.target.value)} />
      <Results query={deferredQuery} />
    </>
  );
}

// CORRECT: Use one or the other
function Efficient() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);

  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <Results query={deferredQuery} />
    </>
  );
}
```

## Best Practices

### 1. Prioritize User Input

```typescript
function BestPracticeSearch() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<SearchResult[]>([]);
  const [isPending, startTransition] = useTransition();

  const handleSearch = (value: string) => {
    // Always update input immediately
    setQuery(value);

    // Defer expensive operations
    startTransition(() => {
      const filtered = performExpensiveSearch(value);
      setResults(filtered);
    });
  };

  return (
    <>
      <input
        value={query}
        onChange={(e) => handleSearch(e.target.value)}
        placeholder="Search..."
      />
      <div style={{ opacity: isPending ? 0.6 : 1 }}>
        {isPending && <ProgressBar />}
        <SearchResults results={results} />
      </div>
    </>
  );
}
```

### 2. Provide Visual Feedback

```typescript
function VisualFeedbackExample() {
  const [isPending, startTransition] = useTransition();
  const [activeTab, setActiveTab] = useState('home');

  const switchTab = (tab: string) => {
    startTransition(() => {
      setActiveTab(tab);
    });
  };

  return (
    <>
      <nav className={isPending ? 'loading' : ''}>
        {['home', 'profile', 'settings'].map(tab => (
          <button
            key={tab}
            onClick={() => switchTab(tab)}
            className={activeTab === tab ? 'active' : ''}
            disabled={isPending}
          >
            {tab}
            {isPending && activeTab === tab && <Spinner />}
          </button>
        ))}
      </nav>
      
      <Suspense fallback={<PageLoader />}>
        <TabContent tab={activeTab} />
      </Suspense>
    </>
  );
}
```

### 3. Combine with Suspense

```typescript
function SuspenseWithTransition() {
  const [resource, setResource] = useState(initialResource);
  const [isPending, startTransition] = useTransition();

  const refresh = () => {
    startTransition(() => {
      setResource(fetchNewResource());
    });
  };

  return (
    <>
      <button onClick={refresh} disabled={isPending}>
        Refresh {isPending && '...'}
      </button>
      <Suspense fallback={<Skeleton />}>
        <DataDisplay resource={resource} />
      </Suspense>
    </>
  );
}
```

## Interview Questions

### Q1: What is concurrent rendering and how does it improve performance?

**Answer:** Concurrent rendering allows React to work on multiple versions of the UI simultaneously and interrupt less important rendering work to handle urgent updates. Instead of blocking the main thread during rendering, React can pause, prioritize urgent work (like user input), and resume later. This keeps the UI responsive even during expensive updates.

### Q2: Explain the difference between useTransition and useDeferredValue.

**Answer:**
- **useTransition**: Marks state updates as non-urgent. Use when you control the state update and want to defer it. Returns `isPending` for loading states.
- **useDeferredValue**: Defers updating a value. Use when you receive a value (props, context) but want to defer using it. The original value updates immediately, but the deferred version lags behind.

### Q3: When should you NOT use startTransition?

**Answer:** Don't use startTransition for:
1. Direct user input (input values, checkboxes)
2. Focus and blur events
3. Immediate visual feedback (button pressed states)
4. Hover effects
5. Any update where delayed feedback would confuse users

### Q4: How does concurrent rendering handle interruption?

**Answer:** React breaks rendering into small units of work. Between units, it checks if higher-priority work has arrived. If so, React pauses the current work, handles the urgent update, and then either resumes or abandons the previous work. This is possible because rendering is separated from committing to the DOM.

### Q5: What's the relationship between concurrent rendering and Suspense?

**Answer:** Concurrent rendering enables Suspense to work smoothly. When a component suspends, React can continue rendering sibling components and show fallbacks without blocking. startTransition can be used to prevent already-visible content from being replaced by fallbacks during updates.

## Key Takeaways

1. **Concurrent rendering is opt-in** - Enabled with createRoot, but features like startTransition must be explicitly used
2. **Prioritize user input** - Never wrap direct user interactions in transitions
3. **Use isPending for UX** - Always provide visual feedback during transitions
4. **Choose the right API** - useTransition for state updates you control, useDeferredValue for values you receive
5. **Combine with Suspense** - Concurrent features work best with Suspense for data fetching
6. **Non-blocking by default** - Transitions can be interrupted by more urgent work
7. **Visual feedback matters** - Show loading states, dim content, or disable controls during transitions
8. **Not a silver bullet** - Profile and measure; don't assume everything benefits from transitions

## Resources

### Official Documentation
- [React 18 Working Group Discussions](https://github.com/reactwg/react-18/discussions)
- [Concurrent Features](https://react.dev/blog/2022/03/29/react-v18#new-feature-concurrent-rendering)
- [startTransition API](https://react.dev/reference/react/startTransition)
- [useTransition Hook](https://react.dev/reference/react/useTransition)
- [useDeferredValue Hook](https://react.dev/reference/react/useDeferredValue)

### Articles and Talks
- "Concurrent Rendering in React 18" by Dan Abramov
- "React 18 for Application Developers" - React Conf 2021
- "Understanding Concurrent Rendering" by Kent C. Dodds

### Tools
- React DevTools Profiler (shows concurrent mode)
- Chrome Performance tab (visualize interruptions)

---

*Last Updated: 2026-05 - React 19 current*
