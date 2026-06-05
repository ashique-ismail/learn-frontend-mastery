# useSyncExternalStore and Tearing in Concurrent React

## The Idea

**In plain English:** React can pause drawing your page mid-way to handle something more urgent, and if outside data changes during that pause, different parts of the page can end up showing mismatched information — like a price tag and a shopping cart showing different totals at the same moment. `useSyncExternalStore` is a special React tool that prevents this by making sure the whole page either uses the old data or the new data, never a mix of both.

**Real-world analogy:** Imagine a restaurant scoreboard that shows live sports scores. A waiter is updating three separate chalkboard signs around the room with the current score. Halfway through updating the second sign, the game scores change. Now sign 1 shows the old score, sign 2 is mid-update, and sign 3 still has the old score — customers in different parts of the room see different "current" scores at the same moment.

- The scoreboard signs around the room = the different React components on the page
- The waiter pausing mid-update = React yielding (pausing) a render to handle higher-priority work
- The score changing while the waiter is mid-round = an external data store updating during a concurrent render
- A single announcer who freezes all signs, updates them all at once, then unfreezes = `useSyncExternalStore`, which ensures every component sees the same consistent snapshot

---

## Table of Contents

- [Overview](#overview)
- [The Tearing Problem](#the-tearing-problem)
- [Why Concurrent Mode Makes Tearing Possible](#why-concurrent-mode-makes-tearing-possible)
- [useSyncExternalStore Contract](#usesyncexternalstore-contract)
- [getSnapshot Rules](#getsnapshot-rules)
- [Internal Implementation](#internal-implementation)
- [subscribe Contract](#subscribe-contract)
- [Server-Side Rendering with getServerSnapshot](#server-side-rendering-with-getserversnapshot)
- [Integrating Third-Party Stores](#integrating-third-party-stores)
- [Common Pitfalls](#common-pitfalls)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Overview

`useSyncExternalStore` is a React hook introduced in React 18 that enables safe subscription to external (non-React) state stores in concurrent mode. It solves the "tearing" problem — a class of visual inconsistency bug that can occur when React renders the same component tree multiple times with different values of an external store.

Before concurrent mode, a React render was synchronous and uninterruptible. One pass through the tree, one consistent snapshot of every store. Concurrent mode introduces interruptible renders that can be paused, discarded, and restarted — breaking the snapshot assumption for external stores.

`useSyncExternalStore` provides a way for external stores to integrate with React's concurrent scheduler while guaranteeing consistency: either the entire committed tree sees one snapshot, or React throws away the partial render and retries.

## The Tearing Problem

"Tearing" refers to a situation where different parts of the UI show different, inconsistent values of the same underlying state, within a single rendered frame.

### A Visual Example

```
External store: { theme: 'dark', userCount: 42 }

Render starts...
  - <Header> reads theme → 'dark' ✓
  - <Sidebar> reads theme → 'dark' ✓
  [React yields to browser for 5ms]
  [External store update fires: theme changes to 'light']
  - <Body> reads theme → 'light' ✗ ← TEAR! Different value in same render pass

Committed DOM:
  Header: "dark theme"
  Sidebar: "dark theme"
  Body: "light theme"   ← inconsistent

User sees a page that is half dark, half light.
```

### Concrete Code That Tears (Without useSyncExternalStore)

```typescript
// A simple external store
const themeStore = {
  _theme: 'dark' as 'dark' | 'light',
  _listeners: new Set<() => void>(),

  getTheme() { return this._theme; },

  setTheme(theme: 'dark' | 'light') {
    this._theme = theme;
    this._listeners.forEach(l => l());
  },

  subscribe(listener: () => void) {
    this._listeners.add(listener);
    return () => this._listeners.delete(listener);
  },
};

// Component using useEffect + useState — UNSAFE in concurrent mode
function ThemeDisplay({ label }: { label: string }) {
  const [theme, setTheme] = useState(themeStore.getTheme());

  useEffect(() => {
    const unsubscribe = themeStore.subscribe(() => {
      setTheme(themeStore.getTheme()); // Update on store change
    });
    return unsubscribe;
  }, []);

  // Problem: useState was initialized at component mount time.
  // During a concurrent render, two renders of ThemeDisplay can
  // capture different values if the store updates mid-render.
  return <span data-theme={theme}>{label}: {theme}</span>;
}
```

## Why Concurrent Mode Makes Tearing Possible

```
Legacy sync mode (React 17 and earlier):

render() starts
  ├─ Component A reads store: snapshot S₀
  ├─ Component B reads store: snapshot S₀  ← same snapshot guaranteed
  ├─ Component C reads store: snapshot S₀
render() ends → commit
  DOM shows snapshot S₀ consistently


Concurrent mode (React 18+):

render() chunk 1 (0–5ms):
  ├─ Component A reads store: snapshot S₀
  ├─ Component B reads store: snapshot S₀
  yield to browser ←─────────────────────── external store update: S₀ → S₁

render() chunk 2 (resumed):
  ├─ Component C reads store: snapshot S₁  ← TEAR: C saw S₁, A and B saw S₀
render() ends → commit
  DOM shows A(S₀) + B(S₀) + C(S₁) ← inconsistent!
```

The store update happens asynchronously (from a WebSocket, setTimeout, user interaction handled outside React) and React has no way to know that the store changed mid-render unless the store opts into the protocol.

## useSyncExternalStore Contract

```typescript
function useSyncExternalStore<Snapshot>(
  subscribe: (onStoreChange: () => void) => () => void,
  getSnapshot: () => Snapshot,
  getServerSnapshot?: () => Snapshot,
): Snapshot
```

- **`subscribe`**: A function that subscribes to store changes. Must call `onStoreChange` whenever the store changes. Must return an unsubscribe function.
- **`getSnapshot`**: A function that returns the current snapshot of the store. **Must be referentially stable when the store has not changed** — returning a new object/array each call breaks optimization.
- **`getServerSnapshot`** (optional): Same as `getSnapshot` but used during SSR and during the initial hydration render. Required if the component is server-rendered.
- **Return value**: The current snapshot.

```typescript
// Correct usage with our themeStore
function ThemeDisplay({ label }: { label: string }) {
  const theme = useSyncExternalStore(
    themeStore.subscribe.bind(themeStore),     // subscribe
    () => themeStore.getTheme(),               // getSnapshot
    () => 'dark',                             // getServerSnapshot (SSR default)
  );

  return <span data-theme={theme}>{label}: {theme}</span>;
}
```

## getSnapshot Rules

`getSnapshot` is the most important function to get right. React calls it in multiple places — during render, after store updates, and during tearing detection — and makes specific assumptions about its behaviour.

### Rule 1: Return a Stable Reference When Unchanged

```typescript
// BAD: Creates a new object every call — React sees constant "changes"
// This causes infinite re-renders
const badSnapshot = () => ({
  theme: themeStore.theme,
  language: languageStore.language,
});

useSyncExternalStore(subscribe, badSnapshot); // ← will thrash

// GOOD: Return the same reference when nothing changed
let cachedSnapshot: { theme: string; language: string } | null = null;
let cachedTheme = themeStore.theme;
let cachedLanguage = languageStore.language;

const goodSnapshot = () => {
  if (themeStore.theme !== cachedTheme || languageStore.language !== cachedLanguage) {
    cachedTheme = themeStore.theme;
    cachedLanguage = languageStore.language;
    cachedSnapshot = { theme: cachedTheme, language: cachedLanguage };
  }
  return cachedSnapshot!;
};
```

### Rule 2: getSnapshot Must Be Synchronous

```typescript
// BAD: Async snapshot — React cannot use this
const asyncSnapshot = async () => {
  return await store.getValueAsync(); // This won't work
};

// getSnapshot must return synchronously. For async data, use Suspense + use(),
// or cache the latest resolved value and return it synchronously.

// GOOD: Cache the async value, expose it synchronously
let latestValue: Data | null = null;
store.subscribe(async () => {
  latestValue = await store.getValueAsync(); // Update cache
  notifyReact(); // Trigger re-render via the subscribe callback
});

const syncSnapshot = () => latestValue; // Returns cached value synchronously
```

### Rule 3: getSnapshot Must Be Pure

```typescript
// BAD: Side effects in getSnapshot
const sideEffectSnapshot = () => {
  console.log('Getting snapshot...'); // Side effect — React may call this many times
  store.incrementReadCount();          // Mutates state — definitely wrong
  return store.value;
};

// GOOD: Pure read
const pureSnapshot = () => store.value;
```

### Rule 4: getSnapshot Result Must Represent the Full Needed State

```typescript
// BAD: Snapshot doesn't capture all the state the component uses
function UserList() {
  // getSnapshot only returns the length, but the component needs the items too
  const count = useSyncExternalStore(subscribe, () => userStore.users.length);

  // This reads users directly — not protected by useSyncExternalStore!
  const users = userStore.users; // Can tear!

  return <div>{count} users: {users.map(u => u.name).join(', ')}</div>;
}

// GOOD: Include all used data in the snapshot
function UserList() {
  const users = useSyncExternalStore(subscribe, () => userStore.users);
  // Now both count and list come from the same consistent snapshot
  return <div>{users.length} users: {users.map(u => u.name).join(', ')}</div>;
}
```

## Internal Implementation

React's implementation of `useSyncExternalStore` has two key mechanisms: tear detection and forced synchronous re-render.

```typescript
// Simplified ReactFiberHooks.ts

function mountSyncExternalStore<T>(
  subscribe: (fn: () => void) => () => void,
  getSnapshot: () => T,
  getServerSnapshot?: () => T,
): T {
  const fiber = currentlyRenderingFiber!;
  const hook = mountWorkInProgressHook();

  let nextSnapshot: T;
  const isHydrating = getIsHydrating();
  if (isHydrating) {
    // During hydration use the server snapshot
    if (getServerSnapshot === undefined) {
      throw new Error('Missing getServerSnapshot');
    }
    nextSnapshot = getServerSnapshot();
  } else {
    nextSnapshot = getSnapshot();
    // Check if the root is concurrently rendering
    const root = getWorkInProgressRoot();
    if (root !== null && includesBlockingLane(root, renderLanes)) {
      // We're in a concurrent render — push this store onto the
      // list of stores to check for tearing after the render completes
      pushStoreConsistencyCheck(fiber, getSnapshot, nextSnapshot);
    }
  }

  hook.memoizedState = nextSnapshot;
  const inst: StoreInstance<T> = { value: nextSnapshot, getSnapshot };
  hook.queue = inst;

  // Subscribe to store changes (schedules re-render when store changes)
  useEffect(() => {
    checkIfSnapshotChanged(inst);
    const handleStoreChange = () => {
      if (checkIfSnapshotChanged(inst)) {
        // Store changed — trigger a synchronous re-render
        forceStoreRerender(fiber);
      }
    };
    const unsubscribe = subscribe(handleStoreChange);
    return unsubscribe;
  }, [subscribe]);

  return nextSnapshot;
}

function checkIfSnapshotChanged<T>(inst: StoreInstance<T>): boolean {
  const latestGetSnapshot = inst.getSnapshot;
  const prevValue = inst.value;
  try {
    const nextValue = latestGetSnapshot();
    return !Object.is(prevValue, nextValue); // referential equality check
  } catch {
    return true; // If getSnapshot throws, treat as changed
  }
}
```

### Tearing Detection After Render

```typescript
// After a concurrent render completes (before commit),
// React checks all stores that were read during the render
// to see if they've changed since the render started.

function checkIfWorkInProgressStoreIsStillConsistent(): void {
  const didScheduleRerender = didScheduleRenderPhaseUpdate;
  if (didScheduleRerender) return;

  const renderState = currentlyRenderingFiber!.memoizedState;
  // Walk through all hooks looking for useSyncExternalStore instances
  let node: Hook | null = renderState;
  while (node !== null) {
    const queue = node.queue;
    if (queue !== null && isStoreInstance(queue)) {
      const { value, getSnapshot } = queue;
      try {
        const nextValue = getSnapshot();
        if (!Object.is(value, nextValue)) {
          // TEAR DETECTED: store changed during this render
          // Throw to signal React should abort this render and restart
          throw new StoreConsistencyError();
        }
      } catch {
        // Abort and re-render synchronously
        scheduleUpdateOnFiber(currentlyRenderingFiber!, SyncLane, ...);
        return;
      }
    }
    node = node.next;
  }
}
```

### forceStoreRerender

When a store updates after the component is mounted, `useSyncExternalStore` forces a synchronous re-render using `SyncLane` — bypassing the scheduler's normal batching:

```typescript
function forceStoreRerender(fiber: Fiber): void {
  const root = enqueueConcurrentRenderForLane(fiber, SyncLane);
  if (root !== null) {
    scheduleUpdateOnFiber(root, fiber, SyncLane);
    entangleTransitions(root, fiber, SyncLane);
  }
}
// SyncLane forces a synchronous render, guaranteeing that the
// entire tree re-renders with the same new snapshot.
```

## subscribe Contract

The `subscribe` function must follow these rules:

```typescript
// 1. Must call onStoreChange whenever any value that getSnapshot depends on changes
// 2. Must return a cleanup function (unsubscribe)
// 3. Must NOT call onStoreChange synchronously during subscribe
// 4. Must be referentially stable (wrap in useCallback if defined inline)

// BAD: subscribe is a new function every render
function MyComponent() {
  useSyncExternalStore(
    // New function reference every render → React unsubscribes and resubscribes every render
    (callback) => {
      store.on('change', callback);
      return () => store.off('change', callback);
    },
    getSnapshot,
  );
}

// GOOD: Stable subscribe function
const subscribe = useCallback((callback: () => void) => {
  store.on('change', callback);
  return () => store.off('change', callback);
}, []); // empty deps — store never changes

useSyncExternalStore(subscribe, getSnapshot);

// BEST: Define subscribe at module level (outside component)
// so it has a stable identity without useCallback
const subscribeToStore = (callback: () => void) => {
  store.on('change', callback);
  return () => store.off('change', callback);
};

function MyComponent() {
  useSyncExternalStore(subscribeToStore, getSnapshot);
}
```

## Server-Side Rendering with getServerSnapshot

```typescript
// getServerSnapshot provides the initial value for SSR and hydration.
// It must be deterministic: same call, same value, every time.

// BAD: Different on every SSR call — causes hydration mismatch
useSyncExternalStore(
  subscribe,
  () => clientStore.value,
  () => Math.random(), // Non-deterministic — WRONG
);

// GOOD: Stable server-side default
useSyncExternalStore(
  subscribe,
  () => clientStore.value,
  () => DEFAULT_STORE_VALUE, // Constant or derived from server-passed data
);

// Pattern: inject server state via context
function StoreProvider({ serverState, children }: { serverState: State; children: React.ReactNode }) {
  const storeRef = useRef(createStore(serverState));

  return (
    <StoreContext.Provider value={storeRef.current}>
      {children}
    </StoreContext.Provider>
  );
}

function useStore<T>(selector: (state: State) => T): T {
  const store = useContext(StoreContext);
  return useSyncExternalStore(
    store.subscribe,
    () => selector(store.getState()),
    () => selector(serverStateFromContext), // matches server render
  );
}
```

## Integrating Third-Party Stores

### Redux Integration (react-redux v8+)

```typescript
// react-redux's useSelector is built on useSyncExternalStore internally
// redux-toolkit's store is compatible out of the box

import { useSelector, useDispatch } from 'react-redux';

function Counter() {
  // Internally: useSyncExternalStore(store.subscribe, () => store.getState().count)
  const count = useSelector(state => state.count);
  const dispatch = useDispatch();

  return (
    <button onClick={() => dispatch(increment())}>
      {count}
    </button>
  );
}
```

### Building a Custom Store

```typescript
// A minimal Zustand-like store compatible with useSyncExternalStore

type Listener = () => void;
type SetState<T> = (partial: Partial<T> | ((state: T) => Partial<T>)) => void;
type StoreCreator<T> = (set: SetState<T>, get: () => T) => T;

function createStore<T extends object>(creator: StoreCreator<T>) {
  let state: T;
  const listeners = new Set<Listener>();

  const set: SetState<T> = (partial) => {
    const nextState = typeof partial === 'function'
      ? { ...state, ...partial(state) }
      : { ...state, ...partial };
    if (!Object.is(state, nextState)) {
      state = nextState;
      listeners.forEach(l => l());
    }
  };

  const get = () => state;
  state = creator(set, get);

  // subscribe and getSnapshot are the useSyncExternalStore interface
  const subscribe = (listener: Listener) => {
    listeners.add(listener);
    return () => listeners.delete(listener);
  };

  const getSnapshot = () => state; // Returns the same reference unless `set` was called

  // Selector-based hook
  function useStore(): T;
  function useStore<Selected>(selector: (state: T) => Selected): Selected;
  function useStore<Selected>(selector?: (state: T) => Selected) {
    return useSyncExternalStore(
      subscribe,
      selector ? () => selector(getSnapshot()) : getSnapshot,
    );
  }

  return { subscribe, getSnapshot, set, get, useStore };
}

// Usage
const counterStore = createStore((set) => ({
  count: 0,
  increment: () => set(s => ({ count: s.count + 1 })),
}));

function Counter() {
  const count = counterStore.useStore(s => s.count);
  const increment = counterStore.useStore(s => s.increment);
  return <button onClick={increment}>{count}</button>;
}
```

## Common Pitfalls

### Pitfall 1: getSnapshot Returns New Objects

```typescript
// This is the #1 mistake with useSyncExternalStore
// BAD: every call returns a new array, even if contents are same
useSyncExternalStore(
  subscribe,
  () => Object.values(store.entities), // New array reference every time
);

// React uses Object.is() to compare snapshots.
// A new array !== old array → React re-renders infinitely.

// GOOD: memoize the derived snapshot
let prevEntities: Entity[] | null = null;
let prevEntityMap: Record<string, Entity> | null = null;

useSyncExternalStore(
  subscribe,
  () => {
    if (store.entities === prevEntityMap) return prevEntities!;
    prevEntityMap = store.entities;
    prevEntities = Object.values(store.entities);
    return prevEntities;
  },
);
```

### Pitfall 2: Using useEffect Instead for Store Subscriptions

```typescript
// BAD: The useEffect pattern is unsafe in concurrent mode
function useUnsafeStore<T>(store: Store<T>): T {
  const [state, setState] = useState(store.getState());
  useEffect(() => {
    return store.subscribe(() => setState(store.getState()));
  }, [store]);
  return state;
}
// Problems:
// 1. Initial render uses useState initial value — may not match store
// 2. Between mount and effect running, store could change (missed update)
// 3. Tearing is possible in concurrent mode

// GOOD: useSyncExternalStore handles all these cases
function useSafeStore<T>(store: Store<T>): T {
  return useSyncExternalStore(
    store.subscribe.bind(store),
    store.getState.bind(store),
  );
}
```

### Pitfall 3: Subscribing to Parts of Global State Separately

```typescript
// BAD: Two separate subscriptions to the same store
// Can tear relative to each other in concurrent mode
function UserCard({ userId }: { userId: string }) {
  const name = useSyncExternalStore(subscribe, () => userStore.users[userId]?.name);
  const email = useSyncExternalStore(subscribe, () => userStore.users[userId]?.email);
  // name and email might come from different store versions!
  return <div>{name} - {email}</div>;
}

// GOOD: Single snapshot captures related data together
function UserCard({ userId }: { userId: string }) {
  const user = useSyncExternalStore(
    subscribe,
    () => userStore.users[userId] ?? null,
  );
  return user ? <div>{user.name} - {user.email}</div> : null;
}
```

### Pitfall 4: Forgetting getServerSnapshot for SSR

```typescript
// If getServerSnapshot is missing and the component is server-rendered,
// React throws: "Missing getServerSnapshot"

// This can happen with dynamic imports that are still SSR'd (Next.js App Router)
useSyncExternalStore(
  subscribe,
  getSnapshot,
  // Missing! Will throw on server/hydration
);

// Fix: always provide a stable server snapshot
useSyncExternalStore(
  subscribe,
  getSnapshot,
  () => defaultState, // Server-safe fallback
);
```

## Interview Questions

### Question 1: What is tearing and why does concurrent mode enable it?

**Answer**: Tearing is when different parts of the UI show different, inconsistent values of the same external state within a single committed frame. In synchronous rendering (React 17 and earlier), a render pass is never interrupted — the entire tree is rendered with one snapshot of all state. In concurrent mode, React can pause a render between components to yield to higher-priority work (like user input). If an external store updates during that pause, resumed components read the new value while earlier components already read the old one. The committed DOM contains both, creating a visible inconsistency.

### Question 2: How does useSyncExternalStore prevent tearing?

**Answer**: It uses two mechanisms. First, during a concurrent render, React tracks every store snapshot read via `useSyncExternalStore`. After rendering the tree (but before committing), React calls `getSnapshot` again and compares the result with `Object.is`. If any snapshot has changed, React aborts the entire render and re-renders synchronously — ensuring all components see the same updated snapshot. Second, when the subscribe callback fires after mount, `useSyncExternalStore` uses `forceStoreRerender` with `SyncLane`, which schedules a synchronous (non-interruptible) re-render, guaranteeing the whole tree updates together.

### Question 3: Why must getSnapshot return a stable reference when the store hasn't changed?

**Answer**: React uses `Object.is` (strict reference equality) to decide whether a store has changed and whether a re-render is needed. If `getSnapshot` returns a new object/array/function on every call even when the underlying data hasn't changed, `Object.is` always returns false, and React thinks the store is continuously updating, causing infinite re-renders. The contract is: if the store's relevant state hasn't changed since the last call, return the exact same reference.

### Question 4: What is the difference between useSyncExternalStore and useEffect + useState for store subscriptions?

**Answer**: The `useEffect + useState` pattern has three problems in concurrent mode: (1) The initial `useState` value is captured at render time and may already be stale by the time the effect runs. (2) There is a window between render and effect execution where store updates are missed. (3) The `setState` call from the subscription callback goes through the normal scheduler, which means React may render the component with different store values across a concurrent render pass. `useSyncExternalStore` addresses all three: it reads `getSnapshot` during render (no stale initial value), its subscription is set up before any yield points matter, and store updates always trigger synchronous re-renders.

### Question 5: When would you use useSyncExternalStore in a React application?

**Answer**: Any time you need to read from a state store that lives outside React's own state management (useState/useReducer/Context). Common cases include: Redux stores (react-redux uses it internally since v8), Zustand, MobX, browser APIs like `window.matchMedia` or `navigator.onLine`, WebSocket connection state, URL search params, and any custom pub/sub event system. You would NOT use it for state that is already managed by React primitives — useState, useReducer, and Context are all safe from tearing by definition since React controls their update cycle.

## Key Takeaways

1. **Tearing is a concurrent-mode problem**: Synchronous renders are immune; interruptible concurrent renders can observe store changes mid-tree

2. **useSyncExternalStore = subscribe + getSnapshot + getServerSnapshot**: the three-function protocol that safely bridges external stores to React's concurrent scheduler

3. **getSnapshot stability is critical**: must return the same reference when store hasn't changed — use `Object.is` as your mental model for what React checks

4. **Tearing detection happens post-render**: React re-checks all snapshots before committing; stale snapshots trigger a synchronous re-render of the full tree

5. **forceStoreRerender uses SyncLane**: store updates trigger synchronous, non-interruptible re-renders to guarantee consistency

6. **subscribe must be stable**: wrap in `useCallback` or define at module scope to avoid unsubscribe/resubscribe on every render

7. **getServerSnapshot is required for SSR**: must return the same value that the server would have returned during rendering

8. **One snapshot per logical unit**: don't split related data across multiple `useSyncExternalStore` calls — they can still observe different versions during the same render

## Resources

- [React docs: useSyncExternalStore](https://react.dev/reference/react/useSyncExternalStore)
- [React RFC: useMutableSource → useSyncExternalStore](https://github.com/reactjs/rfcs/blob/main/text/0213-suspense-in-react-18.md)
- [React source: ReactFiberHooks.ts — mountSyncExternalStore](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.ts)
- [use-sync-external-store shim (npm)](https://www.npmjs.com/package/use-sync-external-store)
- ["Tearing in Concurrent React" by Daishi Kato](https://github.com/dai-shi/will-this-react-global-state-work-in-concurrent-mode)
- [React 18 Working Group: External Store Tearing](https://github.com/reactwg/react-18/discussions/70)
