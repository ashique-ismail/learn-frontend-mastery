# Solid.js and Fine-Grained Reactivity

## Overview

Solid.js is a declarative JavaScript library for building user interfaces that achieves exceptional performance by abandoning the virtual DOM entirely. Instead of re-running component functions on every state change, Solid compiles JSX into actual DOM operations and uses a synchronous reactive graph — signals, memos, and effects — to track exactly which DOM nodes depend on which pieces of state. When state changes, only the affected DOM nodes update, with no diffing, no component re-rendering, and no garbage collection pressure from discarded virtual nodes.

```
React (Virtual DOM model):
  State change
    └─► Re-run component function
          └─► Create new vDOM tree
                └─► Diff old/new vDOM
                      └─► Apply minimal DOM patches
  Cost: proportional to component subtree size

Solid (Fine-grained reactive model):
  State change (signal)
    └─► Notify subscribers directly
          └─► Update specific DOM node
  Cost: proportional to number of changed nodes (often 1)
```

This design makes Solid one of the fastest UI frameworks on benchmarks, while the JSX syntax keeps it familiar to React developers. Understanding its reactivity model builds intuition that transfers directly to Angular Signals, Vue's Composition API, and Qwik.

## Signals — The Core Primitive

A signal is a getter/setter pair around a reactive value. Reading a signal inside a reactive scope (effect, memo, JSX expression) registers a subscription. Writing a signal notifies all subscribers synchronously.

```typescript
import { createSignal } from 'solid-js';

// createSignal returns [getter, setter] — note: getter is a function, not a value
const [count, setCount] = createSignal(0);

// Read: always call as function
console.log(count()); // 0

// Write: call setter
setCount(1);
console.log(count()); // 1

// Functional update (like React's setState updater form)
setCount(prev => prev + 1);
console.log(count()); // 2
```

The getter-as-function pattern is intentional: it allows Solid's runtime to detect when a signal is read inside a reactive context and register the dependency.

```typescript
// Reactive context example — effect tracks signal reads
import { createSignal, createEffect } from 'solid-js';

const [name, setName] = createSignal('Alice');
const [age, setAge] = createSignal(30);

createEffect(() => {
  // This effect reads both `name` and `age`
  // It re-runs whenever EITHER changes
  console.log(`${name()} is ${age()} years old`);
});

setName('Bob');  // Effect re-runs: "Bob is 30 years old"
setAge(31);      // Effect re-runs: "Bob is 31 years old"
```

## createSignal Deep Dive

```typescript
import { createSignal } from 'solid-js';

// Signal with equality check (default: strict equality ===)
const [user, setUser] = createSignal(
  { name: 'Alice', role: 'admin' },
  {
    // Custom equality — avoids unnecessary updates
    equals: (prev, next) => prev.name === next.name && prev.role === next.role
  }
);

// Setting same value won't trigger subscribers
setUser({ name: 'Alice', role: 'admin' }); // No re-run (equal by custom check)
setUser({ name: 'Bob', role: 'admin' });    // Triggers re-run

// Disable equality check entirely (always notify)
const [list, setList] = createSignal([], { equals: false });
setList(l => (l.push(1), l)); // Mutate and force notify
```

## createMemo — Derived State

`createMemo` creates a memoized computation that only recalculates when its signal dependencies change. Unlike `useCallback`/`useMemo` in React (which are hints), Solid's `createMemo` is guaranteed to cache — it will not recompute unless dependencies change, and it will not recompute more than once per dependency change.

```typescript
import { createSignal, createMemo } from 'solid-js';

const [items, setItems] = createSignal([
  { id: 1, name: 'Apple', price: 1.5, qty: 3 },
  { id: 2, name: 'Banana', price: 0.5, qty: 5 },
  { id: 3, name: 'Cherry', price: 3.0, qty: 2 },
]);
const [taxRate, setTaxRate] = createSignal(0.08);

// Memo only recalculates when `items` or `taxRate` changes
const subtotal = createMemo(() =>
  items().reduce((sum, item) => sum + item.price * item.qty, 0)
);

const tax = createMemo(() => subtotal() * taxRate());

const total = createMemo(() => subtotal() + tax());

// Reading these in JSX or effects is free — no recalculation unless deps change
console.log(total()); // 13.72

setItems(prev => [...prev, { id: 4, name: 'Date', price: 4.0, qty: 1 }]);
// Only subtotal, tax, and total recalculate; in that order
console.log(total()); // 18.20

setTaxRate(0.1);
// Only tax and total recalculate (subtotal didn't change)
```

### Memo vs Effect

```typescript
// createMemo: has a return value, used for derived state
// Re-computation is lazy — only when memo is read AND deps changed
const doubled = createMemo(() => count() * 2);

// createEffect: no return value, used for side effects
// Re-runs eagerly when deps change (even if no one reads its output)
createEffect(() => {
  document.title = `Count: ${count()}`;
});
```

## createEffect — Reactive Side Effects

```typescript
import { createSignal, createEffect, onCleanup } from 'solid-js';

const [userId, setUserId] = createSignal<string | null>(null);

createEffect(() => {
  const id = userId();
  if (!id) return;

  // Side effect: start data fetch
  const controller = new AbortController();

  fetch(`/api/users/${id}`, { signal: controller.signal })
    .then(r => r.json())
    .then(data => console.log('User:', data))
    .catch(err => {
      if (err.name !== 'AbortError') console.error(err);
    });

  // onCleanup runs BEFORE the effect re-runs or component unmounts
  // This cancels the inflight request when userId changes
  onCleanup(() => controller.abort());
});

setUserId('user-1'); // Fetches /api/users/user-1
setUserId('user-2'); // Aborts previous fetch, starts /api/users/user-2
```

### Effect Execution Timing

```typescript
// Effects run synchronously AFTER the reactive update batch
import { createSignal, createEffect, batch } from 'solid-js';

const [a, setA] = createSignal(0);
const [b, setB] = createSignal(0);

createEffect(() => {
  console.log('Effect:', a(), b());
});

// Without batch: effect runs twice
setA(1); // Effect: 1 0
setB(1); // Effect: 1 1

// With batch: defers effects until batch completes, effect runs once
batch(() => {
  setA(2);
  setB(2);
}); // Effect: 2 2 (runs once, not twice)
```

## Solid Components — Not Re-Rendered

This is the most critical mental shift from React: **Solid component functions run exactly once**. They are factory functions that set up reactive subscriptions, not render functions that re-run on state changes.

```typescript
import { createSignal, createEffect, onMount, onCleanup } from 'solid-js';

function Counter() {
  // This function runs ONCE — not on every render like React
  console.log('Counter created'); // Only printed once

  const [count, setCount] = createSignal(0);

  // Setup subscription — runs once, but the DOM updates reactively
  createEffect(() => {
    // This runs whenever count() changes, but Counter() itself doesn't re-run
    console.log('Count changed to:', count());
  });

  // Lifecycle hooks
  onMount(() => {
    console.log('Mounted to DOM');
  });

  onCleanup(() => {
    console.log('Removed from DOM');
  });

  // JSX compiles to reactive DOM expressions, NOT to a vDOM tree
  // The `{count()}` expression becomes a live subscription in the DOM
  return (
    <div>
      <p>Count: {count()}</p>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
    </div>
  );
}
```

### What the JSX Compiles To

```typescript
// Your Solid JSX:
// <p>Count: {count()}</p>

// Compiled output (roughly):
const _el = document.createElement('p');
_el.textContent = 'Count: ';
const _text = document.createTextNode('');
_el.appendChild(_text);

// Reactive subscription: updates _text whenever count() changes
createEffect(() => {
  _text.data = count();
});
```

No virtual DOM creation. No diffing. Just a direct DOM text node that Solid updates when the signal changes.

## Stores — Nested Reactive State

For complex nested state, `createStore` provides fine-grained reactivity at any depth:

```typescript
import { createStore, produce, reconcile } from 'solid-js/store';

// createStore returns [state proxy, setter]
const [state, setState] = createStore({
  todos: [
    { id: 1, text: 'Learn Solid', done: false },
    { id: 2, text: 'Build app', done: false },
  ],
  filter: 'all' as 'all' | 'active' | 'done',
});

// Path-based update — only 'done' property of todo[0] triggers subscribers
setState('todos', 0, 'done', true);

// Functional update with produce (Immer-style)
setState(produce(draft => {
  draft.todos.push({ id: 3, text: 'Ship it', done: false });
  draft.filter = 'active';
}));

// Reconcile — smart merge for replacing arrays/objects from API
// Preserves identity of unchanged items to avoid unnecessary re-renders
const newTodos = await fetchTodos();
setState('todos', reconcile(newTodos, { key: 'id' }));
```

## Control Flow — `<For>`, `<Show>`, `<Switch>`

Solid provides built-in control flow components that are reactive-aware and avoid unnecessary DOM reconciliation:

```typescript
import { For, Show, Switch, Match, createSignal } from 'solid-js';

interface Todo { id: number; text: string; done: boolean }

function TodoList() {
  const [todos, setTodos] = createSignal<Todo[]>([
    { id: 1, text: 'Learn Solid', done: false },
    { id: 2, text: 'Ship it', done: true },
  ]);
  const [showDone, setShowDone] = createSignal(true);

  const filtered = () =>
    showDone() ? todos() : todos().filter(t => !t.done);

  return (
    <div>
      {/* For: keyed iteration — only updates changed items */}
      <For each={filtered()} fallback={<p>No todos</p>}>
        {(todo, index) => (
          // This function runs once per item — like a component
          <div>
            <span>{index() + 1}. {todo.text}</span>
            {/* Show: conditional rendering with optional fallback */}
            <Show when={todo.done} fallback={<span> (pending)</span>}>
              <span> ✓ Done</span>
            </Show>
          </div>
        )}
      </For>

      {/* Switch/Match: multi-condition branching */}
      <Switch fallback={<p>Loading...</p>}>
        <Match when={todos().length === 0}>
          <p>Add your first todo!</p>
        </Match>
        <Match when={todos().every(t => t.done)}>
          <p>All done! 🎉</p>
        </Match>
      </Switch>

      <button onClick={() => setShowDone(v => !v)}>
        {showDone() ? 'Hide' : 'Show'} completed
      </button>
    </div>
  );
}
```

### `<For>` vs Array.map()

```typescript
// DO NOT use Array.map() for lists — loses fine-grained updates
// This re-creates all DOM nodes when the array changes
<ul>
  {todos().map(todo => <li>{todo.text}</li>)}
</ul>

// USE <For> — tracks items by key, only updates changed items
// Adding one item only creates one new <li>, not all of them
<ul>
  <For each={todos()}>
    {todo => <li>{todo.text}</li>}
  </For>
</ul>
```

## Context API

```typescript
import { createContext, useContext, createSignal, ParentComponent } from 'solid-js';

interface ThemeContextValue {
  theme: () => 'light' | 'dark';
  toggleTheme: () => void;
}

const ThemeContext = createContext<ThemeContextValue>();

export const ThemeProvider: ParentComponent = (props) => {
  const [theme, setTheme] = createSignal<'light' | 'dark'>('light');

  const value: ThemeContextValue = {
    theme,
    toggleTheme: () => setTheme(t => t === 'light' ? 'dark' : 'light'),
  };

  return (
    <ThemeContext.Provider value={value}>
      {props.children}
    </ThemeContext.Provider>
  );
};

export function useTheme(): ThemeContextValue {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error('useTheme must be used inside ThemeProvider');
  return ctx;
}

// Usage
function ThemedButton() {
  const { theme, toggleTheme } = useTheme();
  return (
    <button
      class={`btn btn-${theme()}`}
      onClick={toggleTheme}
    >
      Toggle Theme
    </button>
  );
}
```

## Comparison with React Hooks

```
Concept                | React                    | Solid
──────────────────────|──────────────────────────|──────────────────────────
State                  | useState hook            | createSignal
Derived state          | useMemo                  | createMemo (guaranteed)
Side effects           | useEffect                | createEffect
Reactive context       | Context + useContext      | createContext + useContext
Component re-render    | On every state change     | Never (runs once)
DOM update granularity | Component subtree         | Individual DOM nodes
Batching               | Automatic (React 18)      | Manual batch() / auto in browser events
List rendering         | .map() + key prop         | <For> component
Conditional rendering  | Ternary / &&             | <Show> component
Cleanup                | useEffect return fn       | onCleanup()
Mount                  | useEffect([])             | onMount()
Scheduler              | Concurrent (Fiber)        | Synchronous
Virtual DOM            | Yes                       | No
Bundle size (min+gz)   | ~45KB (react + react-dom) | ~7KB
```

## Async Data — createResource

```typescript
import { createSignal, createResource, Suspense, ErrorBoundary } from 'solid-js';

async function fetchUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error('User not found');
  return res.json();
}

function UserProfile() {
  const [userId, setUserId] = createSignal('user-1');

  // createResource: reactive async data fetching
  // Re-fetches when userId changes
  const [user, { refetch, mutate }] = createResource(userId, fetchUser);

  return (
    <ErrorBoundary fallback={err => <p>Error: {err.message}</p>}>
      <Suspense fallback={<p>Loading...</p>}>
        {/* user() is undefined while loading, throws Promise (Suspense), throws Error (ErrorBoundary) */}
        <Show when={user()}>
          {resolved => (
            <div>
              <h1>{resolved().name}</h1>
              <p>Loading: {user.loading.toString()}</p>
              <button onClick={refetch}>Refresh</button>
              <button onClick={() => mutate(u => ({ ...u!, name: 'Edited' }))}>
                Optimistic edit
              </button>
            </div>
          )}
        </Show>
      </Suspense>
    </ErrorBoundary>
  );
}
```

## Performance Characteristics

```
Benchmark: JS Framework Benchmark (v1.0, higher = better)
                     | Create 1k rows | Replace 1k rows | Partial update | Memory
─────────────────────|────────────────|─────────────────|───────────────|────────
Vanilla JS           | 1.0x (baseline)| 1.0x            | 1.0x          | 1.0x
Solid                | 1.08x          | 1.03x           | 1.04x         | 1.3x
Svelte               | 1.33x          | 1.39x           | 1.46x         | 1.3x
Vue 3                | 1.35x          | 1.47x           | 1.54x         | 1.8x
React 18             | 1.65x          | 1.84x           | 3.21x         | 2.5x
Angular 17           | 1.55x          | 1.73x           | 2.45x         | 2.0x

Lower multiplier = faster and more memory-efficient
Solid is within ~10% of vanilla JS
```

## Comparison with Angular Signals

Angular 17+ adopted signals directly inspired by Solid:

```typescript
// Angular Signals (Angular 17+)
import { signal, computed, effect } from '@angular/core';

const count = signal(0);
const doubled = computed(() => count() * 2);

effect(() => {
  console.log('count is', count());
});

count.set(1);
count.update(v => v + 1);

// Solid createSignal
import { createSignal, createMemo, createEffect } from 'solid-js';

const [count, setCount] = createSignal(0);
const doubled = createMemo(() => count() * 2);

createEffect(() => {
  console.log('count is', count());
});

setCount(1);
setCount(v => v + 1);
```

The APIs are nearly identical — Angular adopted Solid's signal semantics. The key differences are execution context (Angular runs in a component zone, Solid is standalone), change detection integration (Angular still needs to integrate signals with templates), and the broader framework ecosystem.

## Common Pitfalls

### Destructuring Loses Reactivity

```typescript
// WRONG — destructuring a signal breaks the reactive binding
function Bad() {
  const [state, setState] = createSignal({ count: 0, name: 'Alice' });

  // This reads state() once at component creation and never updates
  const { count, name } = state(); // ← WRONG

  return <p>{count}</p>; // Never updates
}

// CORRECT — always call the getter in the reactive context
function Good() {
  const [state, setState] = createSignal({ count: 0, name: 'Alice' });

  // Reading state() inside JSX registers a live subscription
  return <p>{state().count}</p>; // Updates correctly
}

// ALSO CORRECT — with a store (uses Proxy, destructuring works)
function GoodWithStore() {
  const [state, setState] = createStore({ count: 0, name: 'Alice' });
  const { count } = state; // Store uses Proxy, this is fine
  return <p>{count}</p>;
}
```

### Effect Dependency Tracking in Conditions

```typescript
// Conditional reads may not track correctly
createEffect(() => {
  if (isLoggedIn()) {
    // count() is only tracked when isLoggedIn() is true
    // If isLoggedIn becomes false then true again, effect re-runs
    // but count changes while logged out are missed
    console.log(count());
  }
});

// Solution: read all signals unconditionally OR restructure logic
createEffect(() => {
  const logged = isLoggedIn();
  const c = count(); // Always read — always tracked
  if (logged) {
    console.log(c);
  }
});
```

## Interview Questions

1. **Why doesn't Solid use a virtual DOM, and how does it achieve DOM updates without one?**
   Solid compiles JSX at build time into direct DOM operations. Each reactive expression (signal read inside JSX) becomes a fine-grained subscription — a `createEffect` that updates a specific DOM node. When a signal changes, it synchronously notifies only its direct subscribers (specific text nodes, attribute bindings, etc.). There is no tree to diff because Solid tracks dependencies at the granularity of individual DOM nodes, not component subtrees.

2. **What is the significance of component functions running exactly once in Solid?**
   React component functions re-run on every render, so all hook calls must maintain stable order across re-runs. Solid component functions are setup factories — they run once to establish the reactive graph and return live DOM. This eliminates rules-of-hooks constraints, makes closures naturally reactive (you capture the getter, not a snapshot), and means component setup cost is paid once regardless of how many times state updates.

3. **Explain the difference between `createMemo` in Solid and `useMemo` in React.**
   `useMemo` is an optimization hint — React can choose to recalculate it at any time, and developers must provide a dependency array that may become stale. `createMemo` is a reactive primitive that participates in Solid's dependency graph: it tracks its own signal reads, only recalculates when those dependencies change, and memoizes the result. It's guaranteed to never compute more than once per dependency change, and it notifies its own subscribers only if its output value changed.

4. **How does `<For>` differ from `.map()` in a Solid template, and why does it matter?**
   `.map()` inside JSX is called every time the signal holding the array changes, recreating all DOM elements each time. `<For>` uses a keyed algorithm — it tracks array items by identity and only creates/destroys DOM nodes for added/removed items. When an item's properties change, only that item's DOM updates. This is the difference between O(n) DOM recreation and O(changed) DOM updates.

5. **What is the purpose of `batch()` in Solid, and when is it applied automatically?**
   `batch()` defers notification of effects until the end of the batch, preventing intermediate states from triggering unnecessary re-computations. Inside browser event handlers, Solid automatically batches signal updates. Outside event handlers (e.g., in setTimeout, async code, manual imperative calls), updates are not automatically batched. Use `batch()` explicitly when multiple signals should update atomically.

6. **How does `createStore` differ from `createSignal` for object/array state?**
   `createSignal` treats its value atomically — any setter call notifies all subscribers even if only one property changed. `createStore` wraps objects in a Proxy, tracking reads at the property level. A component reading `state.user.name` subscribes only to that specific path. Changing `state.user.age` does not notify the `name` subscriber. This is fine-grained reactivity for nested data structures.

7. **Describe Solid's `createResource` and how it integrates with Suspense.**
   `createResource` is a reactive wrapper around async data fetching. It takes a source signal and an async fetcher function. When the source changes, it re-fetches. While fetching, `resource()` throws a Promise (Suspense integration). When the fetch errors, it throws the error (ErrorBoundary integration). When resolved, it returns the data. This is the same pattern React's `use()` hook adopts in React 19.

8. **How do Solid's performance characteristics compare to React on the JS Framework Benchmark, and what architectural decisions produce the difference?**
   Solid typically benchmarks within 5–10% of vanilla JavaScript, while React is 60–90% slower for DOM operations. The gap comes from: (1) no virtual DOM creation/garbage collection, (2) no component re-execution on updates, (3) fine-grained subscriptions eliminating unnecessary work, (4) synchronous updates (no scheduler overhead for simple cases). React's concurrent scheduler adds overhead for simple cases but enables time-slicing for complex apps — Solid trades that flexibility for raw throughput.
