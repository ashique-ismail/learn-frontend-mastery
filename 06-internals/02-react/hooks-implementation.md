# Hooks Implementation

## Overview

React Hooks fundamentally changed how we write components, but underneath the intuitive API lies a sophisticated implementation based on linked lists, dispatcher patterns, and fiber architecture. Understanding how hooks work internally reveals why the Rules of Hooks exist, how useState and useEffect actually function, and what performance characteristics different patterns have.

This deep dive explores the internal mechanics of hooks, from the dispatcher pattern to the hooks linked list, memory models, and execution timing.

## The Hooks Linked List

At the core of hooks implementation is a linked list structure. Each hook call creates a node in this list, and React uses the position in the list to match hooks between renders.

```javascript
// Example 1: Internal hook structure (simplified)
type Hook = {
  memoizedState: any;      // The hook's state
  baseState: any;          // Base state for updates
  queue: UpdateQueue;      // Pending updates
  baseQueue: UpdateQueue;  // Base queue for updates
  next: Hook | null;       // Next hook in the list
};

type Fiber = {
  memoizedState: Hook | null;  // First hook in the list
  // ... other fiber properties
};

// Example 2: How hooks are stored
function Component() {
  const [count, setCount] = useState(0);     // Hook #0
  const [name, setName] = useState('');      // Hook #1
  const value = useMemo(() => count * 2, [count]); // Hook #2
  
  // Internally creates linked list:
  // Hook #0: { memoizedState: 0, next: Hook #1 }
  // Hook #1: { memoizedState: '', next: Hook #2 }
  // Hook #2: { memoizedState: 0, next: null }
}
```

This is why the order of hooks must be consistent:

```javascript
// Example 3: Why conditional hooks break
function BrokenComponent({ showExtra }) {
  const [count, setCount] = useState(0);  // Hook #0
  
  if (showExtra) {
    // WRONG: Hook #1 only exists sometimes
    const [extra, setExtra] = useState('');
  }
  
  const [name, setName] = useState('');  // Hook #1 or #2 depending on showExtra
  
  // On first render (showExtra = true):
  // Hook #0: count
  // Hook #1: extra
  // Hook #2: name
  
  // On second render (showExtra = false):
  // Hook #0: count
  // Hook #1: name (but React expects this to be 'extra'!)
  
  // React tries to restore 'extra' into name state → BUG
}

// Example 4: Correct conditional logic
function CorrectComponent({ showExtra }) {
  const [count, setCount] = useState(0);
  const [extra, setExtra] = useState('');  // Always present
  const [name, setName] = useState('');
  
  // Use the value conditionally, not the hook
  return (
    <div>
      <p>Count: {count}</p>
      {showExtra && <p>Extra: {extra}</p>}
      <p>Name: {name}</p>
    </div>
  );
}
```

## The Dispatcher Pattern

React uses a dispatcher to swap hook implementations between mount and update phases:

```javascript
// Example 5: Dispatcher pattern (simplified)
const ReactCurrentDispatcher = {
  current: null  // Points to current dispatcher
};

// Mount phase dispatcher
const HooksDispatcherOnMount = {
  useState: mountState,
  useEffect: mountEffect,
  useMemo: mountMemo,
  // ...
};

// Update phase dispatcher
const HooksDispatcherOnUpdate = {
  useState: updateState,
  useEffect: updateEffect,
  useMemo: updateMemo,
  // ...
};

// The public API
function useState(initialState) {
  const dispatcher = ReactCurrentDispatcher.current;
  return dispatcher.useState(initialState);
}

// Example 6: Render cycle dispatcher switching
function renderWithHooks(workInProgress, Component, props) {
  // Set appropriate dispatcher
  if (workInProgress.alternate === null) {
    // First render - mount
    ReactCurrentDispatcher.current = HooksDispatcherOnMount;
  } else {
    // Subsequent render - update
    ReactCurrentDispatcher.current = HooksDispatcherOnUpdate;
  }
  
  // Call component
  const children = Component(props);
  
  // Clear dispatcher
  ReactCurrentDispatcher.current = null;
  
  return children;
}
```

This is why hooks can only be called during render:

```javascript
// Example 7: Why hooks can't be called outside render
function BadComponent() {
  const [count, setCount] = useState(0);
  
  // WRONG: Called outside render
  setTimeout(() => {
    const [delayed] = useState(100);  // Error!
    // ReactCurrentDispatcher.current is null here
  }, 1000);
  
  return <div>{count}</div>;
}
```

## useState Implementation

useState is simpler than it appears:

```javascript
// Example 8: Simplified useState mount implementation
function mountState(initialState) {
  // Create hook node
  const hook = mountWorkInProgressHook();
  
  // Resolve initial state
  if (typeof initialState === 'function') {
    initialState = initialState();
  }
  
  hook.memoizedState = initialState;
  hook.baseState = initialState;
  
  // Create update queue
  const queue = {
    pending: null,
    dispatch: null,
  };
  hook.queue = queue;
  
  // Create dispatch function
  const dispatch = dispatchSetState.bind(
    null,
    currentlyRenderingFiber,
    queue
  );
  queue.dispatch = dispatch;
  
  return [hook.memoizedState, dispatch];
}

// Example 9: Simplified useState update implementation
function updateState(initialState) {
  return updateReducer(basicStateReducer);
}

function basicStateReducer(state, action) {
  return typeof action === 'function' ? action(state) : action;
}

// Example 10: How setState works internally
function dispatchSetState(fiber, queue, action) {
  // Create update object
  const update = {
    action,
    next: null,
  };
  
  // Add to queue
  const pending = queue.pending;
  if (pending === null) {
    // First update - create circular list
    update.next = update;
  } else {
    // Add to circular list
    update.next = pending.next;
    pending.next = update;
  }
  queue.pending = update;
  
  // Schedule render
  scheduleUpdateOnFiber(fiber);
}
```

### State Updates are Queued

```javascript
// Example 11: Multiple setState calls are batched
function Counter() {
  const [count, setCount] = useState(0);
  
  const handleClick = () => {
    setCount(count + 1);  // Schedules update, doesn't execute immediately
    setCount(count + 1);  // Another update scheduled
    setCount(count + 1);  // Another update scheduled
    
    // All three updates batched, executed later
    // Each uses the same 'count' value (stale closure)
    // Result: count increases by 1, not 3
  };
  
  return <button onClick={handleClick}>{count}</button>;
}

// Example 12: Functional updates solve this
function BetterCounter() {
  const [count, setCount] = useState(0);
  
  const handleClick = () => {
    setCount(c => c + 1);  // Uses previous state
    setCount(c => c + 1);  // Uses result of previous update
    setCount(c => c + 1);  // Uses result of previous update
    
    // Updates applied sequentially
    // Result: count increases by 3
  };
  
  return <button onClick={handleClick}>{count}</button>;
}

// Example 13: How update queue is processed
function processUpdateQueue(workInProgress, queue) {
  let newState = queue.baseState;
  let update = queue.pending;
  
  if (update !== null) {
    // Circular list - break the circle
    const first = update.next;
    let current = first;
    
    do {
      const action = current.action;
      // Apply update
      newState = typeof action === 'function'
        ? action(newState)
        : action;
      current = current.next;
    } while (current !== first);
    
    // Clear queue
    queue.pending = null;
  }
  
  return newState;
}
```

## useEffect Implementation

useEffect has more complexity due to its lifecycle nature:

```javascript
// Example 14: Simplified useEffect mount implementation
function mountEffect(create, deps) {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  
  // Mark fiber for effect
  currentlyRenderingFiber.flags |= PassiveEffect;
  
  hook.memoizedState = pushEffect(
    HookPassive | HookHasEffect,
    create,
    undefined,  // destroy function (from cleanup)
    nextDeps
  );
}

// Example 15: Effect linked list structure
type Effect = {
  tag: HookEffectTag;
  create: () => (() => void) | void;  // Effect function
  destroy: (() => void) | void;       // Cleanup function
  deps: Array<any> | null;            // Dependencies
  next: Effect;                       // Next effect (circular list)
};

// Example 16: How effects are stored on fiber
function Component() {
  useEffect(() => {
    console.log('Effect 1');
    return () => console.log('Cleanup 1');
  }, []);
  
  useEffect(() => {
    console.log('Effect 2');
  }, [count]);
  
  // Creates circular effect list on fiber:
  // Effect1 <-> Effect2 <-> Effect1 (circular)
}

// Example 17: Simplified useEffect update implementation
function updateEffect(create, deps) {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  let destroy = undefined;
  
  if (currentHook !== null) {
    const prevEffect = currentHook.memoizedState;
    destroy = prevEffect.destroy;
    
    if (nextDeps !== null) {
      const prevDeps = prevEffect.deps;
      // Compare dependencies
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        // Dependencies unchanged - skip effect
        hook.memoizedState = pushEffect(
          HookPassive,  // No HookHasEffect flag
          create,
          destroy,
          nextDeps
        );
        return;
      }
    }
  }
  
  // Dependencies changed - run effect
  currentlyRenderingFiber.flags |= PassiveEffect;
  hook.memoizedState = pushEffect(
    HookPassive | HookHasEffect,
    create,
    destroy,
    nextDeps
  );
}

// Example 18: Dependency comparison
function areHookInputsEqual(nextDeps, prevDeps) {
  if (prevDeps === null) return false;
  
  for (let i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
    if (Object.is(nextDeps[i], prevDeps[i])) {
      continue;
    }
    return false;
  }
  return true;
}
```

### Effect Execution Timing

```javascript
// Example 19: When effects actually run
function commitRoot(root) {
  // 1. Render phase completes
  // 2. Commit phase begins
  
  // Before mutation - run getSnapshotBeforeUpdate
  commitBeforeMutationEffects(root);
  
  // Mutation - update DOM
  commitMutationEffects(root);
  
  // Layout effects - useLayoutEffect runs synchronously
  commitLayoutEffects(root);
  
  // Schedule passive effects - useEffect runs asynchronously
  scheduleCallback(NormalPriority, () => {
    flushPassiveEffects();
  });
  
  // 3. Browser paints
  // 4. useEffect callbacks run
}

// Example 20: Effect cleanup timing
function flushPassiveEffects() {
  // Phase 1: Run all cleanups first
  for (const fiber of fibersWithPassiveEffects) {
    const effect = fiber.updateQueue;
    if (effect !== null) {
      const destroy = effect.destroy;
      if (destroy !== undefined) {
        effect.destroy = undefined;
        destroy();  // Run cleanup
      }
    }
  }
  
  // Phase 2: Run all effect callbacks
  for (const fiber of fibersWithPassiveEffects) {
    const effect = fiber.updateQueue;
    if (effect !== null) {
      const create = effect.create;
      effect.destroy = create();  // Run effect, store cleanup
    }
  }
}
```

## useMemo and useCallback Implementation

These hooks are simpler - they just cache values:

```javascript
// Example 21: Simplified useMemo implementation
function mountMemo(nextCreate, deps) {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  
  // Compute value
  const nextValue = nextCreate();
  
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}

function updateMemo(nextCreate, deps) {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;
  
  if (prevState !== null && nextDeps !== null) {
    const prevDeps = prevState[1];
    if (areHookInputsEqual(nextDeps, prevDeps)) {
      // Dependencies unchanged - return cached value
      return prevState[0];
    }
  }
  
  // Dependencies changed - recompute
  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}

// Example 22: useCallback is just useMemo for functions
function mountCallback(callback, deps) {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  hook.memoizedState = [callback, nextDeps];
  return callback;
}

function updateCallback(callback, deps) {
  // Equivalent to: useMemo(() => callback, deps)
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;
  
  if (prevState !== null && nextDeps !== null) {
    const prevDeps = prevState[1];
    if (areHookInputsEqual(nextDeps, prevDeps)) {
      return prevState[0];  // Return previous callback
    }
  }
  
  hook.memoizedState = [callback, nextDeps];
  return callback;  // Return new callback
}
```

## useRef Implementation

useRef is the simplest hook:

```javascript
// Example 23: Simplified useRef implementation
function mountRef(initialValue) {
  const hook = mountWorkInProgressHook();
  const ref = { current: initialValue };
  hook.memoizedState = ref;
  return ref;
}

function updateRef(initialValue) {
  const hook = updateWorkInProgressHook();
  // Always return the same object
  return hook.memoizedState;
}

// Example 24: Why ref.current doesn't trigger re-render
function Component() {
  const countRef = useRef(0);
  
  const increment = () => {
    countRef.current += 1;
    // No re-render scheduled
    // React doesn't track mutations to ref.current
  };
  
  // The ref object itself never changes identity
  // Only its .current property changes
}
```

## Custom Hooks

Custom hooks are just functions that call other hooks:

```javascript
// Example 25: Custom hook implementation
function useLocalStorage(key, initialValue) {
  // Each hook call creates nodes in the linked list
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      return initialValue;
    }
  });
  
  const setValue = useCallback((value) => {
    try {
      setStoredValue(value);
      window.localStorage.setItem(key, JSON.stringify(value));
    } catch (error) {
      console.error(error);
    }
  }, [key]);
  
  return [storedValue, setValue];
}

// When used:
function Component() {
  const [name, setName] = useLocalStorage('name', '');
  
  // Internally creates hook nodes:
  // Hook #0: useState (from useLocalStorage)
  // Hook #1: useCallback (from useLocalStorage)
  // Both are part of the same linked list
}
```

## Memory Management

```javascript
// Example 26: Hook cleanup and memory
function Component() {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    let cancelled = false;
    
    fetchData().then(result => {
      if (!cancelled) {
        setData(result);
      }
    });
    
    // Cleanup prevents memory leak
    return () => {
      cancelled = true;
    };
  }, []);
  
  // When component unmounts:
  // 1. Cleanup function runs
  // 2. Effect list is cleared
  // 3. Hook linked list is garbage collected
  // 4. Fiber is removed from work tree
}

// Example 27: Circular references in effects
function ProblematicComponent() {
  const [data, setData] = useState({});
  
  useEffect(() => {
    const subscription = subscribe(event => {
      // Closure captures setData
      setData(prevData => ({ ...prevData, event }));
    });
    
    // If we forget cleanup:
    // return () => subscription.unsubscribe();
    
    // subscription holds reference to callback
    // callback holds reference to setData
    // setData is bound to fiber
    // fiber cannot be garbage collected → memory leak
  }, []);
}
```

## Common Misconceptions

1. **"Hooks are magic"** - False. They're just functions that access a linked list on the current fiber using a dispatcher pattern.

2. **"setState is synchronous"** - False. setState schedules an update that executes later. React batches multiple setState calls.

3. **"useEffect runs after render"** - Partially true. It runs after the commit phase and after the browser paints, making it asynchronous.

4. **"useCallback prevents all re-renders"** - False. It only prevents passing a new function reference, which only matters if children use React.memo or similar optimizations.

5. **"Hooks have significant overhead"** - Mostly false. The linked list and dispatcher pattern are quite efficient. The real cost is in how you use hooks, not the hooks themselves.

## Performance Implications

1. **Hook Call Order** - Must be consistent because React relies on call order to match hooks between renders.

2. **Dependency Arrays** - Incorrect dependencies cause bugs; missing dependencies cause stale closures; excessive dependencies cause unnecessary re-execution.

3. **Effect Cleanup** - Always clean up subscriptions, timers, and async operations to prevent memory leaks.

4. **Memoization Overhead** - useMemo and useCallback have a cost. Only use them when the computation or referential equality actually matters.

## Interview Questions

1. **Q: How does React know which hook is which between renders?**
   A: React maintains a linked list of hooks on each fiber. Hooks are matched by their position in the list, which is determined by call order. This is why hooks must be called in the same order every render.

2. **Q: What is the dispatcher pattern in hooks implementation?**
   A: React uses different dispatcher objects for mount vs update phases. Each dispatcher has different implementations of hooks (mountState vs updateState). The public useState API calls dispatcher.useState, which resolves to the appropriate implementation.

3. **Q: Explain how useState batching works.**
   A: setState doesn't update state immediately. It creates an update object and adds it to a circular linked list (update queue). React processes this queue during the next render, applying updates sequentially. Multiple setState calls in the same event handler are batched into one render.

4. **Q: How does useEffect differ from useLayoutEffect internally?**
   A: Both create effect nodes in a linked list, but useLayoutEffect has the HookLayout tag while useEffect has HookPassive. Layout effects run synchronously during the commit phase before browser paint. Passive effects are scheduled to run asynchronously after paint.

5. **Q: Why can't you call hooks conditionally?**
   A: Hooks are stored in a linked list matched by position. Conditional hooks would change the list structure between renders, causing React to mismatch hooks with their previous state (e.g., treating hook #1's state as hook #2's state).

6. **Q: How does dependency comparison work in useEffect?**
   A: React uses Object.is() to compare each dependency in the array with its previous value. If all dependencies are equal, the effect is skipped. This is a shallow comparison—objects are compared by reference, not deep value.

7. **Q: What happens to hooks when a component unmounts?**
   A: React runs all effect cleanups, then garbage collects the fiber including its hooks linked list. Any closures referencing the fiber (like pending setState calls) become no-ops if they execute after unmount.

8. **Q: Explain how useMemo avoids unnecessary recalculations.**
   A: useMemo stores [computedValue, dependencies] in the hook's memoizedState. On updates, it compares new dependencies with previous ones. If they're equal, it returns the cached value without calling the computation function.

## Key Takeaways

1. Hooks are implemented as a linked list on each fiber node
2. React uses a dispatcher pattern to swap mount and update implementations
3. Hook order must be consistent because position determines identity
4. useState queues updates in a circular linked list, processed during render
5. useEffect creates effect nodes with create/destroy functions and dependencies
6. Dependency comparison uses Object.is() for shallow equality
7. Effect cleanups run before new effects and on unmount
8. Understanding hooks implementation explains the Rules of Hooks

## Resources

- [React Hooks Source Code](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.js)
- [Building Your Own Hooks](https://react.dev/learn/reusing-logic-with-custom-hooks)
- [Deep Dive into React Hooks](https://medium.com/@ryardley/react-hooks-not-magic-just-arrays-cd4f1857236e)
- [Under the Hood: React](https://indepth.dev/posts/1008/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react)
- [React Fiber Architecture](https://github.com/acdlite/react-fiber-architecture)
