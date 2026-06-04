# Selector Pattern and Memoization

## Overview

Selectors are functions that derive data from the Redux (or any other) store. They act as the query layer between raw state and the UI. Without memoization, every `useSelector` call re-computes derived values on every render, even when inputs haven't changed — defeating the purpose of state management. `reselect`'s `createSelector` solves this by caching the last result and only recomputing when input selectors return new references. This guide covers the full selector lifecycle, composition patterns, parameterized selectors, and common pitfalls.

## What Is a Selector?

```
Store state (raw, normalized)
        │
        ▼
  Input Selectors  ◄──── cheap, direct reads from state
  (s) => s.posts.ids
  (s) => s.posts.entities
        │
        ▼
  Output Selector  ◄──── expensive derivation (filter, sort, map, join)
  (ids, entities) => ids.map(id => entities[id])
        │
        ▼
  Memoized Result  ◄──── cached; returned as-is if inputs unchanged
        │
        ▼
   Component       ◄──── only re-renders when result reference changes
```

## Basic Selectors (Without Memoization)

```typescript
import { RootState } from './store';

// Plain selector — fine for trivial reads
export const selectUserId = (state: RootState) => state.auth.userId;
export const selectUserEntities = (state: RootState) => state.users.entities;

// ❌ Problem: inline derivation re-runs every render
function UserList() {
  // This creates a NEW array on every render even if users didn't change
  const activeUsers = useSelector((state) =>
    Object.values(state.users.entities).filter((u) => u?.isActive)
  );
  // React sees a new array reference → re-renders children unnecessarily
}
```

## createSelector: The Core API

```typescript
import { createSelector } from '@reduxjs/toolkit'; // re-exports from reselect
// or
import { createSelector } from 'reselect';

// Signature:
// createSelector(
//   ...inputSelectors,  (last arg is the result function)
//   resultFunction
// )

const selectUsers    = (state: RootState) => state.users.entities;
const selectUserIds  = (state: RootState) => state.users.ids;

// Memoized: recomputes only when selectUsers or selectUserIds return new refs
export const selectAllUsers = createSelector(
  [selectUsers, selectUserIds],
  (entities, ids) =>
    ids.map((id) => entities[id]).filter((u): u is User => Boolean(u))
);

// The result function runs iff at least one input selector returns a !== value
```

### How Memoization Works Internally

```typescript
// Simplified reselect internals
function createSelector(...funcs) {
  const inputSelectors = funcs.slice(0, -1);
  const resultFn = funcs[funcs.length - 1];

  let lastInputs: unknown[] = [];
  let lastResult: unknown;

  return function(state: unknown) {
    const inputs = inputSelectors.map((fn) => fn(state));

    // Shallow equality check on each input
    const inputsChanged = inputs.some((input, i) => input !== lastInputs[i]);

    if (!inputsChanged) {
      return lastResult;  // ✅ Cache hit — no recomputation
    }

    lastInputs = inputs;
    lastResult = resultFn(...inputs);  // 💡 Recompute
    return lastResult;
  };
}
```

## Selector Composition

Build complex selectors from simple building blocks:

```typescript
// --- Base layer: raw reads ---
const selectPostEntities = (state: RootState) => state.posts.entities;
const selectPostIds      = (state: RootState) => state.posts.ids;
const selectUserEntities = (state: RootState) => state.users.entities;
const selectTagEntities  = (state: RootState) => state.tags.entities;
const selectFilter       = (state: RootState) => state.ui.postFilter;

// --- Mid layer: derived collections ---
export const selectAllPosts = createSelector(
  [selectPostEntities, selectPostIds],
  (entities, ids) => ids.map((id) => entities[id]).filter(Boolean) as Post[]
);

export const selectActivePosts = createSelector(
  [selectAllPosts],
  (posts) => posts.filter((p) => p.status === 'published')
);

// --- Composed layer: enriched with relationships ---
export const selectPostsWithAuthor = createSelector(
  [selectActivePosts, selectUserEntities],
  (posts, users) =>
    posts.map((post) => ({
      ...post,
      author: users[post.authorId],
    }))
);

// --- Application layer: filtered for current UI state ---
export const selectFilteredPosts = createSelector(
  [selectPostsWithAuthor, selectFilter],
  (posts, filter) => {
    if (!filter.tag) return posts;
    return posts.filter((p) => p.tagIds.includes(filter.tag!));
  }
);
```

## Parameterized Selectors (Selector Factories)

A selector takes state as its only argument by convention. To pass additional parameters (like an ID), use a factory function:

```typescript
// ✅ Factory pattern: create a new selector per unique parameter
export const makeSelectPostById = (postId: string) =>
  createSelector(
    [selectPostEntities, selectUserEntities, selectTagEntities],
    (posts, users, tags) => {
      const post = posts[postId];
      if (!post) return null;
      return {
        ...post,
        author: users[post.authorId],
        tags: post.tagIds.map((id) => tags[id]).filter(Boolean),
      };
    }
  );

// Usage in component — memoize the factory call so the selector
// instance remains stable across renders
function PostDetail({ postId }: { postId: string }) {
  const selectPost = useMemo(() => makeSelectPostById(postId), [postId]);
  const post = useSelector(selectPost);
  // ...
}
```

### RTK's createSelectorCreator and Memoization Caching

```typescript
import { createSelectorCreator, weakMapMemoize, lruMemoize } from 'reselect';

// Default memoize: cache size 1 — only last call remembered
// Problem: two instances of PostDetail with different postIds
// will cause the selector to miss cache continuously

// ✅ Solution 1: selector factory (each instance has its own cache)
// ✅ Solution 2: increase cache size with lruMemoize
const createLRUSelector = createSelectorCreator(lruMemoize, {
  maxSize: 50,  // caches last 50 unique argument combinations
});

// ✅ Solution 3: RTK 2.x weakMapMemoize — unlimited cache keyed by object identity
const createWeakMapSelector = createSelectorCreator(weakMapMemoize);

export const selectPostByIdCached = createWeakMapSelector(
  [
    selectPostEntities,
    selectUserEntities,
    (_: RootState, postId: string) => postId,
  ],
  (posts, users, postId) => {
    const post = posts[postId];
    if (!post) return null;
    return { ...post, author: users[post.authorId] };
  }
);

// Usage: pass postId as second argument
const post = useSelector((state) => selectPostByIdCached(state, postId));
```

## Input Selector Performance

```typescript
// ❌ Bad: input selector that creates new reference each call
const selectActiveUserIds = createSelector(
  [(state: RootState) => Object.values(state.users.entities)], // new array always!
  (users) => users.filter((u): u is User => u?.isActive === true).map((u) => u.id)
);
// The inner createSelector will ALWAYS re-run because its input always changes

// ✅ Good: stable input selectors
const selectUserEntitiesStable = (state: RootState) => state.users.entities;
// state.users.entities is the same reference unless something in users slice changed

const selectActiveUserIds = createSelector(
  [selectUserEntitiesStable],
  (entities) =>
    Object.values(entities)
      .filter((u): u is User => u?.isActive === true)
      .map((u) => u.id)
);
```

## When Selectors Re-Run

```
Trigger                          | Selector re-runs?
---------------------------------|------------------
Unrelated slice updates          | No (input unchanged)
Same slice updates, same values  | No (shallow equality passes)
Slice updates, inputs change     | Yes — result function called
New object with same shape       | Yes — !== by reference
Primitive value unchanged        | No — === passes for primitives
```

```typescript
// Demonstrates when re-run happens
const selectCount = (state: RootState) => state.counter.value; // primitive

const selectDouble = createSelector(
  [selectCount],
  (count) => {
    console.log('recomputing double');
    return count * 2;
  }
);

// dispatch({ type: 'counter/increment' }) → count changes → recomputes ✓
// dispatch({ type: 'posts/add', ... })   → count unchanged → cache hit ✓
// dispatch({ type: 'counter/increment', amount: 0 }) → count unchanged → cache hit ✓
```

## Advanced Patterns

### Combining Selectors Across Slices

```typescript
// Dashboard selector aggregating data from multiple slices
export const selectDashboardStats = createSelector(
  [
    (state: RootState) => state.posts.ids.length,
    (state: RootState) => state.comments.ids.length,
    (state: RootState) => state.users.ids.length,
    (state: RootState) => state.analytics.pageViews,
  ],
  (postCount, commentCount, userCount, pageViews) => ({
    postCount,
    commentCount,
    userCount,
    pageViews,
    avgCommentsPerPost: postCount > 0 ? commentCount / postCount : 0,
  })
);
```

### Selector with Default Value

```typescript
const EMPTY_ARRAY: Post[] = []; // Stable reference to avoid breaking memo

export const selectPostsByAuthor = createSelector(
  [selectAllPosts, (_: RootState, authorId: string) => authorId],
  (posts, authorId): Post[] => {
    const result = posts.filter((p) => p.authorId === authorId);
    // Return stable empty array instead of [] literal to prevent re-renders
    return result.length === 0 ? EMPTY_ARRAY : result;
  }
);
```

### Debug Selector Performance

```typescript
import { createSelectorCreator, defaultMemoize } from 'reselect';

// Instrument memoize to log hits/misses
function debugMemoize<T extends (...args: any[]) => any>(fn: T): T {
  let calls = 0;
  let hits = 0;
  let lastArgs: unknown[];
  let lastResult: unknown;

  return function (...args: unknown[]) {
    calls++;
    const inputsChanged = args.some((arg, i) => arg !== lastArgs?.[i]);
    if (!inputsChanged && lastArgs !== undefined) {
      hits++;
      console.log(`[selector] cache hit (${hits}/${calls})`);
      return lastResult;
    }
    lastArgs = args;
    lastResult = fn(...args);
    console.log(`[selector] recomputed (hits: ${hits}/${calls})`);
    return lastResult;
  } as T;
}

const createDebugSelector = createSelectorCreator(debugMemoize);
```

## React Integration Patterns

```typescript
// Pattern 1: Inline parameterized selector (correct with useMemo)
function PostCard({ postId }: { postId: string }) {
  const selectPost = useMemo(() => makeSelectPostById(postId), [postId]);
  const post = useSelector(selectPost);
}

// Pattern 2: Curried selector (RTK 2.x style with second arg)
function PostCard({ postId }: { postId: string }) {
  const post = useSelector((state) => selectPostByIdCached(state, postId));
  // Works with weakMapMemoize — no useMemo needed
}

// Pattern 3: Reusable hook encapsulating selector logic
function usePost(postId: string) {
  const selectPost = useMemo(() => makeSelectPostById(postId), [postId]);
  const post = useSelector(selectPost);
  const dispatch = useDispatch();

  const update = useCallback(
    (changes: Partial<Post>) => dispatch(updatePost({ id: postId, changes })),
    [postId, dispatch]
  );

  return { post, update };
}
```

## Common Mistakes

### 1. Returning a New Object from Input Selector

```typescript
// ❌ Bad: new object on every call → output selector always reruns
const selectFilters = (state: RootState) => ({
  tag: state.ui.tagFilter,
  status: state.ui.statusFilter,
});

// ✅ Good: stable reference, or read primitives individually
const selectTagFilter    = (state: RootState) => state.ui.tagFilter;
const selectStatusFilter = (state: RootState) => state.ui.statusFilter;
```

### 2. Memoization Without createSelector

```typescript
// ❌ Bad: looks memoized but isn't
const selectUsers = (state: RootState) =>
  state.users.ids.map((id) => state.users.entities[id]); // new array every call

// ✅ Good: createSelector provides memoization
const selectUsers = createSelector(
  [(s: RootState) => s.users.entities, (s: RootState) => s.users.ids],
  (entities, ids) => ids.map((id) => entities[id]).filter(Boolean)
);
```

### 3. Sharing a Single Parameterized Selector Across Multiple Component Instances

```typescript
// ❌ Bad: one shared selector with cache size 1
export const selectPostById = createSelector(
  [selectPostEntities, (_: RootState, id: string) => id],
  (entities, id) => entities[id]
);

// Two components use different IDs → constant cache invalidation
// <PostCard postId="p1" />  → cache for p1
// <PostCard postId="p2" />  → cache busted, recompute for p2
// <PostCard postId="p1" />  → cache busted again!

// ✅ Good: factory or weakMapMemoize
```

### 4. Forgetting That Selectors Must Be Pure

```typescript
// ❌ Bad: side effect in selector
const selectSortedPosts = createSelector(
  [selectAllPosts],
  (posts) => {
    console.log('computing sorted posts'); // logs on every recompute — OK for debug
    return [...posts].sort((a, b) => b.createdAt.localeCompare(a.createdAt));
    // Note: always use spread before sort — don't mutate the input array
  }
);

// ❌ Really bad: mutation
const selectSortedPosts = createSelector(
  [selectAllPosts],
  (posts) => posts.sort(/* ... */)  // mutates the cached input!
);
```

### 5. Using Selectors Outside of React Without resetting memoization

```typescript
// In tests: each test should see fresh state
// Memoization might return stale result if state reference doesn't change

// ✅ Good: reset memoize before critical test assertions
selectAllPosts.resetRecomputations(); // reselect API
selectAllPosts.clearCache();          // RTK weakMapMemoize API
```

## Interview Questions

### 1. What is the difference between an input selector and an output selector in reselect?

**Answer**: Input selectors are simple functions that extract raw slices of state. They are called every time the combined selector is invoked, and their results are compared via `===` to decide whether the output selector needs to recompute. The output selector (the last argument to `createSelector`) is the expensive derivation function — it only runs when at least one input selector returns a value that is `!==` its previous return. Input selectors should be trivial reads; heavy computation belongs in the output selector where it is memoized.

### 2. Why does a selector factory solve the parameterized selector cache problem?

**Answer**: `createSelector` maintains a cache of size 1 by default — it remembers only the most recent set of inputs and result. If two component instances call the same parameterized selector with different IDs, they continuously bust each other's cache, meaning the result function recomputes on every call. A factory (`makeSelectById(id) => createSelector(...)`) creates one selector instance per ID, giving each its own independent cache. RTK 2.x's `weakMapMemoize` offers an alternative: unlimited cache keyed by object identity, so a single selector handles all IDs without cache collisions.

### 3. How does reselect's memoization differ from React.useMemo?

**Answer**: Both cache the result of an expensive computation and recompute when dependencies change, but they operate at different layers. `React.useMemo` caches inside a component instance and runs during React's render phase — its dependencies are values you explicitly list. `createSelector` caches at the module level (or selector instance level) — it's independent of React and can be called outside components, in thunks, in SSR, or in tests. Multiple components can share one selector instance and all benefit from the same cached result. The memoization strategy is also different: React uses identity equality for all deps, while reselect uses identity equality on each input selector's return value.

### 4. When should you NOT use createSelector?

**Answer**: When the derivation is already O(1) or produces a stable reference. Examples: reading a single primitive (`state.auth.userId`), returning a pre-existing entity object (`state.users.entities[id]`), or reading a boolean flag. `createSelector` has overhead: it calls every input selector on every run and does a shallow comparison. For trivial reads that overhead isn't justified. Rule of thumb: use `createSelector` when the result function involves iteration (`.map`, `.filter`, `.reduce`), sorting, or any operation that creates a new reference.

### 5. What happens to selector memoization during server-side rendering?

**Answer**: Module-level selectors created with `createSelector` persist across requests in SSR — their cache is shared between concurrent requests. If request A calls a selector with state-A and caches the result, request B calling the same selector with different state-B will correctly get a cache miss and recompute (because input selectors return different values). However, parameterized selectors with a single cache entry create a race condition: request A's cached inputs can be overwritten by request B before A reads the result. The solution for SSR is to use `createSelectorCreator` with a per-request memoize factory, or use the factory pattern so each request gets its own selector instance.
