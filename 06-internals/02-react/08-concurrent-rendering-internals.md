# Concurrent Rendering Internals

## The Idea

**In plain English:** Concurrent rendering is React's ability to pause the work of updating the screen midway through, handle something more urgent, and then come back to finish where it left off — so the page never feels frozen even when there is a lot going on.

**Real-world analogy:** Imagine a chef cooking a big stew (a long task) but also watching for a kitchen timer to go off. Every few minutes the chef puts down the ladle, checks if anything urgent needs attention (a boiling pot, a customer request), handles it, then returns to the stew right where they left off.

- The chef = React's rendering engine
- The stew = a low-priority, expensive screen update (like rendering a huge list)
- Checking the timer = React pausing every ~5ms to ask "is there something more urgent?"
- A customer order = a high-priority update (like a user typing in an input box)

---

## Overview

Concurrent rendering is React's ability to interrupt rendering work, prioritize updates, and resume work later. This fundamental shift from synchronous to interruptible rendering enables features like time-slicing, Suspense, transitions, and responsive UIs even under heavy load.

Understanding concurrent rendering internals—the work loop, priority lanes, time-slicing mechanism, and scheduling—reveals how React keeps applications responsive and why certain patterns work better than others.

## The Problem: Synchronous Rendering

Before concurrent features, React rendering was synchronous and blocking:

```javascript
// Example 1: Synchronous rendering problem
function HeavyComponent() {
  // Expensive computation
  const items = Array.from({ length: 10000 }, (_, i) => ({
    id: i,
    value: Math.random()
  }));
  
  return (
    <ul>
      {items.map(item => (
        <ExpensiveItem key={item.id} item={item} />
      ))}
    </ul>
  );
}

// In synchronous rendering:
// 1. User clicks button → triggers update
// 2. React starts rendering HeavyComponent
// 3. CPU works for 500ms rendering 10,000 items
// 4. Main thread blocked - UI frozen
// 5. User clicks don't register, animations stutter
// 6. Finally, update commits to DOM
//
// Problem: No way to interrupt or prioritize other work
```

Concurrent rendering solves this by making rendering interruptible:

```javascript
// Example 2: Concurrent rendering solution
function ConcurrentHeavyComponent() {
  const [isPending, startTransition] = useTransition();
  const [items, setItems] = useState([]);
  
  const loadItems = () => {
    startTransition(() => {
      // This update is interruptible
      const newItems = Array.from({ length: 10000 }, (_, i) => ({
        id: i,
        value: Math.random()
      }));
      setItems(newItems);
    });
  };
  
  return (
    <div>
      <button onClick={loadItems}>Load Items</button>
      {isPending && <Spinner />}
      <ul>
        {items.map(item => (
          <ExpensiveItem key={item.id} item={item} />
        ))}
      </ul>
    </div>
  );
}

// With concurrent rendering:
// 1. User clicks button
// 2. React starts rendering in background
// 3. After 5ms of work, React yields to browser
// 4. Browser handles user input, animations
// 5. React resumes rendering
// 6. Repeats until complete
// 7. Commits when ready
//
// Result: UI stays responsive throughout
```

## The Work Loop

The core of concurrent rendering is the work loop that can be interrupted:

```javascript
// Example 3: Simplified work loop (conceptual)
function workLoopConcurrent() {
  // Work on fibers until we run out of time or finish
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}

function shouldYield() {
  // Check if we've used our time slice
  const currentTime = getCurrentTime();
  if (currentTime >= deadline) {
    // Yield control back to browser
    return true;
  }
  return false;
}

// Example 4: Time slicing mechanism
let deadline = 0;
const frameInterval = 5; // 5ms time slices

function scheduleCallback(callback) {
  const currentTime = getCurrentTime();
  const timeout = frameInterval;
  const expirationTime = currentTime + timeout;
  
  const newTask = {
    callback,
    expirationTime,
    priorityLevel: currentPriorityLevel
  };
  
  push(taskQueue, newTask);
  
  // Start the work loop
  requestHostCallback(flushWork);
}

function flushWork(initialTime) {
  deadline = initialTime + frameInterval;
  
  try {
    // Work on tasks until time is up
    return workLoop(initialTime);
  } finally {
    deadline = 0;
  }
}

function workLoop(initialTime) {
  let currentTask = peek(taskQueue);
  
  while (currentTask !== null) {
    if (currentTask.expirationTime > initialTime && shouldYield()) {
      // Time slice expired, yield to browser
      break;
    }
    
    const callback = currentTask.callback;
    if (callback !== null) {
      const continuationCallback = callback();
      
      if (typeof continuationCallback === 'function') {
        // Work isn't finished, continue later
        currentTask.callback = continuationCallback;
      } else {
        // Task completed, remove from queue
        pop(taskQueue);
      }
    }
    
    currentTask = peek(taskQueue);
  }
  
  // Return true if there's more work
  return currentTask !== null;
}
```

## Priority Lanes

React uses a lane-based priority system for updates:

```javascript
// Example 5: Lane priorities (simplified)
const SyncLane = 0b0000000000000000000000000000001;
const InputContinuousLane = 0b0000000000000000000000000000100;
const DefaultLane = 0b0000000000000000000000000010000;
const TransitionLane1 = 0b0000000000000000000000001000000;
const TransitionLane2 = 0b0000000000000000000000010000000;
const IdleLane = 0b0100000000000000000000000000000;

// Example 6: How lanes are assigned to updates
function dispatchAction(fiber, queue, action) {
  // Determine priority based on context
  const lane = requestUpdateLane(fiber);
  
  const update = {
    lane,
    action,
    next: null
  };
  
  // Add to queue with priority
  enqueueUpdate(fiber, queue, update, lane);
  scheduleUpdateOnFiber(fiber, lane);
}

function requestUpdateLane(fiber) {
  // During user input
  if (isInputEvent) {
    return InputContinuousLane;
  }
  
  // During transition
  if (isTransition) {
    return TransitionLane1;
  }
  
  // Default priority
  return DefaultLane;
}

// Example 7: Priority comparison
function UpdatePriority() {
  const [count, setCount] = useState(0);
  const [isPending, startTransition] = useTransition();
  
  const handleClick = () => {
    // High priority - immediate
    setCount(c => c + 1);
    
    // Low priority - can be interrupted
    startTransition(() => {
      // Expensive update
      performHeavyCalculation();
    });
  };
  
  // If user clicks multiple times:
  // 1. High priority updates (setCount) complete first
  // 2. Low priority updates (transition) can be interrupted
  // 3. New high priority updates preempt low priority work
}
```

## Rendering Phases

```javascript
// Example 8: Render phase (can be interrupted)
function renderRootConcurrent(root, lanes) {
  // Render phase - pure, can be interrupted
  do {
    try {
      workLoopConcurrent();
      break;
    } catch (thrownValue) {
      handleError(root, thrownValue);
    }
  } while (true);
  
  // If interrupted, return continuation
  if (workInProgress !== null) {
    return RootInProgress;
  }
  
  // Completed, move to commit
  return RootCompleted;
}

function workLoopConcurrent() {
  // Work until interrupted
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}

// Example 9: Commit phase (cannot be interrupted)
function commitRoot(root) {
  // Commit phase - has side effects, must be atomic
  const finishedWork = root.finishedWork;
  
  // Phase 1: Before mutation
  commitBeforeMutationEffects(finishedWork);
  
  // Phase 2: Mutation (update DOM)
  commitMutationEffects(finishedWork);
  
  // Phase 3: Layout effects
  commitLayoutEffects(finishedWork);
  
  // Phase 4: Passive effects (scheduled)
  schedulePassiveEffects(finishedWork);
}
```

## startTransition Implementation

```javascript
// Example 10: Transition API internals
function startTransition(callback) {
  // Mark transition start
  const prevTransition = ReactCurrentBatchConfig.transition;
  ReactCurrentBatchConfig.transition = 1;
  
  try {
    // Updates inside have low priority
    callback();
  } finally {
    // Restore previous priority
    ReactCurrentBatchConfig.transition = prevTransition;
  }
}

// Example 11: Using transitions
function SearchComponent() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();
  
  const handleChange = (e) => {
    // High priority - update input immediately
    setQuery(e.target.value);
    
    // Low priority - update results in background
    startTransition(() => {
      const filtered = expensiveFilter(allItems, e.target.value);
      setResults(filtered);
    });
  };
  
  return (
    <div>
      <input value={query} onChange={handleChange} />
      {isPending && <Spinner />}
      <Results items={results} />
    </div>
  );
}

// Timeline:
// 1. User types 'r'
// 2. setQuery runs immediately (high priority)
// 3. Input shows 'r' instantly
// 4. Transition starts (low priority)
// 5. User types 'e' while transition rendering
// 6. setQuery preempts transition
// 7. Input shows 're' instantly
// 8. Old transition discarded, new one starts
```

## Suspense and Concurrent Rendering

```javascript
// Example 12: Suspense with concurrent features
function UserProfile({ userId }) {
  const user = use(fetchUser(userId)); // Suspends
  
  return <div>{user.name}</div>;
}

function App() {
  const [userId, setUserId] = useState(1);
  const [isPending, startTransition] = useTransition();
  
  const handleChange = (newId) => {
    startTransition(() => {
      setUserId(newId);
    });
  };
  
  return (
    <div>
      <button onClick={() => handleChange(2)}>
        Change User
      </button>
      {isPending && <Spinner />}
      <Suspense fallback={<Loading />}>
        <UserProfile userId={userId} />
      </Suspense>
    </div>
  );
}

// Behavior:
// 1. User clicks "Change User"
// 2. Transition marks update as low priority
// 3. React starts rendering new UserProfile
// 4. UserProfile suspends (data not ready)
// 5. Because it's a transition, React keeps showing old UI
// 6. isPending becomes true, shows Spinner
// 7. When data ready, transition completes
// 8. New UserProfile replaces old one
```

## useDeferredValue

```javascript
// Example 13: Deferred value internals
function useDeferredValue(value) {
  const [deferredValue, setDeferredValue] = useState(value);
  
  useEffect(() => {
    startTransition(() => {
      setDeferredValue(value);
    });
  }, [value]);
  
  return deferredValue;
}

// Example 14: Using deferred values
function SearchResults({ query }) {
  const deferredQuery = useDeferredValue(query);
  
  // Expensive calculation uses deferred value
  const results = useMemo(
    () => expensiveSearch(deferredQuery),
    [deferredQuery]
  );
  
  return (
    <div>
      {/* Show user what they typed */}
      <p>Searching for: {query}</p>
      {/* Results may lag behind */}
      {query !== deferredQuery && <Spinner />}
      <Results items={results} />
    </div>
  );
}

// How it works:
// 1. User types 'react'
// 2. query updates immediately (high priority)
// 3. "Searching for: react" shows instantly
// 4. deferredQuery updates in background (low priority)
// 5. If user keeps typing, deferred updates are interrupted
// 6. Only when user pauses do deferred updates complete
```

## Batching Updates

```javascript
// Example 15: Automatic batching in concurrent mode
function Counter() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);
  
  const handleClick = () => {
    // Before React 18: Batched only in event handlers
    // React 18: Always batched (in concurrent mode)
    setCount(c => c + 1);
    setFlag(f => !f);
    // Only one re-render, even outside event handlers
  };
  
  const handleAsync = () => {
    setTimeout(() => {
      // Before React 18: Two separate renders
      // React 18: Still batched!
      setCount(c => c + 1);
      setFlag(f => !f);
    }, 1000);
  };
  
  return (
    <div>
      <button onClick={handleClick}>Sync</button>
      <button onClick={handleAsync}>Async</button>
    </div>
  );
}

// Example 16: Opting out of batching
function UnbatchedUpdate() {
  const [count, setCount] = useState(0);
  
  const handleClick = () => {
    flushSync(() => {
      setCount(c => c + 1);
    }); // Forces immediate render
    
    // Read DOM here, guaranteed to be updated
    console.log(document.getElementById('count').textContent);
    
    flushSync(() => {
      setCount(c => c + 1);
    }); // Another immediate render
  };
  
  return <div id="count">{count}</div>;
}
```

## Scheduler Integration

```javascript
// Example 17: Task prioritization
const ImmediatePriority = 1;
const UserBlockingPriority = 2;
const NormalPriority = 3;
const LowPriority = 4;
const IdlePriority = 5;

function scheduleUpdateOnFiber(fiber, lane) {
  // Determine priority
  const priorityLevel = lanesToEventPriority(lane);
  
  // Schedule work
  if (lane === SyncLane) {
    // Synchronous update - schedule immediately
    scheduleSyncCallback(performSyncWorkOnRoot.bind(null, root));
  } else {
    // Concurrent update - schedule with priority
    scheduleCallback(
      priorityLevel,
      performConcurrentWorkOnRoot.bind(null, root)
    );
  }
}

// Example 18: Priority inheritance
function PriorityExample() {
  const [urgent, setUrgent] = useState(0);
  const [lazy, setLazy] = useState(0);
  
  const handleClick = () => {
    // High priority update
    setUrgent(u => u + 1);
    
    // During high priority render, start low priority update
    startTransition(() => {
      setLazy(l => l + 1);
    });
  };
  
  return (
    <div>
      <button onClick={handleClick}>Update</button>
      <div>Urgent: {urgent}</div>
      <Suspense fallback={<Loading />}>
        <ExpensiveComponent value={lazy} />
      </Suspense>
    </div>
  );
}

// Scheduling:
// 1. High priority update scheduled
// 2. Low priority update scheduled
// 3. Scheduler picks high priority first
// 4. High priority work completes and commits
// 5. Scheduler picks low priority next
// 6. Low priority work can be interrupted by new high priority work
```

## Expiration Times and Starvation Prevention

```javascript
// Example 19: Starvation prevention
function preventStarvation(lane) {
  const currentTime = now();
  const expirationTime = computeExpirationTime(lane, currentTime);
  
  // Low priority updates get older over time
  // Eventually they become expired and must be processed
  if (currentTime >= expirationTime) {
    // This update has expired, upgrade to sync priority
    return SyncLane;
  }
  
  return lane;
}

// Example 20: Long-running low priority work
function LongRunningUpdate() {
  const [items, setItems] = useState([]);
  
  const addManyItems = () => {
    startTransition(() => {
      // Even though this is low priority,
      // it won't be interrupted forever
      const newItems = Array.from({ length: 100000 }, (_, i) => i);
      setItems(newItems);
    });
  };
  
  // After enough time passes, React ensures this completes
  // even if high priority updates keep coming
}
```

## Common Misconceptions

1. **"Concurrent rendering makes everything faster"** - False. It makes apps more responsive by prioritizing important updates, but individual operations may take longer due to interruptions.

2. **"All updates are concurrent in React 18"** - False. Only updates marked with concurrent features (transitions, Suspense, etc.) are concurrent. Regular setState is still synchronous.

3. **"Time-slicing means splitting work across frames"** - Partially true. React yields to the browser every few milliseconds, but doesn't necessarily wait for the next frame.

4. **"Concurrent rendering breaks component lifecycles"** - Partially true. Render can be called multiple times before commit, but commit is still atomic.

5. **"startTransition makes updates async"** - Misleading. It marks updates as low priority and interruptible, not truly async.

## Performance Implications

1. **Rendering Can Happen Multiple Times** - In concurrent mode, render can be called multiple times for the same update if interrupted. Keep renders pure.

2. **Priority Inversion** - Be careful not to make too many updates low priority; important work might feel sluggish.

3. **Memory Overhead** - Concurrent rendering maintains more state (work in progress trees, priority queues) than synchronous rendering.

4. **Scheduler Overhead** - Time-slicing and priority management add CPU cost. Benefits outweigh costs in complex UIs, but simple apps might not benefit.

## Interview Questions

1. **Q: What is concurrent rendering in React?**
   A: Concurrent rendering is React's ability to interrupt rendering work, prioritize different updates, and resume work later. It enables features like time-slicing (yielding to the browser every few milliseconds), transitions (marking updates as low priority), and keeping the UI responsive during heavy rendering.

2. **Q: How does the work loop enable interruptible rendering?**
   A: The work loop checks after each unit of work whether it should yield control back to the browser using shouldYield(). If too much time has passed (typically 5ms), React pauses, lets the browser handle pending work, then resumes rendering later. This prevents long rendering tasks from blocking the main thread.

3. **Q: What are lanes and how do they relate to priority?**
   A: Lanes are React's priority system represented as bit flags. Different types of updates get different lanes: user input gets high priority lanes, transitions get low priority lanes. React uses lanes to determine which updates to work on first and which can be interrupted.

4. **Q: Explain the difference between render and commit phases in concurrent mode.**
   A: The render phase is pure and interruptible—React can pause and resume rendering. The commit phase has side effects (DOM updates, lifecycle methods) and must be atomic—it cannot be interrupted. Render can happen multiple times, but commit happens once per update.

5. **Q: How does startTransition work internally?**
   A: startTransition temporarily marks updates inside its callback as low priority by setting a transition context. These updates get assigned to transition lanes, which the scheduler knows can be interrupted by higher priority work like user input. This keeps the UI responsive during expensive updates.

6. **Q: Why doesn't React just use requestIdleCallback?**
   A: requestIdleCallback runs at too low a priority and may not run for several hundred milliseconds if the browser is busy. React needs more control over scheduling and wants to yield more frequently (every 5ms) to stay responsive while still making progress on work.

7. **Q: What prevents low priority updates from starving?**
   A: React tracks expiration times for updates. If a low priority update has been pending too long, React upgrades its priority to sync, ensuring it eventually completes even if high priority updates keep coming.

8. **Q: How does concurrent rendering affect component renders?**
   A: In concurrent mode, React may call your component function multiple times before committing. This is why renders must be pure—they may be interrupted and restarted. Effects and lifecycle methods still run once per commit, maintaining their guarantees.

## Key Takeaways

1. Concurrent rendering makes rendering interruptible through time-slicing
2. The work loop yields to the browser every 5ms to stay responsive
3. Lanes provide a priority system for different types of updates
4. Render phase is interruptible and pure; commit phase is atomic
5. startTransition marks updates as low priority and interruptible
6. High priority updates can preempt low priority work
7. Starvation prevention ensures old updates eventually complete
8. Concurrent features trade total speed for responsiveness

## Resources

- [Concurrent Rendering in React](https://react.dev/blog/2022/03/29/react-v18#what-is-concurrent-react)
- [React 18 Working Group Discussions](https://github.com/reactwg/react-18/discussions)
- [Scheduler Source Code](https://github.com/facebook/react/tree/main/packages/scheduler)
- [React Fiber Architecture](https://github.com/acdlite/react-fiber-architecture)
- [Inside React's Concurrent Rendering](https://indepth.dev/posts/1426/react-18-concurrent-rendering)
