# Automatic Batching in React 18

## The Idea

**In plain English:** Automatic batching is React's way of collecting several changes you want to make to your app's data and applying them all at once in a single screen update, instead of updating the screen after every single change. Think of "state" as the data your app remembers (like a counter number or whether a menu is open).

**Real-world analogy:** Imagine a waiter at a restaurant taking orders from a table. Instead of sprinting to the kitchen after every single person orders, the waiter waits until the whole table has finished ordering, then makes one trip to the kitchen with all the orders at once.

- The waiter collecting orders = React queuing up state updates
- Each person at the table placing an order = each `setState` call in your code
- The single trip to the kitchen = the one re-render React performs after batching all updates

---

## Table of Contents

- [Introduction](#introduction)
- [What is Batching?](#what-is-batching)
- [React 17 vs React 18 Batching](#react-17-vs-react-18-batching)
- [How Automatic Batching Works](#how-automatic-batching-works)
- [Batching in Different Scenarios](#batching-in-different-scenarios)
- [Opting Out with flushSync](#opting-out-with-flushsync)
- [Performance Implications](#performance-implications)
- [Common Mistakes](#common-mistakes)
- [Best Practices](#best-practices)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Introduction

Automatic batching is one of React 18's performance improvements that extends batching to all updates, regardless of where they originate. Previously, React only batched updates inside event handlers; now it batches updates in promises, timeouts, and native event handlers as well.

## What is Batching?

Batching is when React groups multiple state updates into a single re-render for better performance. Instead of re-rendering after each state update, React waits until all updates in the same event are complete, then performs one re-render.

**Without Batching (Multiple Re-renders):**

```typescript
function Counter() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  // Without batching: 2 re-renders
  const handleClick = () => {
    setCount(c => c + 1); // Re-render 1
    setFlag(f => !f);     // Re-render 2
  };

  console.log('Rendered'); // Logs twice per click

  return (
    <>
      <button onClick={handleClick}>Click</button>
      <div>Count: {count}</div>
      <div>Flag: {flag.toString()}</div>
    </>
  );
}
```

**With Batching (Single Re-render):**

```typescript
function Counter() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  // With batching: 1 re-render
  const handleClick = () => {
    setCount(c => c + 1); // Queued
    setFlag(f => !f);     // Queued
    // React batches both updates → 1 re-render
  };

  console.log('Rendered'); // Logs once per click

  return (
    <>
      <button onClick={handleClick}>Click</button>
      <div>Count: {count}</div>
      <div>Flag: {flag.toString()}</div>
    </>
  );
}
```

## React 17 vs React 18 Batching

### React 17: Limited Batching

React 17 only batched updates inside React event handlers:

```typescript
// REACT 17 BEHAVIOR

function React17Example() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  // ✅ BATCHED in React 17 (inside event handler)
  const handleClick = () => {
    setCount(c => c + 1);
    setFlag(f => !f);
    // Only 1 re-render
  };

  // ❌ NOT BATCHED in React 17 (inside timeout)
  const handleClickWithTimeout = () => {
    setTimeout(() => {
      setCount(c => c + 1); // Re-render 1
      setFlag(f => !f);     // Re-render 2
    }, 0);
  };

  // ❌ NOT BATCHED in React 17 (inside promise)
  const handleClickWithPromise = () => {
    fetch('/api/data').then(() => {
      setCount(c => c + 1); // Re-render 1
      setFlag(f => !f);     // Re-render 2
    });
  };

  // ❌ NOT BATCHED in React 17 (native event listener)
  useEffect(() => {
    const handler = () => {
      setCount(c => c + 1); // Re-render 1
      setFlag(f => !f);     // Re-render 2
    };
    window.addEventListener('resize', handler);
    return () => window.removeEventListener('resize', handler);
  }, []);

  return <button onClick={handleClick}>Click</button>;
}
```

### React 18: Automatic Batching Everywhere

React 18 batches updates in all scenarios:

```typescript
// REACT 18 BEHAVIOR

function React18Example() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  // ✅ BATCHED in React 18
  const handleClick = () => {
    setCount(c => c + 1);
    setFlag(f => !f);
    // 1 re-render
  };

  // ✅ NOW BATCHED in React 18 (timeout)
  const handleClickWithTimeout = () => {
    setTimeout(() => {
      setCount(c => c + 1);
      setFlag(f => !f);
      // 1 re-render (batched!)
    }, 0);
  };

  // ✅ NOW BATCHED in React 18 (promise)
  const handleClickWithPromise = async () => {
    await fetch('/api/data');
    setCount(c => c + 1);
    setFlag(f => !f);
    // 1 re-render (batched!)
  };

  // ✅ NOW BATCHED in React 18 (native event)
  useEffect(() => {
    const handler = () => {
      setCount(c => c + 1);
      setFlag(f => !f);
      // 1 re-render (batched!)
    };
    window.addEventListener('resize', handler);
    return () => window.removeEventListener('resize', handler);
  }, []);

  return <button onClick={handleClick}>Click</button>;
}
```

## How Automatic Batching Works

### Batching Mechanism

React 18 uses a queuing system to batch updates:

```typescript
function BatchingDemo() {
  const [state1, setState1] = useState(0);
  const [state2, setState2] = useState(0);
  const [state3, setState3] = useState(0);

  const renderCount = useRef(0);
  renderCount.current++;

  const updateMultipleStates = () => {
    console.log('Before updates');
    
    setState1(1); // Queued
    console.log('After setState1');
    
    setState2(2); // Queued
    console.log('After setState2');
    
    setState3(3); // Queued
    console.log('After setState3');
    
    console.log('All updates queued');
    // React processes queue → 1 re-render
  };

  return (
    <>
      <button onClick={updateMultipleStates}>Update All</button>
      <div>Render count: {renderCount.current}</div>
      <div>State1: {state1}</div>
      <div>State2: {state2}</div>
      <div>State3: {state3}</div>
    </>
  );
}
```

### Updates in Different Contexts

```typescript
function ContextualBatching() {
  const [count, setCount] = useState(0);
  const [text, setText] = useState('');

  // Event handler: batched
  const handleClick = () => {
    setCount(c => c + 1);
    setText('clicked');
    // 1 re-render
  };

  // Timeout: batched in React 18
  const handleTimeout = () => {
    setTimeout(() => {
      setCount(c => c + 1);
      setText('timeout');
      // 1 re-render
    }, 1000);
  };

  // Promise: batched in React 18
  const handlePromise = async () => {
    await Promise.resolve();
    setCount(c => c + 1);
    setText('promise');
    // 1 re-render
  };

  // Fetch: batched in React 18
  const handleFetch = async () => {
    const res = await fetch('/api/data');
    const data = await res.json();
    setCount(c => c + 1);
    setText(data.message);
    // 1 re-render
  };

  return (
    <>
      <button onClick={handleClick}>Click</button>
      <button onClick={handleTimeout}>Timeout</button>
      <button onClick={handlePromise}>Promise</button>
      <button onClick={handleFetch}>Fetch</button>
      <div>Count: {count}</div>
      <div>Text: {text}</div>
    </>
  );
}
```

## Batching in Different Scenarios

### 1. Event Handlers (Always Batched)

```typescript
function EventHandlerBatching() {
  const [count, setCount] = useState(0);
  const [doubled, setDoubled] = useState(0);

  const handleClick = () => {
    setCount(c => c + 1);
    setDoubled(d => d + 2);
    // Batched: 1 re-render
  };

  const handleMouseMove = (e: React.MouseEvent) => {
    setCount(e.clientX);
    setDoubled(e.clientX * 2);
    // Batched: 1 re-render
  };

  return (
    <div onMouseMove={handleMouseMove}>
      <button onClick={handleClick}>Update</button>
      <div>Count: {count}</div>
      <div>Doubled: {doubled}</div>
    </div>
  );
}
```

### 2. Timeouts and Intervals

```typescript
function TimeoutBatching() {
  const [count, setCount] = useState(0);
  const [status, setStatus] = useState('idle');

  const startTimeout = () => {
    setTimeout(() => {
      setCount(c => c + 1);
      setStatus('updated');
      // React 18: Batched → 1 re-render
      // React 17: NOT batched → 2 re-renders
    }, 1000);
  };

  const startInterval = () => {
    const id = setInterval(() => {
      setCount(c => c + 1);
      setStatus(`Updated at ${Date.now()}`);
      // React 18: Batched → 1 re-render per interval
    }, 1000);

    setTimeout(() => clearInterval(id), 5000);
  };

  return (
    <>
      <button onClick={startTimeout}>Start Timeout</button>
      <button onClick={startInterval}>Start Interval</button>
      <div>Count: {count}</div>
      <div>Status: {status}</div>
    </>
  );
}
```

### 3. Promises and Async/Await

```typescript
function PromiseBatching() {
  const [data, setData] = useState<any>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const fetchData = async () => {
    try {
      setLoading(true);
      setError(null);
      // React 18: These 2 updates are batched

      const response = await fetch('/api/data');
      const result = await response.json();

      setData(result);
      setLoading(false);
      // React 18: These 2 updates are batched
      // React 17: NOT batched → 2 re-renders
    } catch (err) {
      setError(err.message);
      setLoading(false);
      // React 18: Batched
    }
  };

  return (
    <>
      <button onClick={fetchData} disabled={loading}>
        Fetch Data
      </button>
      {loading && <div>Loading...</div>}
      {error && <div>Error: {error}</div>}
      {data && <div>Data: {JSON.stringify(data)}</div>}
    </>
  );
}
```

### 4. Native Event Listeners

```typescript
function NativeEventBatching() {
  const [x, setX] = useState(0);
  const [y, setY] = useState(0);
  const [count, setCount] = useState(0);

  useEffect(() => {
    // Native event listener
    const handleMouseMove = (e: MouseEvent) => {
      setX(e.clientX);
      setY(e.clientY);
      setCount(c => c + 1);
      // React 18: Batched → 1 re-render
      // React 17: NOT batched → 3 re-renders
    };

    window.addEventListener('mousemove', handleMouseMove);
    return () => window.removeEventListener('mousemove', handleMouseMove);
  }, []);

  return (
    <>
      <div>X: {x}, Y: {y}</div>
      <div>Updates: {count}</div>
    </>
  );
}
```

### 5. Multiple State Updates in Custom Hooks

```typescript
function useMultipleUpdates() {
  const [count, setCount] = useState(0);
  const [timestamp, setTimestamp] = useState(Date.now());
  const [status, setStatus] = useState('idle');

  const update = () => {
    setCount(c => c + 1);
    setTimestamp(Date.now());
    setStatus('updated');
    // All batched: 1 re-render
  };

  return { count, timestamp, status, update };
}

function CustomHookBatching() {
  const state = useMultipleUpdates();

  return (
    <>
      <button onClick={state.update}>Update</button>
      <div>Count: {state.count}</div>
      <div>Time: {state.timestamp}</div>
      <div>Status: {state.status}</div>
    </>
  );
}
```

## Opting Out with flushSync

Sometimes you need immediate updates without batching. Use `flushSync` to force synchronous updates:

### Basic flushSync Usage

```typescript
import { flushSync } from 'react-dom';

function FlushSyncExample() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  const handleClick = () => {
    // Force immediate update (no batching)
    flushSync(() => {
      setCount(c => c + 1);
    });
    // Re-render happens here

    console.log('Count updated:', count); // Still shows old value
    // (state updates are async, even with flushSync)

    // Separate update, separate re-render
    flushSync(() => {
      setFlag(f => !f);
    });
    // Another re-render happens here
  };

  return (
    <>
      <button onClick={handleClick}>Update</button>
      <div>Count: {count}</div>
      <div>Flag: {flag.toString()}</div>
    </>
  );
}
```

### When to Use flushSync

```typescript
// Use case 1: DOM measurements between updates
function MeasurementExample() {
  const [height, setHeight] = useState(0);
  const [items, setItems] = useState<string[]>([]);
  const listRef = useRef<HTMLDivElement>(null);

  const addItem = (item: string) => {
    flushSync(() => {
      setItems(prev => [...prev, item]);
    });
    // DOM is updated, can measure immediately
    
    if (listRef.current) {
      setHeight(listRef.current.scrollHeight);
    }
  };

  return (
    <>
      <button onClick={() => addItem('New Item')}>Add</button>
      <div ref={listRef}>
        {items.map((item, i) => <div key={i}>{item}</div>)}
      </div>
      <div>Height: {height}px</div>
    </>
  );
}

// Use case 2: Third-party library integration
function ThirdPartyIntegration() {
  const [data, setData] = useState<Data[]>([]);
  const chartRef = useRef<ChartInstance>(null);

  const updateData = (newData: Data[]) => {
    flushSync(() => {
      setData(newData);
    });
    // DOM updated, safe to call chart library
    
    chartRef.current?.update();
  };

  return <Chart ref={chartRef} data={data} />;
}

// Use case 3: Focus management
function FocusExample() {
  const [items, setItems] = useState<string[]>(['Item 1']);
  const newInputRef = useRef<HTMLInputElement>(null);

  const addItem = () => {
    flushSync(() => {
      setItems(prev => [...prev, `Item ${prev.length + 1}`]);
    });
    // DOM updated, new input exists
    
    newInputRef.current?.focus();
  };

  return (
    <>
      <button onClick={addItem}>Add Item</button>
      {items.map((item, i) => (
        <input
          key={i}
          ref={i === items.length - 1 ? newInputRef : null}
          defaultValue={item}
        />
      ))}
    </>
  );
}
```

### flushSync Performance Considerations

```typescript
// WRONG: Overusing flushSync
function BadFlushSync() {
  const [items, setItems] = useState<number[]>([]);

  const addManyItems = () => {
    for (let i = 0; i < 100; i++) {
      flushSync(() => {
        setItems(prev => [...prev, i]);
      });
      // 100 separate re-renders! Very slow!
    }
  };

  return <button onClick={addManyItems}>Add Items</button>;
}

// CORRECT: Let React batch
function GoodBatching() {
  const [items, setItems] = useState<number[]>([]);

  const addManyItems = () => {
    const newItems = Array.from({ length: 100 }, (_, i) => i);
    setItems(prev => [...prev, ...newItems]);
    // 1 re-render, much faster!
  };

  return <button onClick={addManyItems}>Add Items</button>;
}

// CORRECT: flushSync only when necessary
function GoodFlushSync() {
  const [items, setItems] = useState<number[]>([]);
  const listRef = useRef<HTMLDivElement>(null);

  const addManyItems = () => {
    const newItems = Array.from({ length: 100 }, (_, i) => i);
    
    flushSync(() => {
      setItems(prev => [...prev, ...newItems]);
    });
    // Single forced re-render, then measure
    
    listRef.current?.scrollTo(0, listRef.current.scrollHeight);
  };

  return (
    <>
      <button onClick={addManyItems}>Add Items</button>
      <div ref={listRef}>
        {items.map(item => <div key={item}>{item}</div>)}
      </div>
    </>
  );
}
```

## Performance Implications

### Measuring Batching Impact

```typescript
function PerformanceDemo() {
  const [count, setCount] = useState(0);
  const [doubled, setDoubled] = useState(0);
  const [tripled, setTripled] = useState(0);
  
  const renderCount = useRef(0);
  renderCount.current++;

  // Without batching: 3 re-renders
  const updateWithFlushSync = () => {
    flushSync(() => setCount(c => c + 1));
    flushSync(() => setDoubled(d => d + 2));
    flushSync(() => setTripled(t => t + 3));
  };

  // With batching: 1 re-render
  const updateWithBatching = () => {
    setCount(c => c + 1);
    setDoubled(d => d + 2);
    setTripled(t => t + 3);
  };

  return (
    <>
      <button onClick={updateWithFlushSync}>
        Update (No Batch): {renderCount.current} renders
      </button>
      <button onClick={updateWithBatching}>
        Update (Batched): {renderCount.current} renders
      </button>
      <div>Count: {count}</div>
      <div>Doubled: {doubled}</div>
      <div>Tripled: {tripled}</div>
    </>
  );
}
```

### Real-World Performance Example

```typescript
interface FormData {
  name: string;
  email: string;
  phone: string;
  address: string;
}

function ComplexForm() {
  const [formData, setFormData] = useState<FormData>({
    name: '',
    email: '',
    phone: '',
    address: ''
  });
  const [errors, setErrors] = useState<Partial<FormData>>({});
  const [touched, setTouched] = useState<Partial<Record<keyof FormData, boolean>>>({});
  const [isValid, setIsValid] = useState(false);

  const validateAndUpdate = async (field: keyof FormData, value: string) => {
    // All 4 updates are batched in React 18
    setFormData(prev => ({ ...prev, [field]: value }));
    setTouched(prev => ({ ...prev, [field]: true }));
    
    const error = validateField(field, value);
    setErrors(prev => ({ ...prev, [field]: error }));
    
    const allValid = checkAllValid(formData, errors);
    setIsValid(allValid);
    
    // Only 1 re-render in React 18
    // Would be 4 re-renders in React 17 (if in async context)
  };

  return <form>{/* Form fields */}</form>;
}
```

## Common Mistakes

### 1. Expecting Immediate State Values

```typescript
// WRONG: Expecting updated value immediately
function WrongExpectation() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(c => c + 1);
    console.log(count); // Still old value!
    // Batching doesn't change this behavior
  };

  return <button onClick={handleClick}>Click</button>;
}

// CORRECT: Use functional updates or useEffect
function CorrectApproach() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(c => {
      console.log('New count will be:', c + 1);
      return c + 1;
    });
  };

  useEffect(() => {
    console.log('Count updated to:', count);
  }, [count]);

  return <button onClick={handleClick}>Click</button>;
}
```

### 2. Overusing flushSync

```typescript
// WRONG: Unnecessary flushSync
function UnnecessaryFlushSync() {
  const [data, setData] = useState([]);

  const loadData = async () => {
    const response = await fetch('/api/data');
    const result = await response.json();
    
    flushSync(() => {
      setData(result); // No need for flushSync here!
    });
  };

  return <button onClick={loadData}>Load</button>;
}

// CORRECT: Let React batch naturally
function NaturalBatching() {
  const [data, setData] = useState([]);

  const loadData = async () => {
    const response = await fetch('/api/data');
    const result = await response.json();
    setData(result); // Batched automatically
  };

  return <button onClick={loadData}>Load</button>;
}
```

### 3. Assuming Synchronous Updates

```typescript
// WRONG: Chaining state-dependent logic
function WrongChaining() {
  const [count, setCount] = useState(0);
  const [doubled, setDoubled] = useState(0);

  const handleClick = () => {
    setCount(c => c + 1);
    setDoubled(count * 2); // Uses OLD count!
  };

  return <button onClick={handleClick}>Update</button>;
}

// CORRECT: Use functional updates
function CorrectChaining() {
  const [count, setCount] = useState(0);
  const [doubled, setDoubled] = useState(0);

  const handleClick = () => {
    setCount(c => {
      const newCount = c + 1;
      setDoubled(newCount * 2); // Still not ideal
      return newCount;
    });
  };

  return <button onClick={handleClick}>Update</button>;
}

// BETTER: Derive state
function BestApproach() {
  const [count, setCount] = useState(0);
  const doubled = count * 2; // Derived, always correct

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>Update</button>
      <div>Count: {count}</div>
      <div>Doubled: {doubled}</div>
    </>
  );
}
```

## Best Practices

### 1. Embrace Automatic Batching

```typescript
// Good: Let React batch automatically
function AutomaticBatchingExample() {
  const [loading, setLoading] = useState(false);
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);

  const fetchData = async () => {
    setLoading(true);
    setError(null);
    // Batched

    try {
      const result = await fetch('/api/data');
      const json = await result.json();
      
      setData(json);
      setLoading(false);
      // Batched in React 18
    } catch (err) {
      setError(err.message);
      setLoading(false);
      // Batched in React 18
    }
  };

  return (
    <>
      <button onClick={fetchData}>Fetch</button>
      {loading && <Spinner />}
      {error && <Error message={error} />}
      {data && <DataDisplay data={data} />}
    </>
  );
}
```

### 2. Use flushSync Sparingly

```typescript
// Good: Only use flushSync when truly needed
function SparsFlushSyncUsage() {
  const [items, setItems] = useState<string[]>([]);
  const containerRef = useRef<HTMLDivElement>(null);

  const addAndScrollToItem = (item: string) => {
    // Need DOM to be updated before scrolling
    flushSync(() => {
      setItems(prev => [...prev, item]);
    });

    // Now safe to scroll
    containerRef.current?.scrollTo({
      top: containerRef.current.scrollHeight,
      behavior: 'smooth'
    });
  };

  return (
    <>
      <button onClick={() => addAndScrollToItem('New item')}>
        Add Item
      </button>
      <div ref={containerRef} style={{ maxHeight: 200, overflow: 'auto' }}>
        {items.map((item, i) => <div key={i}>{item}</div>)}
      </div>
    </>
  );
}
```

### 3. Prefer Derived State

```typescript
// Good: Derive state instead of syncing multiple states
function DerivedStateExample() {
  const [items, setItems] = useState<Item[]>([]);
  const [filter, setFilter] = useState('');

  // Derived values (no extra state)
  const filteredItems = useMemo(
    () => items.filter(item => item.name.includes(filter)),
    [items, filter]
  );
  const itemCount = filteredItems.length;
  const isEmpty = itemCount === 0;

  return (
    <>
      <input value={filter} onChange={e => setFilter(e.target.value)} />
      <div>Showing {itemCount} items</div>
      {isEmpty ? <EmptyState /> : <ItemList items={filteredItems} />}
    </>
  );
}
```

## Interview Questions

### Q1: What is automatic batching in React 18?

**Answer:** Automatic batching is when React groups multiple state updates into a single re-render, regardless of where the updates occur. In React 18, batching happens automatically in event handlers, timeouts, promises, native event handlers, and any other context. Previously in React 17, batching only occurred in React event handlers.

### Q2: How does batching improve performance?

**Answer:** Batching reduces the number of re-renders by grouping state updates. Instead of re-rendering after each setState call, React batches all updates that happen in the same event/task and performs one re-render. This means fewer component re-executions, fewer DOM updates, and fewer side effects running, resulting in better performance.

### Q3: When should you use flushSync?

**Answer:** Use flushSync when you need the DOM to be updated immediately before proceeding, such as:

1. Measuring DOM elements after a state update
2. Integrating with third-party libraries that need updated DOM
3. Managing focus after adding/removing elements
4. Scrolling to newly added content

Avoid overusing it, as it bypasses React's optimizations.

### Q4: Does batching change how state updates work?

**Answer:** No, state updates are still asynchronous and work the same way. Batching only affects when re-renders happen, not the semantics of state updates. You still can't access updated state immediately after setState, and you should still use functional updates when depending on previous state.

### Q5: How can you opt out of automatic batching?

**Answer:** Use `flushSync` from 'react-dom' to force immediate synchronous updates without batching. However, this should be rare and only used when you specifically need the DOM to be updated before continuing execution.

## Key Takeaways

1. **Automatic batching is automatic** - Works everywhere in React 18 without configuration
2. **Major performance improvement** - Reduces re-renders from multiple state updates
3. **Behavior change from React 17** - Updates in timeouts/promises are now batched
4. **State updates still async** - Batching doesn't make setState synchronous
5. **flushSync for opt-out** - Use sparingly when you need immediate DOM updates
6. **Backward compatible** - Existing code continues to work, just faster
7. **Prefer derived state** - Reduces need for multiple coordinated state updates
8. **No breaking changes** - App should work the same, just with better performance

## Resources

### Official Documentation

- [Automatic Batching in React 18](https://react.dev/blog/2022/03/29/react-v18#new-feature-automatic-batching)
- [flushSync API](https://react.dev/reference/react-dom/flushSync)
- [React 18 Upgrade Guide](https://react.dev/blog/2022/03/08/react-18-upgrade-guide)

### Articles

- "Automatic Batching for Fewer Renders" - React Team
- "Understanding React 18 Batching" by Kent C. Dodds
- "Performance Improvements in React 18" - Web.dev

### Tools

- React DevTools Profiler (visualize re-renders)
- React DevTools Components (see state updates)

---

Last Updated: 2026-05 - React 19 current
