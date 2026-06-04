# Glitch-Free Propagation in Angular Signals

## Table of Contents
- [Overview](#overview)
- [What Is a Glitch](#what-is-a-glitch)
- [The Problem with Push-Only Systems](#the-problem-with-push-only-systems)
- [Push-Pull DAG: The Solution](#push-pull-dag-the-solution)
- [Angular Signals Internals](#angular-signals-internals)
- [The Mark-Then-Pull Algorithm](#the-mark-then-pull-algorithm)
- [Computed Signals and Memoization](#computed-signals-and-memoization)
- [Effects and Scheduling](#effects-and-scheduling)
- [Signal Graph Traversal](#signal-graph-traversal)
- [Comparison with Other Reactive Systems](#comparison-with-other-reactive-systems)
- [Common Pitfalls](#common-pitfalls)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Overview

"Glitch-free" is a property of reactive systems describing the guarantee that observers never see inconsistent intermediate states when multiple signals change together. In a glitchy system, a computation that depends on two signals A and B might temporarily see the new value of A with the old value of B — a "glitch" — before both have settled to their final values.

Angular's signal implementation achieves glitch-free propagation through a **push-pull directed acyclic graph (DAG)** algorithm:

- **Push phase**: when a signal is written, it "pushes" a dirty notification up through all dependent computed signals and effects — but it does NOT recalculate any values
- **Pull phase**: when a computed signal is actually read (or an effect runs), it "pulls" the latest value by recomputing from source signals, verifying that each dependency is actually dirty before recomputing

This separation ensures that by the time any derived value is read, all its upstream dependencies have a chance to settle.

## What Is a Glitch

```
Consider three signals:
  a = signal(1)
  b = computed(() => a() * 2)   // depends on a
  c = computed(() => a() + b()) // depends on both a and b

Mathematically: c = a + (a * 2) = 3a

When a changes from 1 to 2, the expected result is c = 6.

In a glitchy push-based system (naive implementation):
  Step 1: a changes to 2 → push to b and c immediately
  Step 2: c re-runs: a() = 2, b() = 2 (stale!) → c = 4 ← GLITCH (intermediate state)
  Step 3: b re-runs: b = 4
  Step 4: c re-runs again: a() = 2, b() = 4 → c = 6 ✓

The problem: c was computed with inconsistent values (new a, old b).
An observer/effect that ran during step 2 would see a wrong value.

Topological order execution partially solves this, but fails for dynamic
dependency graphs and still computes eagerly even if the result is never read.
```

## The Problem with Push-Only Systems

```typescript
// Naive push-based system (like early MobX reaction or Vue 2 watchers)
class NaiveComputed<T> {
  private value: T;
  private subscribers: Set<() => void> = new Set();

  constructor(private compute: () => T, deps: NaiveSignal<any>[]) {
    this.value = this.compute();
    // Subscribe to each dependency — PUSH model
    deps.forEach(dep => dep.subscribe(() => this.recompute()));
  }

  private recompute() {
    const oldValue = this.value;
    this.value = this.compute(); // Recomputes immediately on any dep change
    if (this.value !== oldValue) {
      this.subscribers.forEach(s => s()); // Immediately push to subscribers
    }
  }

  get(): T { return this.value; }
}

// Problem 1: Eager recomputation even when result is never read
// Problem 2: Observers see intermediate (glitchy) values
// Problem 3: Computed values recompute even when inputs cancel out
// Example: signal(1) → signal(2) → signal(1): computed fires twice unnecessarily
```

## Push-Pull DAG: The Solution

```
The algorithm has two distinct phases:

WRITE (push phase — cheap, no recomputation):
───────────────────────────────────────────────
a.set(2)
  │
  ├── Mark a as DIRTY
  │
  ├── Walk graph: find all computeds that (transitively) depend on a
  │     b (depends on a) → mark b as MAYBE_DIRTY
  │     c (depends on a) → mark c as MAYBE_DIRTY
  │     (b also marks c as MAYBE_DIRTY, but it's already marked)
  │
  └── Notify scheduled effects that something might have changed
      (but don't run them yet)


READ (pull phase — lazy recomputation):
───────────────────────────────────────
c()   ← triggered by effect or template read
  │
  ├── Is c DIRTY or MAYBE_DIRTY? → yes, check dependencies
  │
  ├── Check a: is a DIRTY? → yes
  │     a has no further deps to check → a's value is 2 (authoritative)
  │
  ├── Check b: is b DIRTY or MAYBE_DIRTY? → yes, recursively pull b
  │     b's dep a is DIRTY → recompute b = a() * 2 = 4
  │     mark b as CLEAN, value = 4
  │
  ├── All of c's deps now verified and settled
  └── Recompute c = a() + b() = 2 + 4 = 6 ✓ CLEAN, no glitch
```

## Angular Signals Internals

Angular's signal implementation lives in `packages/core/src/signals/`. The key types are:

```typescript
// Simplified signal node type from Angular source
interface ReactiveNode {
  version: number;           // Incremented when value changes
  dirty: boolean;            // Is the cached value stale?

  // Producer side (for signals and computeds that produce values)
  producerNode?: ReactiveNode[];     // deps that this node watches
  producerLastReadVersion?: number[]; // last version we read from each dep

  // Consumer side (for computeds and effects that consume values)
  consumerNode?: ReactiveNode[];     // what consumes this node
  consumerIndexOfThis?: number[];    // index in consumer's producerNode array
}

// Signal (writable)
interface SignalNode<T> extends ReactiveNode {
  value: T;
}

// Computed signal
interface ComputedNode<T> extends ReactiveNode {
  value: T | typeof UNSET;
  error: unknown | typeof UNSET;
  computation: () => T;
  equal: (a: T, b: T) => boolean;
}

// Effect
interface EffectNode extends ReactiveNode {
  fn: () => void;
  zone: Zone | null;
  scheduler: EffectScheduler;
}
```

### Version-Based Tracking

Angular uses a global monotonically-increasing version counter:

```typescript
let globalVersion = 0;

// When a signal is written:
function signalSet<T>(node: SignalNode<T>, value: T): void {
  if (node.equal(node.value, value)) return; // no change — skip
  node.value = value;
  node.version++;          // increment this signal's version
  globalVersion++;         // increment global version
  // Notify consumers that something changed (push phase)
  producerNotifyConsumers(node);
}

// When a computed checks if it is stale:
function computedIsStale<T>(node: ComputedNode<T>): boolean {
  for (let i = 0; i < node.producerNode!.length; i++) {
    const dep = node.producerNode![i];
    const lastReadVersion = node.producerLastReadVersion![i];
    // Has this dependency changed since we last computed?
    if (dep.version !== lastReadVersion) {
      return true;
    }
    // If dep is itself a computed that might be stale, check it recursively
    if (dep.dirty || (isComputedNode(dep) && computedIsStale(dep))) {
      return true;
    }
  }
  return false;
}
```

## The Mark-Then-Pull Algorithm

```typescript
// Push phase: mark consumers as potentially dirty
function producerNotifyConsumers(producer: ReactiveNode): void {
  const consumers = producer.consumerNode;
  if (!consumers) return;

  for (const consumer of consumers) {
    if (isEffectNode(consumer)) {
      // Effects get scheduled but NOT run immediately
      consumer.scheduler.notify(consumer);
    } else if (isComputedNode(consumer)) {
      // Computed signals become MAYBE_DIRTY
      markComputedMaybeDirty(consumer);
      // Propagate further down the graph
      producerNotifyConsumers(consumer);
    }
  }
}

// Pull phase: compute value on demand
function computedGet<T>(node: ComputedNode<T>): T {
  if (node.dirty) {
    // Definitely stale — recompute unconditionally
    return recomputeAndCache(node);
  }

  if (node.value !== UNSET) {
    // Check if any source has actually changed since last computation
    // This is the key optimization: walk deps, check versions
    if (!computedIsStale(node)) {
      // No dep has actually changed — return cached value (fast path)
      return node.value;
    }
  }

  // We are MAYBE_DIRTY and a dep version has changed — recompute
  return recomputeAndCache(node);
}

function recomputeAndCache<T>(node: ComputedNode<T>): T {
  // Track which signals are read during the computation
  // (automatic dependency tracking)
  const prevConsumer = setActiveConsumer(node);
  node.producerNode = []; // Reset dep list
  node.producerLastReadVersion = [];

  let newValue: T;
  try {
    newValue = node.computation();
    node.error = UNSET;
  } catch (e) {
    node.error = e;
    newValue = UNSET as any;
  } finally {
    setActiveConsumer(prevConsumer);
  }

  node.dirty = false;
  if (node.equal(node.value, newValue)) {
    // Value didn't change — version stays the same
    // This is important: downstream consumers won't need to recompute
    return node.value;
  }

  node.value = newValue;
  node.version++;
  return newValue;
}
```

### Automatic Dependency Tracking

```typescript
// When a signal is read inside a computation or effect,
// it registers itself as a dependency automatically

function signalGet<T>(node: SignalNode<T>): T {
  // If there's an active consumer (computed or effect running),
  // register this signal as a dependency
  const consumer = getActiveConsumer();
  if (consumer !== null) {
    producerAccessed(node, consumer);
  }
  return node.value;
}

function producerAccessed(producer: ReactiveNode, consumer: ReactiveNode): void {
  // Check if this producer is already tracked by the consumer
  // (prevents duplicates in the dep list)
  const existingIndex = findIndex(consumer.producerNode, producer);
  if (existingIndex !== -1) {
    // Already tracked — update the last-read version
    consumer.producerLastReadVersion![existingIndex] = producer.version;
    return;
  }

  // New dependency — add to both sides of the graph
  const index = consumer.producerNode!.length;
  consumer.producerNode!.push(producer);
  consumer.producerLastReadVersion!.push(producer.version);

  producer.consumerNode = producer.consumerNode ?? [];
  producer.consumerNode.push(consumer);
  consumer.consumerIndexOfThis = consumer.consumerIndexOfThis ?? [];
  consumer.consumerIndexOfThis.push(producer.consumerNode.length - 1);
}
```

## Computed Signals and Memoization

```typescript
// A key optimization: if a computed's value does not change after recomputation,
// downstream consumers do NOT need to recompute (version stays the same).

const a = signal(5);
const b = computed(() => a() > 0 ? 'positive' : 'negative'); // depends on a
const c = computed(() => b().toUpperCase()); // depends on b

a.set(10);
// a version: 1 → 2
// b MAYBE_DIRTY → pull b: 10 > 0 → 'positive' (same as before!)
//   b's value didn't change → b.version stays the same
// c: checks b.version — unchanged → c is still CLEAN!
// c does NOT recompute even though a changed

// This is glitch-free AND efficient: unnecessary recomputation is avoided
// when the derived value is the same despite the source changing.
```

```typescript
// Custom equality function for non-primitive values
const userProfile = computed(
  () => ({ name: users.name(), age: users.age() }),
  {
    equal: (a, b) => a.name === b.name && a.age === b.age,
    // Without this, every computation produces a new object → never equal
    // With this, downstream signals only update when name or age actually changes
  },
);
```

## Effects and Scheduling

Effects are consumers that run side effects (DOM updates, network calls) in response to signal changes. Angular schedules effect execution carefully to avoid running effects mid-render.

```typescript
// Creating an effect
import { effect, signal } from '@angular/core';

@Component({ /* ... */ })
class LogComponent {
  count = signal(0);

  constructor() {
    // Effect automatically tracks dependencies
    effect(() => {
      console.log('Count changed:', this.count());
      // Any signal read here becomes a dependency
    });
  }
}
```

### Effect Scheduling in Angular

```
Signal write → mark effects as "needs run"
     │
     │  (NOT run immediately — scheduled)
     ▼
Angular's scheduler queues the effect
     │
     │  (runs after current synchronous operations)
     ▼
Effect executes in microtask or during next CD cycle
     │
     └── Effect runs its fn() → reads signals (pull phase)
         → evaluates glitch-free (all deps settled)
```

```typescript
// Angular's EffectScheduler for view effects (tied to change detection):
class ViewEffectScheduler {
  private readonly effectsToRun = new Set<EffectNode>();

  notify(effect: EffectNode): void {
    this.effectsToRun.add(effect);
    // Schedule Angular change detection (or use queueMicrotask in zoneless)
    this.scheduleChangeDetection();
  }

  flush(): void {
    for (const effect of this.effectsToRun) {
      this.effectsToRun.delete(effect);
      runEffect(effect);
    }
  }
}

// Root effects (created outside components) use a microtask scheduler:
class RootEffectScheduler {
  private scheduled = false;

  notify(effect: EffectNode): void {
    enqueueRootEffect(effect);
    if (!this.scheduled) {
      this.scheduled = true;
      queueMicrotask(() => {
        this.scheduled = false;
        flushRootEffects();
      });
    }
  }
}
```

## Signal Graph Traversal

The complete traversal for a write and subsequent read:

```
Graph:
  a ──→ b ──→ d
  a ──→ c ──→ d
  b ──→ e
  (d and e are effects/templates)

After a.set(newValue):

Push (mark dirty):
  a.version++
  a → b: mark b MAYBE_DIRTY, recurse
    b → d: mark d MAYBE_DIRTY (effect: schedule)
    b → e: mark e MAYBE_DIRTY (effect: schedule)
  a → c: mark c MAYBE_DIRTY, recurse
    c → d: already marked, skip

State after push:
  a: DIRTY (new version)
  b: MAYBE_DIRTY
  c: MAYBE_DIRTY
  d: effect scheduled
  e: effect scheduled
  (no values have been computed yet)

Pull (when d effect runs):
  d reads b() and c()
  
  Pull b:
    b.producerNode[0] = a, check a.version vs lastRead → CHANGED
    Recompute b = fn(a) → new value
    b.version++
  
  Pull c:
    c.producerNode[0] = a, check a.version vs lastRead → CHANGED
    Recompute c = fn(a) → new value
    c.version++
  
  All of d's deps settled
  Recompute d = fn(b, c)
  → Sees latest b AND latest c → no glitch
```

## Comparison with Other Reactive Systems

```
System          | Algorithm      | Glitch-free | Lazy    | Auto-track
──────────────────────────────────────────────────────────────────────
Angular Signals | Push-pull DAG  | ✓           | ✓       | ✓
SolidJS         | Push-pull DAG  | ✓           | ✓       | ✓
MobX 5+         | Push-pull      | ✓           | Partial | ✓
Vue Computed    | Push-pull      | ✓           | ✓       | ✓
RxJS BehaviorSubject | Push only | ✗           | ✗       | ✗
Svelte stores   | Push only      | ✗           | ✗       | ✗ (compile-time)
Redux (naive)   | Pull only      | ✓           | ✓       | ✗ (manual)
```

```typescript
// SolidJS uses the same push-pull approach (createSignal / createMemo)
// This is not coincidental — Angular's signals were heavily inspired by SolidJS

// SolidJS createMemo (equivalent to Angular computed):
import { createSignal, createMemo } from 'solid-js';

const [a, setA] = createSignal(1);
const b = createMemo(() => a() * 2);
const c = createMemo(() => a() + b()); // Glitch-free: always 3a

setA(2); // c will read 6, never the intermediate 4
```

## Common Pitfalls

### Pitfall 1: Writing Signals Inside Computed

```typescript
// BAD: Side effects in computed create potential cycles and bugs
const counter = signal(0);
const doubled = computed(() => {
  counter.set(counter() + 1); // Writes inside computed — throws in Angular
  return counter() * 2;
});

// Computed signals must be pure functions.
// Angular throws an error if you write to a signal inside a computed.
// Use effects for side effects.
```

### Pitfall 2: Object Mutation Not Detected

```typescript
// BAD: Angular signals use reference equality for change detection
const user = signal({ name: 'Alice', age: 30 });

// Mutating the object does NOT trigger updates — reference unchanged
user().name = 'Bob'; // NO update — signal value is still the same object

// GOOD: Replace the object with a new one
user.update(u => ({ ...u, name: 'Bob' }));
// OR
user.set({ ...user(), name: 'Bob' });
```

### Pitfall 3: Creating Signals in Computed

```typescript
// BAD: Creating new signals inside computed breaks the DAG topology
const items = signal([1, 2, 3]);
const itemSignals = computed(() => {
  return items().map(i => signal(i)); // Creates new signals on every recomputation!
});
// The downstream graph gets rebuilt on every change — hugely inefficient

// GOOD: Keep the graph structure stable
const items = signal([1, 2, 3]);
const doubledItems = computed(() => items().map(i => i * 2));
```

### Pitfall 4: Effects That Create Infinite Loops

```typescript
// BAD: Effect writes to a signal it reads — creates a loop
const count = signal(0);

effect(() => {
  console.log(count());
  count.set(count() + 1); // Reads and writes the same signal → infinite loop
});

// GOOD: Use untracked() to read without creating a dependency
import { untracked } from '@angular/core';

effect(() => {
  const currentCount = count(); // Creates dependency — runs when count changes
  console.log('Count changed to:', currentCount);
  // Read without dependency using untracked:
  const snapshot = untracked(() => someOtherSignal());
  // Write is fine if not read in same effect:
  logService.log(currentCount, snapshot);
});
```

### Pitfall 5: Relying on Synchronous Effect Execution

```typescript
// BAD: Expecting effects to run synchronously after signal writes
const value = signal(0);
let effectRan = false;

effect(() => {
  value();
  effectRan = true;
});

value.set(1);
console.log(effectRan); // FALSE — effect is scheduled, not run immediately

// Effects run asynchronously (microtask or change detection cycle).
// If you need synchronous derived values, use computed() instead.
```

## Interview Questions

### Question 1: What is a "glitch" in a reactive system?

**Answer**: A glitch is when a derived value (computed/observer) temporarily sees an inconsistent combination of signal values during a single logical update. For example, if `c = a + b` and both `a` and `b` change together (where `b = a * 2`, so `c` should always equal `3a`), a naive push system might let `c` compute with the new `a` but the old `b`, producing an incorrect intermediate result. A glitch-free system guarantees that observers only ever see fully settled states.

### Question 2: How does Angular's push-pull DAG prevent glitches?

**Answer**: Angular separates signal writes from value recomputation into two phases. In the push phase (on write), Angular traverses the signal graph and marks downstream computed signals as MAYBE_DIRTY but does NOT recompute any values. In the pull phase (on read), Angular lazily recomputes a computed signal by first recursively checking and recomputing all its dependencies. By the time a computed value is read, all its dependencies have been updated to their final values. No observer sees an inconsistent intermediate state.

### Question 3: What is the role of the version counter in Angular's signal implementation?

**Answer**: Every signal node maintains a version number that increments when its value changes. Computed signals store the last-read version of each dependency. When a computed checks if it is stale (in `computedIsStale`), it compares its stored dependency versions against the current dependency versions. If they all match, the cached value is still valid and the computed returns it without recomputing. If a computed recomputes to the same value, its version does NOT increment — meaning downstream nodes do not need to recompute either. This makes the system efficient: changes that cancel out don't propagate unnecessary work.

### Question 4: How do effects fit into the glitch-free model?

**Answer**: Effects are consumers at the "leaf" of the signal graph — they run side effects (DOM updates, logging, network calls) in response to signal changes. Angular schedules effects asynchronously (not immediately on signal write) to ensure they run only after the current synchronous code has settled. When an effect runs, it performs a pull — reading signals and recomputing computed dependencies lazily from the graph root. By the time the effect reads any value, the entire graph has settled to its final state, so effects are always glitch-free.

### Question 5: Why does a computed signal's version not increment when the recomputed value is the same as the cached value?

**Answer**: This is the "diamond dependency" optimization. If `a` changes but the derived value of a `computed` based on `a` happens to produce the same result (e.g., `computed(() => a() > 0 ? 'positive' : 'negative')` when `a` changes from 5 to 10 — both are positive), Angular does not increment the computed's version. Downstream computeds that depend on this computed then check its version, see it hasn't changed, and skip recomputation. This prevents unnecessary cascading recomputation through the graph when intermediate derived values don't actually change.

## Key Takeaways

1. **Two-phase algorithm**: push (mark dirty, no recompute) → pull (lazy recompute on demand) — the separation is what makes the system both glitch-free and efficient

2. **MAYBE_DIRTY vs DIRTY**: MAYBE_DIRTY means "a source changed, check if I actually need to recompute"; DIRTY means "definitely recompute" — the distinction enables the version-check fast path

3. **Version numbers enable the fast path**: if all dependencies' versions match their last-read versions, return cached value without recomputing (O(depth) check instead of recomputation)

4. **Value stability stops propagation**: if a computed recomputes to the same value, its version stays the same and downstream nodes don't propagate changes

5. **Effects are scheduled, not eager**: effects run asynchronously after writes, ensuring they always see a fully settled graph state

6. **Automatic dependency tracking**: dependencies are recorded dynamically at runtime — if a signal read is conditional, only the signals actually read in the current execution are tracked

7. **untracked() escapes tracking**: reads inside `untracked()` do not create dependencies, useful for accessing signals in effects without causing re-runs

8. **Computed must be pure**: no writes inside computed — this preserves the DAG structure and prevents cycles

## Resources

- [Angular Signals RFC](https://github.com/angular/angular/discussions/49685)
- [Angular source: packages/core/src/signals/](https://github.com/angular/angular/tree/main/packages/core/src/signals)
- [SolidJS fine-grained reactivity deep dive](https://dev.to/ryansolid/a-hands-on-introduction-to-fine-grained-reactivity-3ndf)
- ["Glitch prevention in reactive systems" — Deprecating the Observer Pattern](https://hal.science/hal-00926548/document)
- [MobX conceptual overview (push-pull model)](https://mobx.js.org/computeds.html#rules)
- [Angular.dev: Signals guide](https://angular.dev/guide/signals)
- ["Signals vs Observables" by Pawel Kozlowski (AngularNation 2023)](https://www.youtube.com/watch?v=iA6iyoantuo)
