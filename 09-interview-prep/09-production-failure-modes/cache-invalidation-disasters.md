# Production Failure: Cache Invalidation Disasters

## Overview

"There are only two hard things in computer science: cache invalidation and naming things." Every frontend engineer laughs at this — until they ship a bug at 2am where half their users see stale data after a critical update. These are real failure patterns, how they manifest, and how to design your way out of them.

---

## Failure 1: Normalized State + Partial Mutation

### The Setup

You're using Redux Toolkit with a normalized entity store. Orders reference products. Products have prices.

```typescript
// State shape
{
  orders: {
    ids: ['o1', 'o2'],
    entities: {
      o1: { id: 'o1', productIds: ['p1', 'p2'], total: 150 },
    }
  },
  products: {
    ids: ['p1', 'p2'],
    entities: {
      p1: { id: 'p1', name: 'Widget', price: 50 },
      p2: { id: 'p2', name: 'Gadget', price: 100 },
    }
  }
}
```

### The Bug

A price change comes in via WebSocket for `p1`. You update the product entity:

```typescript
productsAdapter.updateOne(state, { id: 'p1', changes: { price: 75 } });
```

The order total `o1.total = 150` is now **stale** — it reflects the old price. The order detail page shows `Total: $150` but the line items add up to `$175`.

### Why It's Hard to Catch

- Unit tests for the order component pass (they mock the total)
- Unit tests for the product slice pass (price updated correctly)
- The bug only surfaces when both slices are rendered together
- It doesn't throw — it silently shows wrong numbers

### The Fix Patterns

**Option A: Derived state — never store computed values**
```typescript
// Don't store `total` in the order entity
// Compute it from line items in a selector
const selectOrderTotal = createSelector(
  (state, orderId) => state.orders.entities[orderId],
  (state) => state.products.entities,
  (order, products) =>
    order.productIds.reduce((sum, pid) => sum + products[pid].price, 0)
);
```

**Option B: Invalidate dependents on mutation**
```typescript
// When a product price changes, mark all orders containing it as stale
// and re-fetch them
extraReducers: (builder) => {
  builder.addCase(productPriceUpdated, (state, action) => {
    const affectedOrderIds = Object.values(state.entities)
      .filter(o => o.productIds.includes(action.payload.productId))
      .map(o => o.id);
    affectedOrderIds.forEach(id => {
      state.entities[id].stale = true;
    });
  });
}
```

**Option C: Server-authoritative totals — don't replicate computed state**
The cleanest fix: remove `total` from the client model entirely. Fetch the fresh total when displaying it.

---

## Failure 2: Optimistic Update + Rollback Gone Wrong

### The Setup

A todo app with optimistic updates. User checks a todo — UI updates instantly, API call fires in background.

```typescript
onMutate: async (todoId) => {
  await queryClient.cancelQueries(['todos']);
  const snapshot = queryClient.getQueryData(['todos']);
  queryClient.setQueryData(['todos'], (old) =>
    old.map(t => t.id === todoId ? { ...t, done: true } : t)
  );
  return { snapshot };
},
onError: (err, todoId, context) => {
  queryClient.setQueryData(['todos'], context.snapshot); // rollback
},
```

### The Bug

User checks todo #1 (optimistic update: done=true). Before the response, user checks todo #2 (optimistic update: done=true). The todo #1 request **fails** and rolls back — but the rollback snapshot was captured *before* todo #2's optimistic update, so it also undoes todo #2's change.

### Why It's Hard to Catch

- Works perfectly when mutations are sequential
- Only breaks under rapid concurrent mutations
- QA can't reproduce — they click one thing at a time

### The Fix

```typescript
// Don't snapshot the whole list — rollback only the specific item
onMutate: async (todoId) => {
  const todo = queryClient.getQueryData(['todo', todoId]);
  queryClient.setQueryData(['todo', todoId], (old) => ({ ...old, done: true }));
  return { todoId, previousDone: todo.done };
},
onError: (err, todoId, context) => {
  // Roll back only this item, not the whole list
  queryClient.setQueryData(['todo', context.todoId], (old) => ({
    ...old,
    done: context.previousDone,
  }));
},
```

Or use item-level cache keys rather than a single list key.

---

## Failure 3: Cache Stampede on Invalidation

### The Setup

You have 10,000 concurrent users. Your CDN caches product pages for 5 minutes. At 10:00:00, every cached entry expires simultaneously (they all had the same TTL start time during a deployment).

### What Happens

At 10:00:01, all 10,000 users hit the origin server simultaneously. The server is not designed for 10,000 requests/second. It falls over. Every new request also misses the (now-rebuilding) cache and hits the dying server. A cascade failure brings down the product catalog for 15 minutes.

### The Fix Patterns

**Staggered TTLs (Jitter)**
```typescript
// Instead of fixed TTL, add random jitter
const TTL_BASE = 300; // 5 minutes
const TTL_JITTER = 60; // ±1 minute random
const ttl = TTL_BASE + Math.random() * TTL_JITTER;

// Cache entries expire at different times, spreading load
res.setHeader('Cache-Control', `public, max-age=${ttl}`);
```

**Stale-While-Revalidate**
```
Cache-Control: public, max-age=300, stale-while-revalidate=60
```
Users get stale content for up to 60 seconds while the cache revalidates in the background — origin only receives one request, not 10,000.

**Cache Warming Before Expiry**
A background job re-generates the cache entry at 80% of its TTL, before it actually expires — origin never sees the stampede.

---

## Failure 4: Browser Cache Poisoning via Service Worker

### The Setup

You ship a Service Worker that caches API responses aggressively.

```javascript
// v1 service worker — too aggressive
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request) || fetch(event.request).then(res => {
      caches.open('api-v1').then(cache => cache.put(event.request, res.clone()));
      return res;
    })
  );
});
```

### The Bug

A user fetches `/api/user/profile` while the backend has a transient bug returning `{ role: 'admin' }` instead of `{ role: 'user' }`. This response is cached in the Service Worker. The bug is fixed on the server immediately — but the user's browser keeps serving the poisoned cached response for the cache's entire lifetime.

The user has elevated permissions in the UI with no way to recover except clearing the cache manually.

### The Fix

```javascript
// Never cache authenticated or authorization-related API responses in SW
const NEVER_CACHE = ['/api/user/', '/api/auth/', '/api/permissions/'];

self.addEventListener('fetch', (event) => {
  const url = new URL(event.request.url);
  if (NEVER_CACHE.some(path => url.pathname.startsWith(path))) {
    return; // let browser handle normally — no SW caching
  }
  // ... cache only static assets and public API responses
});
```

Also: always include cache-busting version in SW registration.

---

## Failure 5: React Query Stale State After Login/Logout

### The Setup

User A logs in, browses their data (cached in React Query). User A logs out. User B logs in on the same device. React Query still has User A's data in its cache — it serves it immediately to User B before the background refetch completes.

### What Happens

User B sees User A's private data for ~200ms. If `staleTime` is long, they may see it indefinitely.

### The Fix

```typescript
// On logout: destroy the entire query client, don't just clear one query
function logout() {
  await api.logout();
  queryClient.clear(); // ← nuke all cached data
  // Or: recreate the client entirely
  setQueryClient(new QueryClient());
}

// Also: user-scope all query keys so they can't cross-contaminate
const { data } = useQuery({
  queryKey: ['user', userId, 'orders'], // userId scopes the cache
  queryFn: () => fetchOrders(userId),
});
```

---

## Interview Application

When an interviewer asks "What can go wrong with your caching approach?", run through this mental checklist:

1. **Staleness**: When data changes, which cache layers still hold the old version? (Browser cache, Service Worker, CDN, in-memory state, normalized store)
2. **Cascades**: Who depends on this data? If it goes stale, what else is now wrong?
3. **Rollbacks**: If an optimistic update fails, can concurrent mutations corrupt the rollback?
4. **Stampedes**: What happens when the cache expires under high load?
5. **Scope**: Is cached data properly scoped to the user/session? Can it leak across auth boundaries?

**The answer interviewers want to hear:**
> "I'd use stale-while-revalidate with per-user cache keys, scope my optimistic updates to individual items rather than lists, and on logout nuke the entire client-side cache rather than trying to selectively invalidate."
