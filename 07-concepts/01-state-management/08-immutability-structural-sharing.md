# Immutability and Structural Sharing

## The Idea

**In plain English:** Immutability means that once a piece of data is created, you never change it directly — instead, you make a fresh copy with your changes applied. Structural sharing is the smart trick that makes this efficient: when you create that fresh copy, unchanged parts are reused as-is rather than duplicated, so you're not wasting memory.

**Real-world analogy:** Imagine a shared Google Doc that a team of writers works on. Instead of letting everyone scribble over the same document at the same time (causing chaos), the rule is: to make any change, you first duplicate just the page you need to edit, make your change on that copy, and clip it back into the binder — all other pages stay as the originals that everyone can still read.

- The original binder of pages = the previous state of your app's data
- Duplicating only the page you need to edit = creating a new object only for the part that changed (structural sharing)
- The unchanged original pages everyone still references = the shared, untouched parts of the state tree
- The new binder assembled from your edited page + the original pages = the new state object returned after an update

---

## Overview

Immutability is a foundational constraint in modern frontend state management: state is never modified in place; instead, updates produce new objects. This enables reference equality checks (`prevState === nextState`) as a fast proxy for "has anything changed?" — which is the backbone of React's rendering optimization, Redux change detection, and selector memoization. But naive immutability (deep cloning every update) is O(n) in memory. Structural sharing solves this: only the changed nodes in the state tree are copied; unchanged nodes share the same references. This guide covers why immutability matters, how structural sharing works, Immer's approach, and persistent data structures.

## Why Immutability Matters

```text
Mutable State — change detection requires deep comparison:

prevState = { users: [...], settings: { theme: 'dark' } }
             ↓  mutate in place
nextState = prevState  // same reference!

prevState === nextState  → true (even though data changed)
React: "nothing changed" → skips re-render → WRONG

Immutable State — change detection is O(1):

prevState = { users: oldArray, settings: { theme: 'dark' } }
             ↓  create new top-level object
nextState = { users: newArray, settings: prevState.settings } // settings SHARED

prevState === nextState         → false (new root object)
prevState.settings === nextState.settings → true (unchanged, not copied)
```

### Reference Equality in React

```typescript
// How React uses reference equality for optimization

// React.memo — shallow prop comparison
const UserCard = React.memo(function UserCard({ user }: { user: User }) {
  console.log('rendering', user.id);
  return <div>{user.name}</div>;
});

// With mutable state — mutation doesn't trigger re-render
function BadParent() {
  const [users, setUsers] = useState([{ id: 1, name: 'Alice' }]);

  function updateName() {
    users[0].name = 'Bob'; // ❌ mutating in place
    setUsers(users);       // same reference → React bails out → no re-render!
  }

  return <UserCard user={users[0]} />;
}

// With immutable updates — React correctly detects change
function GoodParent() {
  const [users, setUsers] = useState([{ id: 1, name: 'Alice' }]);

  function updateName() {
    setUsers([
      { ...users[0], name: 'Bob' }, // ✅ new object → new reference
      ...users.slice(1),
    ]);
  }

  return <UserCard user={users[0]} />;
}
```

## Structural Sharing

Structural sharing ensures that updating one part of the state tree does not copy unrelated parts:

```text
Initial State Tree:
         root
        /    \
     users   settings
     /   \       \
   u1    u2    theme:'dark'

Update u1.name:
         root*        ← new (references changed)
        /    \
    users*   settings  ← settings SHARED (not copied)
    /   \
  u1*   u2             ← u2 SHARED, u1* is new
```

```typescript
// Manual structural sharing with spread operator
interface AppState {
  users: User[];
  settings: Settings;
  ui: UIState;
}

// ✅ Structural sharing: only changed branch is new
function updateUserName(state: AppState, userId: string, name: string): AppState {
  return {
    ...state,                       // new root (but settings and ui are SHARED)
    users: state.users.map((u) =>   // new users array
      u.id === userId
        ? { ...u, name }            // new user object
        : u                         // other users SHARED (same reference)
    ),
  };
}

// Memory usage: 1 new root + 1 new users array + 1 new user object
// NOT: deep clone of entire state tree
```

## Immer: Write Mutable, Get Immutable

Immer lets you write code that *looks* mutable but produces structurally-shared immutable output:

```typescript
import { produce, enableMapSet } from 'immer';

enableMapSet(); // opt-in for Map/Set support

const state = {
  users: [
    { id: '1', name: 'Alice', scores: [10, 20, 30] },
    { id: '2', name: 'Bob', scores: [5, 15] },
  ],
  settings: { theme: 'dark' },
};

// Immer's produce: draft is a Proxy that tracks changes
const nextState = produce(state, (draft) => {
  // Write as if mutating — Immer creates structurally-shared output
  draft.users[0].name = 'Alicia';
  draft.users[0].scores.push(40);
  // draft.settings is NOT touched → shared reference in output
});

console.log(nextState === state);                    // false (new root)
console.log(nextState.settings === state.settings); // true  (shared)
console.log(nextState.users[1] === state.users[1]); // true  (Bob unchanged, shared)
console.log(nextState.users[0] === state.users[0]); // false (Alice changed, new obj)
```

### How Immer's Proxy Works

```typescript
// Simplified Immer internals
function produce<T>(base: T, recipe: (draft: T) => void): T {
  const modified = new Set<object>();
  const copies = new WeakMap<object, object>();

  function createProxy<O extends object>(obj: O): O {
    return new Proxy(obj, {
      get(target, key) {
        const value = Reflect.get(target, key);
        // Recursively proxy nested objects on access
        if (value && typeof value === 'object') {
          return createProxy(value);
        }
        return value;
      },
      set(target, key, value) {
        // First write: copy-on-write
        if (!copies.has(target)) {
          copies.set(target, Array.isArray(target) ? [...target] : { ...target });
          modified.add(target);
        }
        const copy = copies.get(target) as O;
        Reflect.set(copy, key, value);
        return true;
      },
    });
  }

  const draft = createProxy(base as object) as T;
  recipe(draft);

  // Reconstruct tree with copies where modified, originals elsewhere
  return finalize(base, copies, modified) as T;
}
```

### Immer with RTK

```typescript
import { createSlice, PayloadAction } from '@reduxjs/toolkit';
// RTK uses Immer internally — all reducers are wrapped with produce()

interface CartState {
  items: Array<{ id: string; quantity: number; price: number }>;
  coupon: string | null;
}

const cartSlice = createSlice({
  name: 'cart',
  initialState: { items: [], coupon: null } as CartState,
  reducers: {
    // Write mutably — RTK/Immer gives you structural sharing for free
    addItem(state, action: PayloadAction<{ id: string; price: number }>) {
      const existing = state.items.find((item) => item.id === action.payload.id);
      if (existing) {
        existing.quantity += 1; // ✅ looks like mutation, actually copy-on-write
      } else {
        state.items.push({ ...action.payload, quantity: 1 });
      }
    },

    removeItem(state, action: PayloadAction<string>) {
      const index = state.items.findIndex((item) => item.id === action.payload);
      if (index !== -1) state.items.splice(index, 1);
    },

    applyCoupon(state, action: PayloadAction<string>) {
      state.coupon = action.payload;
      // state.items is NOT touched → structural sharing: same reference
    },
  },
});
```

### Immer Pitfalls

```typescript
// ❌ Bad: returning a new value AND mutating the draft
const nextState = produce(state, (draft) => {
  draft.count = 5;    // mutation
  return { count: 6 }; // also returning — ERROR: can't do both
});

// ✅ Good: either mutate OR return, not both
const nextState = produce(state, (draft) => {
  draft.count = 5; // mutation only
});
// OR
const nextState = produce(state, (_draft) => {
  return { ...state, count: 5 }; // return only (replacement)
});

// ❌ Bad: storing draft reference outside produce
let savedDraft: typeof state;
produce(state, (draft) => {
  savedDraft = draft; // proxy is revoked after produce returns
});
savedDraft.count; // Error: proxy revoked

// ❌ Bad: using draft in async context
produce(state, async (draft) => {
  const data = await fetch('/api/data').then((r) => r.json());
  draft.data = data; // doesn't work — produce is synchronous
});
// ✅ Good: fetch outside, then produce synchronously
const data = await fetch('/api/data').then((r) => r.json());
const nextState = produce(state, (draft) => { draft.data = data; });
```

## Persistent Data Structures

Beyond structural sharing with plain objects, persistent data structures (from Clojure/Haskell) provide O(log n) update and lookup across all operations:

```text
Persistent Hash Array Mapped Trie (HAMT) — used by Immutable.js:

     root (32-way branching)
    / | \
   A  B  C        ← internal nodes shared
      |
     B'            ← only modified branch is new on update
      |
   [b1, b2, b3*]   ← b3 updated, b1 and b2 shared
```

### Immutable.js

```typescript
import { Map, List, fromJS, is } from 'immutable';

// Immutable.js Map — persistent, structural sharing built in
const state = Map({
  users: List([
    Map({ id: '1', name: 'Alice' }),
    Map({ id: '2', name: 'Bob' }),
  ]),
  settings: Map({ theme: 'dark' }),
});

// Update — O(log n), structural sharing automatic
const nextState = state.setIn(['users', 0, 'name'], 'Alicia');

// Unchanged paths share references
state.get('settings') === nextState.get('settings'); // true

// Value equality (deep) — O(n) but rarely needed
is(state, nextState); // false

// ⚠️ Gotcha: Immutable.js objects are NOT plain JS objects
// You can't spread, use Object.keys, or pass to libraries expecting plain objects
nextState.get('users').toJS(); // convert back — expensive O(n)
```

### Why Most Teams Use Plain Objects + Immer Instead

```typescript
// Immutable.js tradeoffs:
// ✓ Persistent data structures with formal guarantees
// ✓ Excellent for large state trees with frequent partial updates
// ✗ Learning curve — different API (.get, .set, .setIn vs dot access)
// ✗ Interop friction — must .toJS() for most libraries
// ✗ Large bundle size
// ✗ Lost TypeScript inference depth

// Plain objects + Immer:
// ✓ Native JS syntax, TypeScript inference works naturally
// ✓ Zero interop cost — plain objects everywhere
// ✓ Structural sharing for typical UI state sizes is sufficient
// ✗ O(n) in worst case for very large arrays
// ✗ No value equality (must use === or deepEqual manually)
```

## Structural Sharing and React Performance

```typescript
// Understanding when React re-renders with structural sharing

interface Post { id: string; title: string; likes: number; }
interface State { posts: Post[]; activePostId: string; }

// Store update: only activePostId changed
const prev: State = { posts: bigPostArray, activePostId: 'p1' };
const next: State = { ...prev, activePostId: 'p2' };

// prev.posts === next.posts  → true (structural sharing)
// Components that only receive posts will not re-render:

const PostList = React.memo(({ posts }: { posts: Post[] }) => {
  console.log('PostList re-rendered'); // NOT called
  return <ul>{posts.map((p) => <li key={p.id}>{p.title}</li>)}</ul>;
});

// Only components receiving activePostId will re-render
```

```typescript
// Anti-pattern: breaking structural sharing unnecessarily
const cartSlice = createSlice({
  name: 'cart',
  initialState: { items: [], metadata: { updatedAt: '' } },
  reducers: {
    // ❌ Bad: always create new items array even when only metadata changes
    touchCart(state) {
      state.metadata.updatedAt = new Date().toISOString();
      state.items = [...state.items]; // unnecessary copy — breaks sharing
    },

    // ✅ Good: only mutate what actually changed
    touchCartGood(state) {
      state.metadata.updatedAt = new Date().toISOString();
      // state.items is untouched → Immer shares it → components don't re-render
    },
  },
});
```

## Immutability in useReducer

```typescript
// Same principles apply to local state with useReducer
type Action =
  | { type: 'ADD_TODO'; text: string }
  | { type: 'TOGGLE_TODO'; id: string }
  | { type: 'SET_FILTER'; filter: string };

interface TodoState {
  todos: Array<{ id: string; text: string; done: boolean }>;
  filter: string;
}

function todoReducer(state: TodoState, action: Action): TodoState {
  switch (action.type) {
    case 'ADD_TODO':
      return {
        ...state,
        todos: [...state.todos, { id: crypto.randomUUID(), text: action.text, done: false }],
        // state.filter is shared — no copy
      };

    case 'TOGGLE_TODO':
      return {
        ...state,
        todos: state.todos.map((t) =>
          t.id === action.id
            ? { ...t, done: !t.done }  // new object for changed todo
            : t                         // other todos shared
        ),
      };

    case 'SET_FILTER':
      return {
        ...state,       // state.todos shared
        filter: action.filter,
      };

    default:
      return state;     // same reference — no re-render
  }
}
```

## Common Mistakes

### 1. Mutating State Directly

```typescript
// ❌ Bad
function reducer(state, action) {
  state.user.name = action.name; // mutation
  return state;                  // same reference → React won't re-render
}

// ✅ Good
function reducer(state, action) {
  return { ...state, user: { ...state.user, name: action.name } };
}
```

### 2. Array Methods That Mutate vs Those That Don't

```typescript
// ❌ Mutations — never use on state directly
state.items.push(newItem);
state.items.pop();
state.items.splice(index, 1);
state.items.sort(compareFn);
state.items.reverse();

// ✅ Immutable alternatives
[...state.items, newItem]             // push
state.items.slice(0, -1)             // pop
state.items.filter((_, i) => i !== index) // splice remove
[...state.items].sort(compareFn)     // sort
[...state.items].reverse()           // reverse
```

### 3. Spreading Only One Level Deep

```typescript
// ❌ Bad: shallow spread doesn't protect nested objects
const next = { ...state };
next.user.name = 'New Name'; // still mutates original state.user!

// ✅ Good: spread every level you modify
const next = {
  ...state,
  user: { ...state.user, name: 'New Name' },
};
```

### 4. Using Object.assign for Nested Updates

```typescript
// ❌ Bad: Object.assign is shallow
const next = Object.assign({}, state, { user: { name: 'New' } });
// state.user is replaced entirely — loses other user fields!

// ✅ Good: explicit deep spread or Immer
const next = { ...state, user: { ...state.user, name: 'New' } };
```

## Interview Questions

### 1. Why does React use reference equality instead of deep equality for re-render decisions?

**Answer**: Deep equality (recursive comparison) is O(n) — proportional to the size of the data structure. For large state trees with frequent updates, doing a deep comparison on every render would be prohibitively expensive. Reference equality is O(1) — just a pointer comparison. Immutability makes this safe: if you haven't created a new reference, nothing has changed (the invariant). The trade-off is that you must always create new objects for updated state — mutation will cause stale renders. Libraries like React.memo, useMemo, and Redux's useSelector all rely on this O(1) change detection.

### 2. Explain structural sharing and why it makes immutability practical

**Answer**: Structural sharing means that when you update part of a data structure, unchanged portions share the same memory as the previous version. Without structural sharing, every state update would require a full deep copy — O(n) memory and time. With structural sharing, an update to a single leaf node copies only the path from root to that leaf — O(log n) for tree-based structures, O(depth) for plain nested objects. In practice: updating `state.users[0].name` creates a new root object, a new users array, and a new user-0 object, but the rest of the users array elements and all other state slices keep their original references.

### 3. How does Immer achieve immutability while letting you write mutable-looking code?

**Answer**: Immer wraps the state in a Proxy (the "draft"). When you read a property, you get another proxy for that nested object. When you write a property, Immer performs copy-on-write: the first write to any object copies it (shallow), stores the copy, and redirects the write there. At the end of `produce`, Immer walks the tree: for every object that was copied, it replaces that node in the output tree with the copy; for untouched objects, it uses the original reference. The result is structurally-shared immutable output even though your recipe code looked mutable.

### 4. What is the difference between Immer and Immutable.js, and when would you choose each?

**Answer**: Immer operates on plain JavaScript objects and arrays. It uses proxies to intercept writes during a synchronous recipe function, producing structurally-shared plain objects as output. No special API, full TypeScript inference, zero interop cost. Immutable.js uses persistent data structures (HAMT tries) that guarantee O(log n) updates and lookups for arbitrarily large collections — you pay upfront with a different API (`.get('key')` instead of `.key`, `.toJS()` to convert back). Choose Immer when working with typical UI state (hundreds to thousands of entities) with native JS ergonomics. Choose Immutable.js when you have very large collections (millions of entries), need structural equality comparisons (`is(a, b)`), or are doing functional pipeline transforms on collections.

### 5. How does immutability interact with React.memo and useCallback?

**Answer**: React.memo performs a shallow comparison of props. If a parent re-renders but passes the same object references to a memoized child, the child skips rendering. This only works correctly if state is immutable — mutable updates would change data without changing references, causing memo to produce stale renders. `useCallback` memoizes a function reference to prevent child re-renders when a callback prop doesn't logically change. For useCallback to be effective, its dependency array entries must be stable references — which again depends on immutable state updates. The entire React optimization model is built on the assumption that new reference = changed data, old reference = unchanged data.
