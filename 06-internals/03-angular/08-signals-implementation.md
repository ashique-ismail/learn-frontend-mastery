# Signals Implementation: Graph, Dependencies, and Scheduling

## The Idea

**In plain English:** Signals are a way for a program to automatically keep track of which pieces of data depend on other pieces, so that when one value changes, only the parts of the app that actually use that value get updated. Think of a "signal" as a special container that holds a value and shouts out to anyone listening whenever that value changes.

**Real-world analogy:** Imagine a school's announcement board system where teachers post grade updates. When a math teacher posts a new grade, only the students enrolled in math (and the counselors tracking those students) are notified — not the entire school.

- The announcement board = the signal (holds the current value)
- A student's enrollment in a class = a dependency (a computed value or effect that reads the signal)
- The counselor who monitors multiple students = a computed signal (derives a new value from other signals)
- The intercom that broadcasts the update = the notification system that marks subscribers as dirty and schedules effects

---

## Overview

Angular signals represent a fundamental shift in reactive programming, implementing a push-pull hybrid reactive system with fine-grained reactivity. Unlike traditional observables or change detection, signals create a directed acyclic graph (DAG) of dependencies that enables precise, efficient updates with minimal overhead.

This document explores how signals work internally, including the dependency graph structure, computed signal scheduling, effect execution, and the algorithms that make signals both powerful and performant.

## Signal Fundamentals

### Core Data Structure

```typescript
// Simplified signal implementation
type SignalNode<T> = {
  // Current value
  value: T;
  
  // Version number for change detection
  version: number;
  
  // For computed signals: computation function
  computation?: () => T;
  
  // Dependencies: signals this node depends on
  dependencies: Set<SignalNode<any>>;
  
  // Subscribers: nodes that depend on this signal
  subscribers: Set<SignalNode<any>>;
  
  // Flags
  flags: {
    dirty: boolean;        // Needs recomputation
    hasError: boolean;     // Computation threw error
    isComputed: boolean;   // Is a computed signal
  };
  
  // Error state
  error?: any;
  
  // Reference to signal wrapper
  signal: Signal<T>;
};

// Global state for tracking
let currentComputation: SignalNode<any> | null = null;
let globalVersion = 0;
let effectScheduler = new EffectScheduler();
```

### Signal Creation

```typescript
// Creating a writable signal
function signal<T>(initialValue: T): WritableSignal<T> {
  const node: SignalNode<T> = {
    value: initialValue,
    version: 0,
    dependencies: new Set(),
    subscribers: new Set(),
    flags: {
      dirty: false,
      hasError: false,
      isComputed: false
    },
    signal: null as any
  };

  const signalFn = () => {
    // Reading the signal
    trackSignalRead(node);
    
    if (node.flags.hasError) {
      throw node.error;
    }
    
    return node.value;
  };

  // Add mutation methods
  signalFn.set = (newValue: T) => {
    if (!Object.is(node.value, newValue)) {
      node.value = newValue;
      node.version = ++globalVersion;
      notifySubscribers(node);
    }
  };

  signalFn.update = (updater: (current: T) => T) => {
    signalFn.set(updater(node.value));
  };

  signalFn.asReadonly = () => {
    // Return version without set/update
    return signalFn as ReadonlySignal<T>;
  };

  node.signal = signalFn as any;
  return signalFn as WritableSignal<T>;
}
```

### Dependency Tracking

```typescript
// Track when a signal is read
function trackSignalRead<T>(node: SignalNode<T>): void {
  // Only track if we're inside a computation
  if (currentComputation === null) {
    return;
  }

  // Don't track self-dependencies
  if (currentComputation === node) {
    throw new Error('Circular dependency detected');
  }

  // Register bidirectional dependency
  currentComputation.dependencies.add(node);
  node.subscribers.add(currentComputation);
}

// Notify subscribers when signal changes
function notifySubscribers<T>(node: SignalNode<T>): void {
  for (const subscriber of node.subscribers) {
    // Mark subscriber as dirty
    subscriber.flags.dirty = true;
    
    // If subscriber is an effect, schedule it
    if (subscriber.flags.isComputed === false) {
      effectScheduler.schedule(subscriber);
    }
    
    // Propagate to subscriber's subscribers
    notifySubscribers(subscriber);
  }
}
```

## Computed Signals

### Lazy Evaluation

```typescript
// Create a computed signal
function computed<T>(computation: () => T): Signal<T> {
  const node: SignalNode<T> = {
    value: undefined as any,
    version: -1, // Not yet computed
    computation,
    dependencies: new Set(),
    subscribers: new Set(),
    flags: {
      dirty: true,
      hasError: false,
      isComputed: true
    },
    signal: null as any
  };

  const signalFn = () => {
    // If dirty, recompute
    if (node.flags.dirty) {
      recomputeSignal(node);
    }
    
    // Track that we read this computed
    trackSignalRead(node);
    
    if (node.flags.hasError) {
      throw node.error;
    }
    
    return node.value;
  };

  node.signal = signalFn as any;
  return signalFn as Signal<T>;
}
```

### Recomputation Algorithm

```typescript
function recomputeSignal<T>(node: SignalNode<T>): void {
  if (!node.flags.dirty) {
    return; // Already up to date
  }

  // Check if dependencies actually changed
  if (node.version >= 0 && !dependenciesChanged(node)) {
    node.flags.dirty = false;
    return; // Dependencies unchanged, use cached value
  }

  // Clear old dependencies
  for (const dep of node.dependencies) {
    dep.subscribers.delete(node);
  }
  node.dependencies.clear();

  // Run computation and track new dependencies
  const previousComputation = currentComputation;
  currentComputation = node;

  try {
    const newValue = node.computation!();
    
    // Only update if value changed
    if (!Object.is(node.value, newValue)) {
      node.value = newValue;
      node.version = ++globalVersion;
    }
    
    node.flags.dirty = false;
    node.flags.hasError = false;
  } catch (error) {
    node.error = error;
    node.flags.hasError = true;
    node.flags.dirty = false;
  } finally {
    currentComputation = previousComputation;
  }
}

// Check if dependencies actually changed
function dependenciesChanged(node: SignalNode<any>): boolean {
  for (const dep of node.dependencies) {
    // If dependency is dirty computed, check recursively
    if (dep.flags.isComputed && dep.flags.dirty) {
      recomputeSignal(dep);
    }
    
    // Check version
    if (dep.version > node.version) {
      return true;
    }
  }
  return false;
}
```

### Example: Complex Computed Graph

```typescript
// Build a complex dependency graph
const firstName = signal('John');
const lastName = signal('Doe');
const age = signal(30);

// First level computed
const fullName = computed(() => {
  console.log('Computing fullName');
  return `${firstName()} ${lastName()}`;
});

const isAdult = computed(() => {
  console.log('Computing isAdult');
  return age() >= 18;
});

// Second level computed
const greeting = computed(() => {
  console.log('Computing greeting');
  const name = fullName();
  const adult = isAdult();
  return `Hello, ${name}! ${adult ? 'Welcome' : 'You must be 18+'}`;
});

// Third level computed
const formalGreeting = computed(() => {
  console.log('Computing formalGreeting');
  return greeting().toUpperCase();
});

// Reading triggers computation cascade
console.log(formalGreeting());
// Logs:
// Computing fullName
// Computing isAdult
// Computing greeting
// Computing formalGreeting
// Prints: HELLO, JOHN DOE! WELCOME

// Update firstName
firstName.set('Jane');
// No logs yet - computeds are lazy

// Read again - only affected computeds re-run
console.log(formalGreeting());
// Logs:
// Computing fullName
// Computing greeting
// Computing formalGreeting
// (isAdult doesn't recompute - age unchanged)
```

## Effect System

### Effect Creation

```typescript
type EffectCleanupFn = () => void;
type EffectOptions = {
  allowSignalWrites?: boolean;
  manualCleanup?: boolean;
};

function effect(
  effectFn: (onCleanup: (cleanupFn: EffectCleanupFn) => void) => void,
  options?: EffectOptions
): EffectRef {
  const node: SignalNode<void> = {
    value: undefined,
    version: 0,
    computation: undefined,
    dependencies: new Set(),
    subscribers: new Set(),
    flags: {
      dirty: true,
      hasError: false,
      isComputed: false // Effects are NOT computed
    },
    signal: null as any
  };

  let cleanupFn: EffectCleanupFn | null = null;
  let destroyed = false;

  const runEffect = () => {
    if (destroyed) return;

    // Run cleanup from previous execution
    if (cleanupFn) {
      cleanupFn();
      cleanupFn = null;
    }

    // Clear old dependencies
    for (const dep of node.dependencies) {
      dep.subscribers.delete(node);
    }
    node.dependencies.clear();

    // Run effect and track dependencies
    const previousComputation = currentComputation;
    currentComputation = node;

    try {
      effectFn((fn) => {
        cleanupFn = fn;
      });
      node.flags.dirty = false;
    } catch (error) {
      console.error('Effect error:', error);
      node.flags.hasError = true;
    } finally {
      currentComputation = previousComputation;
    }
  };

  // Run immediately
  runEffect();

  // Return cleanup handle
  return {
    destroy() {
      if (destroyed) return;
      destroyed = true;
      
      if (cleanupFn) {
        cleanupFn();
      }
      
      // Remove from all dependencies
      for (const dep of node.dependencies) {
        dep.subscribers.delete(node);
      }
      node.dependencies.clear();
    }
  };
}
```

### Effect Scheduling

```typescript
class EffectScheduler {
  private queue: Set<SignalNode<any>> = new Set();
  private scheduled = false;

  schedule(node: SignalNode<any>): void {
    this.queue.add(node);
    
    if (!this.scheduled) {
      this.scheduled = true;
      queueMicrotask(() => this.flush());
    }
  }

  private flush(): void {
    this.scheduled = false;
    
    // Copy queue and clear
    const effects = Array.from(this.queue);
    this.queue.clear();

    // Run all effects
    for (const node of effects) {
      if (node.flags.dirty && node.computation) {
        const previousComputation = currentComputation;
        currentComputation = node;
        
        try {
          node.computation();
          node.flags.dirty = false;
        } catch (error) {
          console.error('Effect error:', error);
        } finally {
          currentComputation = previousComputation;
        }
      }
    }
  }
}
```

### Effect Patterns

```typescript
// Effect with cleanup
@Component({})
export class TimerComponent {
  private count = signal(0);

  constructor() {
    effect((onCleanup) => {
      const timer = setInterval(() => {
        this.count.update(c => c + 1);
      }, 1000);

      onCleanup(() => {
        console.log('Cleaning up timer');
        clearInterval(timer);
      });
    });
  }
}

// Effect with dependency tracking
@Component({})
export class LoggerComponent {
  private userId = signal<string>('');
  private userData = signal<User | null>(null);

  constructor() {
    // Effect re-runs when userId changes
    effect(() => {
      const id = this.userId();
      
      if (id) {
        console.log('Fetching user:', id);
        fetch(`/api/users/${id}`)
          .then(r => r.json())
          .then(user => this.userData.set(user));
      }
    });

    // Separate effect for logging user data
    effect(() => {
      const user = this.userData();
      if (user) {
        console.log('User loaded:', user.name);
      }
    });
  }
}

// Effect with untracked reads
@Component({})
export class AnalyticsComponent {
  private pageViews = signal(0);
  private sessionId = signal('');

  constructor() {
    effect(() => {
      const views = this.pageViews();
      
      // Read sessionId without creating dependency
      const session = untracked(() => this.sessionId());
      
      // This effect only re-runs when pageViews changes
      // not when sessionId changes
      this.logAnalytics({ views, session });
    });
  }

  logAnalytics(data: any) {
    console.log('Analytics:', data);
  }
}
```

## Advanced Features

### Untracked Reads

```typescript
function untracked<T>(fn: () => T): T {
  const previousComputation = currentComputation;
  currentComputation = null;
  
  try {
    return fn();
  } finally {
    currentComputation = previousComputation;
  }
}

// Usage example
const example = computed(() => {
  const a = signalA(); // Tracked
  const b = untracked(() => signalB()); // Not tracked
  return a + b;
});
// This computed only re-runs when signalA changes
```

### Equal Comparator

```typescript
// Signal with custom equality
function signal<T>(
  initialValue: T,
  options?: { equal?: (a: T, b: T) => boolean }
): WritableSignal<T> {
  const equal = options?.equal ?? Object.is;
  
  // ... node creation ...
  
  signalFn.set = (newValue: T) => {
    if (!equal(node.value, newValue)) {
      node.value = newValue;
      node.version = ++globalVersion;
      notifySubscribers(node);
    }
  };
  
  return signalFn;
}

// Example: Deep equality for objects
const deepEqual = (a: any, b: any) => JSON.stringify(a) === JSON.stringify(b);

const user = signal(
  { name: 'John', age: 30 },
  { equal: deepEqual }
);

user.set({ name: 'John', age: 30 }); // No update - equal
user.set({ name: 'Jane', age: 30 }); // Updates - not equal
```

### Memoization

```typescript
// Computed signals are automatically memoized
const expensive = computed(() => {
  console.log('Expensive computation');
  let result = 0;
  for (let i = 0; i < 1000000; i++) {
    result += i;
  }
  return result;
});

// First read - runs computation
console.log(expensive()); // Logs "Expensive computation"

// Second read - returns cached value
console.log(expensive()); // No log - uses cache

// Dependencies change - marks dirty
dependency.set(newValue);

// Next read - recomputes
console.log(expensive()); // Logs "Expensive computation" again
```

## Dependency Graph Visualization

### Graph Structure

```typescript
// Example dependency graph
const a = signal(1);
const b = signal(2);
const c = computed(() => a() + b());
const d = computed(() => c() * 2);
const e = computed(() => a() + d());

// Graph structure:
//     a -----> c -----> d -----> e
//     |                 ^
//     |                 |
//     +------------------+
//     b

// Update 'a' affects: c, d, e
// Update 'b' affects: c, d, e
// Update 'c' affects: d, e
// Update 'd' affects: e
```

### Traversal Algorithm

```typescript
// Get all nodes affected by a change (for debugging)
function getAffectedNodes(node: SignalNode<any>): Set<SignalNode<any>> {
  const affected = new Set<SignalNode<any>>();
  const visited = new Set<SignalNode<any>>();

  function traverse(current: SignalNode<any>) {
    if (visited.has(current)) return;
    visited.add(current);
    affected.add(current);

    for (const subscriber of current.subscribers) {
      traverse(subscriber);
    }
  }

  traverse(node);
  return affected;
}

// Example usage
const firstName = signal('John');
const lastName = signal('Doe');
const fullName = computed(() => `${firstName()} ${lastName()}`);
const greeting = computed(() => `Hello, ${fullName()}`);

// Get all nodes affected by firstName change
const affected = getAffectedNodes(getNode(firstName));
// Returns: [firstName, fullName, greeting]
```

### Cycle Detection

```typescript
function detectCycle(node: SignalNode<any>): boolean {
  const visiting = new Set<SignalNode<any>>();
  const visited = new Set<SignalNode<any>>();

  function hasCycle(current: SignalNode<any>): boolean {
    if (visited.has(current)) return false;
    if (visiting.has(current)) return true; // Cycle detected!

    visiting.add(current);

    for (const dep of current.dependencies) {
      if (hasCycle(dep)) return true;
    }

    visiting.delete(current);
    visited.add(current);
    return false;
  }

  return hasCycle(node);
}

// This would throw during creation
try {
  const a = signal(1);
  const b = computed(() => c() + 1); // References c before defined
  const c = computed(() => b() + 1); // Cycle: b -> c -> b
} catch (error) {
  console.error('Circular dependency detected');
}
```

## Performance Optimizations

### Batch Updates

```typescript
// Batch multiple signal updates
function batch(fn: () => void): void {
  const prevGlobalVersion = globalVersion;
  const updatedNodes = new Set<SignalNode<any>>();

  // Temporarily intercept notifications
  const originalNotify = notifySubscribers;
  notifySubscribers = (node) => {
    updatedNodes.add(node);
  };

  try {
    fn();
  } finally {
    // Restore original notify
    notifySubscribers = originalNotify;

    // Notify all at once
    for (const node of updatedNodes) {
      originalNotify(node);
    }
  }
}

// Usage
batch(() => {
  firstName.set('Jane');
  lastName.set('Smith');
  age.set(25);
  // All three updates batched into one notification
});
```

### Weak References

```typescript
// Use WeakMap for node metadata to allow GC
const nodeMetadata = new WeakMap<SignalNode<any>, {
  debugName?: string;
  createdAt: number;
  computeCount: number;
}>();

function createSignalWithMetadata<T>(value: T, debugName?: string) {
  const sig = signal(value);
  const node = getNode(sig);
  
  nodeMetadata.set(node, {
    debugName,
    createdAt: Date.now(),
    computeCount: 0
  });
  
  return sig;
}
```

### Selective Notification

```typescript
// Only notify if value actually changed after recomputation
function smartNotify<T>(node: SignalNode<T>): void {
  const oldVersion = node.version;
  
  // If computed, recompute to check if value changed
  if (node.flags.isComputed && node.flags.dirty) {
    const oldValue = node.value;
    recomputeSignal(node);
    
    // Value unchanged? Don't notify subscribers
    if (Object.is(oldValue, node.value)) {
      return;
    }
  }
  
  // Value changed, notify subscribers
  notifySubscribers(node);
}
```

## Real-World Implementation

### Store Pattern

```typescript
// Type-safe signal store
interface StoreState {
  users: User[];
  loading: boolean;
  error: string | null;
}

class SignalStore<T extends object> {
  private state: { [K in keyof T]: WritableSignal<T[K]> };
  
  constructor(initialState: T) {
    this.state = {} as any;
    
    for (const key in initialState) {
      this.state[key] = signal(initialState[key]);
    }
  }

  select<K extends keyof T>(key: K): Signal<T[K]> {
    return this.state[key].asReadonly();
  }

  update<K extends keyof T>(key: K, value: T[K]): void {
    this.state[key].set(value);
  }

  patch(partial: Partial<T>): void {
    batch(() => {
      for (const key in partial) {
        if (key in this.state) {
          this.state[key].set(partial[key]!);
        }
      }
    });
  }
}

// Usage
const store = new SignalStore<StoreState>({
  users: [],
  loading: false,
  error: null
});

const users = store.select('users');
const loading = store.select('loading');

// Update multiple properties atomically
store.patch({
  users: newUsers,
  loading: false
});
```

### Async Signal

```typescript
// Signal that loads data asynchronously
function asyncSignal<T>(
  fetcher: () => Promise<T>,
  initialValue: T
): Signal<T> & { reload: () => void } {
  const data = signal<T>(initialValue);
  const loading = signal(false);
  const error = signal<Error | null>(null);

  const load = async () => {
    loading.set(true);
    error.set(null);

    try {
      const result = await fetcher();
      data.set(result);
    } catch (err) {
      error.set(err as Error);
    } finally {
      loading.set(false);
    }
  };

  // Initial load
  load();

  const sig = computed(() => data()) as any;
  sig.reload = load;
  sig.loading = loading.asReadonly();
  sig.error = error.asReadonly();

  return sig;
}

// Usage
const userData = asyncSignal(
  () => fetch('/api/user').then(r => r.json()),
  null
);

// In template
console.log(userData()); // Current data
console.log(userData.loading()); // Loading state
userData.reload(); // Reload data
```

## Common Misconceptions

### "Computed signals run eagerly"

**Reality**: Computed signals are lazy and only run when read:

```typescript
const expensive = computed(() => {
  console.log('Computing...');
  return heavyCalculation();
});

// No log yet - not computed

const value = expensive(); // NOW it logs and computes
```

### "Effects run synchronously"

**Reality**: Effects are scheduled and run asynchronously in microtasks:

```typescript
const count = signal(0);
effect(() => console.log(count()));

count.set(1);
console.log('After set');

// Output:
// 0 (initial effect run)
// "After set"
// 1 (effect runs in microtask)
```

### "Signals create memory leaks"

**Reality**: Signals properly cleanup dependencies. Only effects need manual cleanup:

```typescript
// Computed - no cleanup needed
const computed1 = computed(() => signal1());
// Automatically cleaned up when no longer referenced

// Effect - cleanup recommended
const effectRef = effect(() => {
  // Do something
});

// Clean up when done
effectRef.destroy();
```

## Performance Implications

### Benefits

1. **Fine-grained updates**: Only affected nodes recompute
2. **Lazy evaluation**: Computeds only run when needed
3. **Automatic memoization**: Cached until dependencies change
4. **Minimal overhead**: Simple object graph, no complex diffing
5. **Batch-friendly**: Multiple updates batch automatically

### Benchmarks

```typescript
// 1000 signals, 500 computed, 100 effects
// Update 1 signal: ~0.1ms
// Traditional change detection: ~15ms (checks all 1600 nodes)
// Signal update: ~0.5ms (updates only 3 affected nodes)

// Memory usage
// Traditional CD: ~2KB per component
// Signals: ~0.3KB per signal
```

## Interview Questions

### Q1: How do signals track dependencies internally?

**Answer**: Signals use a global tracking context. When a signal is read (called as a function), it checks if there's a current computation running (stored in `currentComputation`). If so, it registers a bidirectional dependency: the signal adds the computation to its subscribers set, and the computation adds the signal to its dependencies set. This creates a dependency graph. When the signal changes, it walks its subscribers set and marks them dirty. Computed signals track dependencies by setting themselves as the current computation before running their function, then restoring the previous computation after. This automatic tracking works recursively for nested reads.

### Q2: What's the difference between computed signals and effects?

**Answer**: Both track dependencies, but they serve different purposes:

**Computed signals**:
- Produce values (return something)
- Lazy - only run when read
- Memoized - cache results
- Pure - should not have side effects
- Can be read in templates
- Example: `computed(() => firstName() + lastName())`

**Effects**:
- Run side effects (logging, API calls, etc.)
- Eager - run immediately and on dependency changes
- Not memoized - run every time dependencies change
- Impure - designed for side effects
- Cannot be read (return void)
- Example: `effect(() => console.log(count()))`

Use computed for derived state, effects for reactions to state changes.

### Q3: How does signal batching work?

**Answer**: Signal updates are automatically batched within a synchronous execution context. When multiple signals update, Angular queues all notifications and flushes them together in a single microtask. For example:
```typescript
count.set(1);
count.set(2);
count.set(3);
```
These three updates are batched - only the final value (3) triggers effects/computed recomputation. The batching happens because notifications are scheduled as microtasks, not executed immediately. You can also manually batch using Angular's `batch()` function to group updates across async boundaries. This prevents unnecessary intermediate computations and renders, significantly improving performance when making multiple related changes.

### Q4: How do signals prevent circular dependencies?

**Answer**: Signals detect cycles during dependency tracking. When registering a dependency, if the signal being read is the same as the current computation, a circular dependency is detected and an error is thrown. For example:
```typescript
const a = computed(() => b());
const b = computed(() => a()); // Error!
```
When `a` computes, it sets itself as `currentComputation`, then reads `b`. When `b` computes, it tries to read `a`, but `a` is already the current computation - cycle detected. The algorithm maintains a "visiting" set during dependency traversal to detect cycles in more complex scenarios. This prevents infinite loops and helps developers identify architectural issues early.

### Q5: Why are computed signals lazy?

**Answer**: Lazy evaluation provides significant performance benefits:
1. **Avoid unnecessary work**: If a computed is never read, it never runs
2. **Cascade optimization**: If a computed's result won't change (dependencies unchanged), downstream computeds don't recompute
3. **Memory efficiency**: Don't store values that are never used
4. **Better tree-shaking**: Unused computeds can be eliminated

For example:
```typescript
const expensive = computed(() => heavyCalculation());
if (condition) {
  console.log(expensive()); // Only runs if condition is true
}
```
The computation only runs when needed, and results are cached for subsequent reads. This is more efficient than eager evaluation which would compute even if the value is never used.

### Q6: How do effects cleanup work?

**Answer**: Effects provide a cleanup mechanism via the `onCleanup` callback:
```typescript
effect((onCleanup) => {
  const timer = setInterval(() => {}, 1000);
  onCleanup(() => clearInterval(timer));
});
```

Cleanup runs in two scenarios:
1. **Before re-running**: When dependencies change and the effect re-runs, the previous cleanup executes first
2. **On destroy**: When calling `effectRef.destroy()`

Internally, the effect stores the cleanup function and invokes it before the next execution. This prevents resource leaks (timers, subscriptions, event listeners). Effects also automatically remove themselves from their dependencies' subscriber sets on cleanup, breaking the reactivity chain. Always provide cleanup for effects that create resources.

### Q7: What happens when you update a signal inside a computed?

**Answer**: By default, Angular prevents signal writes inside computed to maintain purity:
```typescript
const bad = computed(() => {
  const value = input();
  output.set(value); // ERROR: Writing to signals not allowed in computed
  return value;
});
```

This restriction ensures computeds remain pure functions without side effects. If you need to perform side effects based on signal changes, use an effect instead:
```typescript
effect(() => {
  const value = input();
  output.set(value); // OK in effect
});
```

Effects allow signal writes by default (though it can be disabled with `allowSignalWrites: false`). This separation keeps the reactivity model predictable: computeds derive values, effects perform actions.

### Q8: How does signal performance compare to traditional change detection?

**Answer**: Signals provide significant performance improvements:

**Traditional change detection**:
- Checks entire component tree
- Runs on every async operation (Zone.js)
- O(n) complexity where n = number of components
- Can't skip subtrees without OnPush

**Signal-based**:
- Only updates affected nodes
- Only runs on actual data changes
- O(m) complexity where m = number of affected signals (typically << n)
- Automatic granular updates

Benchmarks show 3-10x faster updates in large apps. For example, updating 1 signal in a 1000-component app: traditional CD checks all 1000 components (~15ms), signals update only 3 affected nodes (~0.5ms). Signals also enable better bundle size (no Zone.js) and more predictable performance (no hidden CD cycles).

## Key Takeaways

1. Signals create a directed acyclic graph (DAG) of dependencies for precise updates
2. Dependency tracking is automatic via global computation context
3. Computed signals are lazy, memoized, and recompute only when dependencies change
4. Effects run eagerly for side effects and schedule execution in microtasks
5. Batching happens automatically - multiple updates trigger one notification cycle
6. Circular dependencies are detected and throw errors
7. Cleanup is important for effects but automatic for computeds
8. Performance is significantly better than traditional change detection for large apps

## Resources

- [Angular Signals RFC](https://github.com/angular/angular/discussions/49685)
- [Signal Implementation Source Code](https://github.com/angular/angular/tree/main/packages/core/src/signals)
- [Reactivity with Signals](https://angular.dev/guide/signals)
- [Solid.js Reactivity (Similar Model)](https://www.solidjs.com/docs/latest/api#createsignal)
- [Fine-Grained Reactivity Explained](https://dev.to/ryansolid/fine-grained-reactivity-explained)
- [Angular Signals Deep Dive - YouTube](https://www.youtube.com/watch?v=signals-deepdive)
- [Effect Scheduling Algorithm](https://github.com/angular/angular/blob/main/packages/core/src/render3/reactivity/effect.ts)
