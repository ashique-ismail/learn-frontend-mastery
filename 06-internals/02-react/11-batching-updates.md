# Batching Updates

## The Idea

**In plain English:** Batching updates means React waits until you are done making all your changes before it refreshes the screen, so it only redraws once instead of once for every single change. A "state update" is just telling React "hey, this piece of data changed."

**Real-world analogy:** Imagine you are writing a grocery list and every time you add one item, someone runs to the store immediately. That would be exhausting and slow. Instead, you wait until you have written down everything you need, then send one person on one trip. React does the same thing with screen redraws.

- The grocery list items = state updates (the individual changes you queue up)
- Waiting until the list is complete = batching (React holding off on re-rendering)
- The single trip to the store = one re-render (React updating the screen once with all changes)

---

## Overview

Update batching is React's optimization that groups multiple state updates together into a single re-render. Instead of rendering once per setState call, React batches them and renders once with all updates applied. React 18 significantly improved batching with automatic batching, extending it beyond event handlers to promises, timeouts, and native events.

Understanding batching internals reveals when updates are batched, how to control batching behavior, and why certain patterns perform better than others.

## Pre-React 18 Batching

Before React 18, batching only worked in React event handlers:

```javascript
// Example 1: React 17 batching behavior
function Counter() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);
  
  console.log('Render');
  
  const handleClick = () => {
    // BATCHED in React 17 - one render
    setCount(c => c + 1);
    setFlag(f => !f);
  };
  
  const handleAsync = () => {
    setTimeout(() => {
      // NOT BATCHED in React 17 - two renders
      setCount(c => c + 1);  // Render 1
      setFlag(f => !f);      // Render 2
    }, 1000);
  };
  
  const handlePromise = () => {
    fetch('/api/data').then(() => {
      // NOT BATCHED in React 17 - two renders
      setCount(c => c + 1);  // Render 1
      setFlag(f => !f);      // Render 2
    });
  };
  
  const handleNative = () => {
    document.addEventListener('custom', () => {
      // NOT BATCHED in React 17 - two renders
      setCount(c => c + 1);  // Render 1
      setFlag(f => !f);      // Render 2
    });
  };
}
```

## React 18 Automatic Batching

React 18 batches all updates by default:

```javascript
// Example 2: React 18 automatic batching
function Counter() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);
  
  console.log('Render');
  
  const handleClick = () => {
    // BATCHED - one render
    setCount(c => c + 1);
    setFlag(f => !f);
  };
  
  const handleAsync = () => {
    setTimeout(() => {
      // BATCHED in React 18 - one render
      setCount(c => c + 1);
      setFlag(f => !f);
    }, 1000);
  };
  
  const handlePromise = () => {
    fetch('/api/data').then(() => {
      // BATCHED in React 18 - one render
      setCount(c => c + 1);
      setFlag(f => !f);
    });
  };
  
  const handleNative = () => {
    document.addEventListener('custom', () => {
      // BATCHED in React 18 - one render
      setCount(c => c + 1);
      setFlag(f => !f);
    });
  };
  
  // All scenarios now produce single render
}
```

## Batching Implementation

Batching is implemented through execution contexts:

```javascript
// Example 3: Simplified batching mechanism
let executionContext = NoContext;

const BatchedContext = 0b000001;
const EventContext = 0b000010;
const DiscreteEventContext = 0b000100;

function batchedUpdates(fn) {
  const prevExecutionContext = executionContext;
  executionContext |= BatchedContext;
  
  try {
    return fn();
  } finally {
    executionContext = prevExecutionContext;
    
    if (executionContext === NoContext) {
      // No longer in batched context, flush updates
      flushSyncCallbacks();
    }
  }
}

// Example 4: How React wraps event handlers
function dispatchEvent(event) {
  // Set batched context
  const prevExecutionContext = executionContext;
  executionContext |= EventContext | BatchedContext;
  
  try {
    // Call user's event handler
    return userEventHandler(event);
  } finally {
    // Restore context
    executionContext = prevExecutionContext;
    
    // Flush batched updates
    if (executionContext === NoContext) {
      flushSyncCallbackQueue();
    }
  }
}
```

### Update Queue Processing

```javascript
// Example 5: How updates are queued
function dispatchSetState(fiber, queue, action) {
  // Create update object
  const update = {
    lane: requestUpdateLane(),
    action,
    next: null
  };
  
  // Add to circular queue
  const pending = queue.pending;
  if (pending === null) {
    update.next = update;
  } else {
    update.next = pending.next;
    pending.next = update;
  }
  queue.pending = update;
  
  // Schedule work
  const root = scheduleUpdateOnFiber(fiber, lane);
  
  if (root !== null) {
    entangleTransitionUpdate(root, queue, lane);
  }
}

// Example 6: Processing queued updates
function processUpdateQueue(workInProgress) {
  const queue = workInProgress.updateQueue;
  let firstBaseUpdate = queue.firstBaseUpdate;
  let lastBaseUpdate = queue.lastBaseUpdate;
  
  // Process pending updates
  let pendingQueue = queue.shared.pending;
  if (pendingQueue !== null) {
    queue.shared.pending = null;
    
    // Cut the circular list
    const lastPendingUpdate = pendingQueue;
    const firstPendingUpdate = lastPendingUpdate.next;
    lastPendingUpdate.next = null;
    
    // Append to base queue
    if (lastBaseUpdate === null) {
      firstBaseUpdate = firstPendingUpdate;
    } else {
      lastBaseUpdate.next = firstPendingUpdate;
    }
    lastBaseUpdate = lastPendingUpdate;
  }
  
  // Apply updates sequentially
  if (firstBaseUpdate !== null) {
    let newState = queue.baseState;
    let update = firstBaseUpdate;
    
    do {
      const action = update.action;
      newState = typeof action === 'function'
        ? action(newState)
        : action;
      update = update.next;
    } while (update !== null);
    
    queue.baseState = newState;
  }
}
```

## Batching Scenarios

```javascript
// Example 7: Multiple state updates in one handler
function MultiUpdate() {
  const [count, setCount] = useState(0);
  const [step, setStep] = useState(1);
  const [flag, setFlag] = useState(false);
  
  console.log('Render:', count, step, flag);
  
  const handleClick = () => {
    setCount(c => c + step);
    setStep(s => s + 1);
    setFlag(f => !f);
    
    // Only renders once with all updates applied
  };
  
  return <button onClick={handleClick}>Update</button>;
}

// Example 8: Batching with async operations
function AsyncBatching() {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  
  console.log('Render');
  
  const fetchData = async () => {
    setLoading(true);
    setError(null);
    // Single render for both updates
    
    try {
      const response = await fetch('/api/data');
      const json = await response.json();
      
      setData(json);
      setLoading(false);
      // Single render for both updates (React 18)
    } catch (err) {
      setError(err);
      setLoading(false);
      // Single render for both updates (React 18)
    }
  };
  
  return <button onClick={fetchData}>Fetch</button>;
}

// Example 9: Batching in native event listeners
function NativeEventBatching() {
  const [x, setX] = useState(0);
  const [y, setY] = useState(0);
  
  console.log('Render:', x, y);
  
  useEffect(() => {
    const handleMove = (e) => {
      setX(e.clientX);
      setY(e.clientY);
      // Single render in React 18
      // Two renders in React 17
    };
    
    window.addEventListener('mousemove', handleMove);
    return () => window.removeEventListener('mousemove', handleMove);
  }, []);
  
  return <div>Position: {x}, {y}</div>;
}
```

## flushSync: Opting Out of Batching

Sometimes you need immediate updates:

```javascript
// Example 10: Using flushSync
import { flushSync } from 'react-dom';

function SearchWithHighlight() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const inputRef = useRef();
  
  const handleSearch = () => {
    flushSync(() => {
      setQuery(inputRef.current.value);
    });
    // DOM is updated, can now read it
    
    const inputHeight = inputRef.current.offsetHeight;
    console.log('Input height:', inputHeight);
    
    flushSync(() => {
      setResults(performSearch(query));
    });
    // DOM updated again
    
    scrollToResults();
  };
  
  return (
    <div>
      <input ref={inputRef} />
      <button onClick={handleSearch}>Search</button>
      <Results items={results} />
    </div>
  );
}

// Example 11: flushSync performance cost
function ExpensiveFlushSync() {
  const [items, setItems] = useState([]);
  
  const addMany = () => {
    // BAD: Forces 100 separate renders
    for (let i = 0; i < 100; i++) {
      flushSync(() => {
        setItems(prev => [...prev, i]);
      });
    }
    
    // GOOD: Single batched render
    const newItems = Array.from({ length: 100 }, (_, i) => i);
    setItems(prev => [...prev, ...newItems]);
  };
  
  return <button onClick={addMany}>Add Many</button>;
}

// Example 12: When to use flushSync
function MeasureAfterUpdate() {
  const [height, setHeight] = useState(100);
  const divRef = useRef();
  
  const adjust = () => {
    flushSync(() => {
      setHeight(200);
    });
    // Can now measure new height
    const newHeight = divRef.current.offsetHeight;
    console.log('New height:', newHeight);
  };
  
  return (
    <div ref={divRef} style={{ height }}>
      <button onClick={adjust}>Adjust</button>
    </div>
  );
}
```

## Batching and Concurrent Features

```javascript
// Example 13: Transitions and batching
function SearchWithTransition() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();
  
  const handleChange = (e) => {
    // High priority - not batched with transition
    setQuery(e.target.value);
    
    startTransition(() => {
      // Low priority - batched separately
      const filtered = expensiveFilter(allItems, e.target.value);
      setResults(filtered);
    });
  };
  
  // Two separate render cycles:
  // 1. High priority: query update
  // 2. Low priority: results update (interruptible)
}

// Example 14: Priority-based batching
function PriorityBatching() {
  const [urgent, setUrgent] = useState(0);
  const [lazy, setLazy] = useState(0);
  
  const handleClick = () => {
    // Both urgent updates batched together
    setUrgent(u => u + 1);
    setUrgent(u => u + 1);
    
    startTransition(() => {
      // Lazy updates batched separately
      setLazy(l => l + 1);
      setLazy(l => l + 1);
    });
  };
  
  // Results in two render passes:
  // 1. Urgent updates (high priority)
  // 2. Lazy updates (low priority, later)
}
```

## Custom Batching

```javascript
// Example 15: Manual batching in React 17
import { unstable_batchedUpdates } from 'react-dom';

function React17Batching() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);
  
  const handleAsync = () => {
    setTimeout(() => {
      // Manually batch in React 17
      unstable_batchedUpdates(() => {
        setCount(c => c + 1);
        setFlag(f => !f);
      });
      // Single render
    }, 1000);
  };
  
  return <button onClick={handleAsync}>Async Update</button>;
}

// Example 16: Batching external updates
function ExternalBatching() {
  const [state, setState] = useState(0);
  
  useEffect(() => {
    const ws = new WebSocket('ws://localhost:8080');
    
    ws.onmessage = (event) => {
      const updates = JSON.parse(event.data);
      
      // React 18: Automatically batched
      updates.forEach(update => {
        setState(prev => prev + update.value);
      });
      // Single render regardless of array length
    };
    
    return () => ws.close();
  }, []);
  
  return <div>{state}</div>;
}
```

## Batching and Effects

```javascript
// Example 17: Effects run after batched updates
function EffectTiming() {
  const [count, setCount] = useState(0);
  const [doubled, setDoubled] = useState(0);
  
  useEffect(() => {
    console.log('Effect runs:', count);
  }, [count]);
  
  const handleClick = () => {
    console.log('Before updates');
    
    setCount(c => c + 1);
    setCount(c => c + 1);
    setCount(c => c + 1);
    
    console.log('After updates (queued, not applied)');
  };
  
  // Output when clicked:
  // "Before updates"
  // "After updates (queued, not applied)"
  // [React applies all three updates together]
  // [Component re-renders once]
  // "Effect runs: 3"
  
  return <button onClick={handleClick}>{count}</button>;
}

// Example 18: useLayoutEffect timing
function LayoutEffectBatching() {
  const [count, setCount] = useState(0);
  
  useLayoutEffect(() => {
    console.log('Layout effect:', count);
  }, [count]);
  
  useEffect(() => {
    console.log('Effect:', count);
  }, [count]);
  
  const handleClick = () => {
    setCount(c => c + 1);
    setCount(c => c + 1);
  };
  
  // Output:
  // [Batched updates applied]
  // [DOM mutated]
  // "Layout effect: 2" (synchronous, before paint)
  // [Browser paints]
  // "Effect: 2" (async, after paint)
}
```

## Batching with Class Components

```javascript
// Example 19: Class component batching
class Counter extends React.Component {
  state = { count: 0, flag: false };
  
  handleClick = () => {
    this.setState({ count: this.state.count + 1 });
    this.setState({ flag: !this.state.flag });
    // Batched - one render
    
    console.log(this.state.count); // Still old value
    // setState is async, state not updated yet
  };
  
  handleAsync = () => {
    setTimeout(() => {
      this.setState({ count: this.state.count + 1 });
      this.setState({ flag: !this.state.flag });
      // React 18: Batched
      // React 17: Not batched (two renders)
    }, 0);
  };
  
  render() {
    console.log('Render');
    return <button onClick={this.handleClick}>{this.state.count}</button>;
  }
}

// Example 20: Class component callbacks
class CallbackTiming extends React.Component {
  state = { count: 0 };
  
  handleClick = () => {
    this.setState(
      { count: this.state.count + 1 },
      () => {
        // Callback runs after update is committed
        console.log('Count updated:', this.state.count);
      }
    );
    
    this.setState(
      { count: this.state.count + 2 },
      () => {
        console.log('Count updated again:', this.state.count);
      }
    );
    
    // Both updates batched, callbacks run after single render
  };
  
  render() {
    return <button onClick={this.handleClick}>{this.state.count}</button>;
  }
}
```

## Edge Cases

```javascript
// Example 21: Reading state immediately after setState
function StateReadTiming() {
  const [count, setCount] = useState(0);
  
  const handleClick = () => {
    setCount(count + 1);
    console.log(count); // Logs old value (0)
    
    setCount(c => {
      console.log(c); // Logs intermediate value (1)
      return c + 1;
    });
    
    console.log(count); // Still logs old value (0)
    
    // State updates are queued, not applied immediately
    // Use functional updates to work with latest state
  };
  
  return <button onClick={handleClick}>{count}</button>;
}

// Example 22: Batching with refs
function RefTiming() {
  const [count, setCount] = useState(0);
  const countRef = useRef(0);
  
  const handleClick = () => {
    setCount(c => c + 1);
    setCount(c => c + 1);
    
    // Ref updates are immediate
    countRef.current += 1;
    console.log('Ref:', countRef.current); // Logs new value
    console.log('State:', count); // Logs old value
  };
  
  return <button onClick={handleClick}>{count}</button>;
}
```

## Common Misconceptions

1. **"setState is asynchronous"** - Misleading. setState schedules an update but may flush synchronously (with flushSync) or be batched with others. It's not truly async like a Promise.

2. **"Batching only works in event handlers"** - False in React 18. Automatic batching works everywhere: timeouts, promises, native events, etc.

3. **"Multiple setState calls are bad"** - False. They're batched together. Multiple calls are often clearer than one complex update.

4. **"flushSync makes updates synchronous"** - Partially true. It forces immediate render but doesn't make setState itself synchronous—you still can't read the new state in the same function scope.

5. **"Batching is always good"** - Mostly true, but sometimes you need immediate updates (measuring DOM, integrating third-party libraries). That's when flushSync is necessary.

## Performance Implications

1. **Render Frequency** - Batching reduces render count dramatically. Without batching, n setState calls cause n renders. With batching, they cause 1 render.

2. **flushSync Cost** - Each flushSync call forces a synchronous render, blocking the main thread. Use sparingly—only when you must read DOM immediately after update.

3. **Automatic Batching Overhead** - Minimal. React tracks execution context with simple bitmask operations. The performance gain from reduced renders far outweighs the tracking cost.

4. **Effect Timing** - Fewer renders mean fewer effect executions. This compounds performance benefits since effects often do expensive work.

## Interview Questions

1. **Q: What is update batching in React?**
   A: Batching is React's optimization that groups multiple state updates into a single re-render. Instead of rendering once per setState call, React accumulates updates in a queue and applies them together in one render pass, improving performance.

2. **Q: How did batching change in React 18?**
   A: React 17 only batched updates inside React event handlers. React 18 introduced automatic batching that works everywhere: timeouts, promises, native event handlers, and async functions. This makes batching consistent and eliminates a major performance footgun.

3. **Q: How does React implement batching?**
   A: React uses execution contexts tracked with bit flags. When entering a batched context (like an event handler), React sets a flag, queues updates instead of processing them immediately, then flushes the queue when exiting the context. All queued updates are applied together in one render.

4. **Q: What is flushSync and when would you use it?**
   A: flushSync forces React to apply updates immediately and synchronously, opting out of batching. Use it when you must read DOM measurements immediately after a state update, like measuring element dimensions or integrating with third-party libraries that need up-to-date DOM.

5. **Q: Why can't you read state immediately after setState?**
   A: setState queues an update but doesn't apply it immediately. React batches updates and processes them later. The state variable in your scope is a constant from that render and won't change. Use functional updates or refs to work with the latest values.

6. **Q: How does batching interact with concurrent features?**
   A: Updates with different priorities are batched separately. High-priority updates (user input) are batched and rendered together. Low-priority updates (transitions) are batched separately and can be interrupted. This allows responsive UI while expensive updates happen in the background.

7. **Q: Do effects run after each setState or after batched updates?**
   A: Effects run once after all batched updates are applied and committed. If three setState calls are batched into one render, effects run once with the final state, not three times with intermediate states.

8. **Q: Can you manually batch updates in React 17?**
   A: Yes, using unstable_batchedUpdates from react-dom. This wraps updates in a batched context. However, in React 18, this is unnecessary since batching is automatic everywhere.

## Key Takeaways

1. React 18 automatically batches all updates regardless of where they occur
2. Batching reduces render count by applying multiple updates in one render
3. React uses execution contexts to track when updates should be batched
4. flushSync opts out of batching for immediate synchronous updates
5. setState queues updates; state isn't updated immediately after the call
6. Different priority updates are batched separately
7. Effects run once after batched updates are committed
8. Batching significantly improves performance with minimal overhead

## Resources

- [Automatic Batching in React 18](https://react.dev/blog/2022/03/29/react-v18#new-feature-automatic-batching)
- [flushSync Documentation](https://react.dev/reference/react-dom/flushSync)
- [React 18 Working Group: Automatic Batching](https://github.com/reactwg/react-18/discussions/21)
- [Batching Source Code](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberWorkLoop.js)
- [State Updates Are Batched](https://legacy.reactjs.org/docs/state-and-lifecycle.html#state-updates-may-be-asynchronous)
