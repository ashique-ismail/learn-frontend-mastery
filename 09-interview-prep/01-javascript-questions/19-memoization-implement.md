# Memoization: Implement from Scratch

## The Idea

**In plain English:** Memoization is a way to make a function faster by remembering the answers it already figured out — so if you ask it the same question twice, it just looks up the saved answer instead of doing all the work again. Think of it like a scratch pad where you write down results so you never have to recalculate them.

**Real-world analogy:** Imagine a student doing homework who keeps a cheat sheet of math problems they have already solved. The first time they see "12 x 13", they work it out and write "12 x 13 = 156" on the sheet. The next time that same problem appears, they just read the answer off the sheet instead of multiplying again.

- The student = the function
- The math problem = the input arguments
- The cheat sheet = the cache (stored results)

---

## Why This Comes Up in Interviews

Memoization tests your grasp of closures, cache invalidation strategy, memory management (WeakMap vs Map), async patterns, and algorithmic complexity. Senior-level questions push past the basic "cache results in an object" answer into TTL, LRU eviction, in-flight deduplication, and framework integration.

---

## 1. Definition

Memoization is an optimization technique that **caches the return value of a pure function keyed by its arguments**. On repeated calls with the same arguments, the cached result is returned directly — the function body is never re-executed.

Three preconditions for safe memoization:

1. **Pure function** — same inputs always produce same outputs, no side effects.
2. **Referential transparency** — return value depends only on arguments, not external mutable state.
3. **Expensive enough** — the cache lookup cost must be less than the computation cost (see section 12).

---

## 2. Basic Single-Argument Implementation

```js
function memoize(fn) {
  const cache = new Map();

  return function(arg) {
    if (cache.has(arg)) {
      return cache.get(arg);
    }
    const result = fn.call(this, arg);
    cache.set(arg, result);
    return result;
  };
}

// Usage
const expensiveSquare = memoize((n) => {
  console.log(`computing ${n}²`);
  return n * n;
});

expensiveSquare(4); // logs "computing 4²", returns 16
expensiveSquare(4); // cache hit — returns 16 silently
expensiveSquare(5); // logs "computing 5²", returns 25
```

**Why Map over a plain object?** Map correctly handles non-string keys (numbers, objects, null, NaN) without prototype pollution. `Object` coerces all keys to strings, so `obj[1]` and `obj["1"]` collide.

---

## 3. Multi-Argument: JSON.stringify as Cache Key

The simplest extension to multiple arguments serializes them into a single string key.

```js
function memoize(fn) {
  const cache = new Map();

  return function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) {
      return cache.get(key);
    }
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

const add = memoize((a, b) => a + b);
add(1, 2); // 3, cached under '[1,2]'
add(1, 2); // cache hit
```

### Limitations of JSON.stringify as a Key

| Problem | Example | Behavior |
|---|---|---|
| Circular references | `JSON.stringify({a: obj, b: obj})` where obj.self = obj | Throws `TypeError` |
| Functions as args | `memoize(fn)(x => x)` | Functions serialize to `undefined`, all function args become `null` or are dropped |
| Symbol keys | `{[Symbol()]: 1}` | Symbols are silently dropped from serialization |
| Argument order irrelevant to fn | `add(1,2)` vs `add(2,1)` if commutative | Two separate cache entries — but both correct, just wastes cache space |
| `undefined` vs missing | `JSON.stringify([undefined])` === `'[null]'` | `undefined` becomes `null` in JSON |
| Class instances | `new Date()` serializes to a string; two equal `Date` objects with same value but different references map to same key by coincidence, but custom objects serialize to `{}` |

For arguments that are primitives only, `JSON.stringify` is fine. For anything else, prefer the trie approach below.

---

## 4. Multi-Argument Trie with WeakMap/Map

Instead of serializing args to a string, build a trie (prefix tree) where each node in the path corresponds to one argument. Objects use WeakMap nodes (allowing GC), primitives use Map nodes.

```js
function memoize(fn) {
  // Root of the argument trie
  const root = makeTrie();

  return function(...args) {
    let node = root;

    for (const arg of args) {
      // Objects/functions: WeakMap so GC can reclaim them
      // Primitives: regular Map (can't WeakMap a number)
      const isObject = arg !== null && typeof arg === 'object' || typeof arg === 'function';
      const store = isObject ? node.weak : node.strong;

      if (!store.has(arg)) {
        store.set(arg, makeTrie());
      }
      node = store.get(arg);
    }

    if (!node.hasResult) {
      node.result = fn.apply(this, args);
      node.hasResult = true;
    }
    return node.result;
  };
}

function makeTrie() {
  return {
    weak: new WeakMap(),   // for object/function args
    strong: new Map(),     // for primitive args
    hasResult: false,
    result: undefined,
  };
}

// Usage
const mergeObj = memoize((a, b) => ({ ...a, ...b }));
const obj1 = { x: 1 };
const obj2 = { y: 2 };

mergeObj(obj1, obj2); // computed
mergeObj(obj1, obj2); // cache hit — same object references
```

**Key insight:** Two calls share a cache node only if they pass the exact same object references in the same positions. This is correct for pure functions that treat objects by identity (like React component props). If you need structural equality, you need deep comparison — but that often costs more than the computation itself.

---

## 5. WeakMap-Based Memoize for Single Object Argument

When the argument is a single object (e.g., a React props object), WeakMap alone is the cleanest solution. When the object is garbage collected, the cache entry is automatically removed — no memory leak.

```js
function memoizeWeak(fn) {
  const cache = new WeakMap();

  return function(obj) {
    if (cache.has(obj)) {
      return cache.get(obj);
    }
    const result = fn.call(this, obj);
    cache.set(obj, result);
    return result;
  };
}

// Example: expensive transformation of a config object
const processConfig = memoizeWeak((config) => {
  return Object.entries(config).reduce((acc, [k, v]) => {
    acc[k.toUpperCase()] = String(v);
    return acc;
  }, {});
});

const cfg = { host: 'localhost', port: 3000 };
processConfig(cfg); // computed
processConfig(cfg); // cache hit

// When cfg goes out of scope, the WeakMap entry is automatically cleaned up
// Plain Map would hold cfg alive indefinitely — a memory leak
```

**WeakMap constraint:** Keys must be objects or non-registered symbols. You cannot WeakMap-key a number, string, boolean, or null. For mixed argument types, use the trie approach from section 4.

---

## 6. LRU Cache with Bounded Size

Unbounded caches grow forever. An **LRU (Least Recently Used)** cache evicts the entry that was accessed least recently when the cache is full.

Implementation uses a **doubly-linked list + Map**. The Map gives O(1) lookup; the linked list maintains access order for O(1) eviction.

```js
class LRUCache {
  constructor(capacity) {
    this.capacity = capacity;
    this.map = new Map();
    // Sentinel head and tail — never evicted, simplify edge cases
    this.head = { key: null, value: null, prev: null, next: null };
    this.tail = { key: null, value: null, prev: null, next: null };
    this.head.next = this.tail;
    this.tail.prev = this.head;
  }

  get(key) {
    if (!this.map.has(key)) return -1;
    const node = this.map.get(key);
    this._remove(node);
    this._insertFront(node); // move to most-recently-used position
    return node.value;
  }

  put(key, value) {
    if (this.map.has(key)) {
      this._remove(this.map.get(key));
    }
    const node = { key, value, prev: null, next: null };
    this._insertFront(node);
    this.map.set(key, node);

    if (this.map.size > this.capacity) {
      // Evict least recently used (node before tail)
      const lru = this.tail.prev;
      this._remove(lru);
      this.map.delete(lru.key);
    }
  }

  _remove(node) {
    node.prev.next = node.next;
    node.next.prev = node.prev;
  }

  _insertFront(node) {
    // Insert after sentinel head = most recently used position
    node.next = this.head.next;
    node.prev = this.head;
    this.head.next.prev = node;
    this.head.next = node;
  }
}

// Wrap as a memoize function with bounded cache
function memoizeLRU(fn, capacity = 100) {
  const lru = new LRUCache(capacity);

  return function(...args) {
    const key = JSON.stringify(args); // or trie for object args
    const cached = lru.get(key);
    if (cached !== -1) return cached;

    const result = fn.apply(this, args);
    lru.put(key, result);
    return result;
  };
}

// Verify LRU behavior
const cache = new LRUCache(2);
cache.put(1, 'a'); // {1:'a'}
cache.put(2, 'b'); // {1:'a', 2:'b'}
cache.get(1);      // 'a' — 1 is now most recent
cache.put(3, 'c'); // evicts 2 (LRU), {1:'a', 3:'c'}
cache.get(2);      // -1 (evicted)
cache.get(3);      // 'c'
```

**Complexity:** get and put are both O(1) — constant time regardless of cache size.

---

## 7. Memoize with TTL (Time-To-Live / Expiry)

For data that becomes stale, attach an expiry timestamp to each cache entry.

```js
function memoizeTTL(fn, ttlMs) {
  const cache = new Map();

  return function(...args) {
    const key = JSON.stringify(args);
    const now = Date.now();

    if (cache.has(key)) {
      const { value, expiresAt } = cache.get(key);
      if (now < expiresAt) {
        return value; // still fresh
      }
      // Expired — fall through to recompute
    }

    const value = fn.apply(this, args);
    cache.set(key, { value, expiresAt: now + ttlMs });
    return value;
  };
}

// Usage: cache API-like computation for 5 seconds
const getRate = memoizeTTL((currency) => fetchExchangeRate(currency), 5000);

getRate('USD'); // computed, cached
getRate('USD'); // cache hit (within 5s)
// ... 5+ seconds later ...
getRate('USD'); // expired, recomputed
```

### TTL with Periodic Cleanup

The above leaks stale entries in memory. Add a cleanup sweep:

```js
function memoizeTTL(fn, ttlMs, cleanupIntervalMs = ttlMs * 2) {
  const cache = new Map();

  const cleanup = setInterval(() => {
    const now = Date.now();
    for (const [key, { expiresAt }] of cache) {
      if (now >= expiresAt) cache.delete(key);
    }
  }, cleanupIntervalMs);

  // Allow the interval to be cleared to prevent keeping the process alive
  if (cleanup.unref) cleanup.unref(); // Node.js: don't block process exit

  function memoized(...args) {
    const key = JSON.stringify(args);
    const now = Date.now();

    if (cache.has(key)) {
      const entry = cache.get(key);
      if (now < entry.expiresAt) return entry.value;
    }

    const value = fn.apply(this, args);
    cache.set(key, { value, expiresAt: now + ttlMs });
    return value;
  }

  memoized.clearCache = () => cache.clear();
  memoized.invalidate = (...args) => cache.delete(JSON.stringify(args));

  return memoized;
}
```

---

## 8. Async Memoization: Caching Promises

Async memoization has two distinct concerns:

1. **Cache the resolved value** — don't re-fetch if we already have data.
2. **In-flight deduplication** — if two callers request the same key simultaneously, they should share one in-flight Promise, not fire two separate requests.

```js
function memoizeAsync(fn) {
  const cache = new Map(); // key -> Promise (or resolved value)

  return function(...args) {
    const key = JSON.stringify(args);

    if (cache.has(key)) {
      return cache.get(key); // returns cached Promise or resolved value
    }

    // Store the Promise immediately — before it resolves.
    // Any concurrent callers hitting this key will get the SAME Promise.
    const promise = Promise.resolve(fn.apply(this, args)).then(
      (value) => {
        // Replace the Promise with the settled value for future synchronous reads
        cache.set(key, Promise.resolve(value));
        return value;
      },
      (error) => {
        // On failure, remove from cache so next call retries
        cache.delete(key);
        return Promise.reject(error);
      }
    );

    cache.set(key, promise);
    return promise;
  };
}

// Usage
const fetchUser = memoizeAsync(async (id) => {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
});

// These three fire simultaneously — only ONE network request is made
const [a, b, c] = await Promise.all([
  fetchUser(42),
  fetchUser(42), // same in-flight Promise returned
  fetchUser(42), // same in-flight Promise returned
]);

// Subsequent call: resolved value returned
const d = await fetchUser(42); // cache hit, no network request
```

**Critical detail:** We cache the Promise before `await` — during the tick before it settles. This is what enables in-flight deduplication. If you awaited first and then cached, concurrent callers would each fire their own request.

### Async LRU Memoize (Combined)

```js
function memoizeAsyncLRU(fn, capacity = 50, ttlMs = Infinity) {
  const lru = new LRUCache(capacity); // from section 6
  const inFlight = new Map();

  return async function(...args) {
    const key = JSON.stringify(args);
    const now = Date.now();

    // Check LRU cache
    const cached = lru.get(key);
    if (cached !== -1 && now < cached.expiresAt) {
      return cached.value;
    }

    // Check in-flight
    if (inFlight.has(key)) {
      return inFlight.get(key);
    }

    const promise = fn.apply(this, args).then(
      (value) => {
        lru.put(key, { value, expiresAt: now + ttlMs });
        inFlight.delete(key);
        return value;
      },
      (err) => {
        inFlight.delete(key);
        return Promise.reject(err);
      }
    );

    inFlight.set(key, promise);
    return promise;
  };
}
```

---

## 9. Recursive Memoization

### Naive Fibonacci — O(2^n)

```js
function fib(n) {
  if (n <= 1) return n;
  return fib(n - 1) + fib(n - 2);
}
// fib(40) makes ~330 million calls
// fib(50) takes several minutes
```

### Wrong way: wrapping the outer function only

```js
const memoFib = memoize(fib); // won't help!
// memoFib(10) is cached, but fib(9) and fib(8) inside still call the
// un-memoized fib recursively. Only the top-level call is cached.
```

### Correct way 1: explicit memo map passed through recursion

```js
function fib(n, memo = new Map()) {
  if (n <= 1) return n;
  if (memo.has(n)) return memo.get(n);
  const result = fib(n - 1, memo) + fib(n - 2, memo);
  memo.set(n, result);
  return result;
}

fib(50); // instant, O(n) time, O(n) space
```

### Correct way 2: self-referencing closure

```js
const fib = (() => {
  const cache = new Map();
  return function fib(n) {
    if (n <= 1) return n;
    if (cache.has(n)) return cache.get(n);
    const result = fib(n - 1) + fib(n - 2);
    cache.set(n, result);
    return result;
  };
})();

fib(100); // 354224848179261915075n (BigInt needed for exact value, but you get the idea)
```

**Why the naive wrap fails:** The internal recursive calls reference the original `fib` symbol, not the memoized wrapper. Memoization only intercepts calls made through the wrapper. The self-referencing closure works because all recursive calls go through the same `fib` name that closes over the cache.

### Complexity Comparison

| Approach | Time | Space | Calls for fib(40) |
|---|---|---|---|
| Naive recursion | O(2^n) | O(n) stack | ~331 million |
| Memoized | O(n) | O(n) | ~79 |
| Bottom-up DP | O(n) | O(1) | n iterations |

Memoization trades space (cache) for time. Bottom-up DP is often preferable when you know the access pattern is linear.

---

## 10. React's useMemo and useCallback

React's hooks are framework-level memoization — they use the same concept but are integrated with the component render cycle.

### useMemo — memoize a computed value

```jsx
import { useMemo, useState } from 'react';

function ProductList({ items, filterText }) {
  // Re-computed only when items or filterText changes
  // Without useMemo, this runs on every render including unrelated state updates
  const filtered = useMemo(
    () => items.filter(item => item.name.includes(filterText)),
    [items, filterText]
  );

  return <ul>{filtered.map(i => <li key={i.id}>{i.name}</li>)}</ul>;
}
```

### useCallback — memoize a function reference

```jsx
function Parent() {
  const [count, setCount] = useState(0);
  const [text, setText] = useState('');

  // Without useCallback: new function reference every render
  // → Child re-renders even when count changes and text hasn't
  const handleTextChange = useCallback((e) => {
    setText(e.target.value);
  }, []); // stable reference, no dependencies

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      <ExpensiveChild onChange={handleTextChange} /> {/* won't re-render on count change */}
    </>
  );
}

const ExpensiveChild = React.memo(({ onChange }) => {
  console.log('ExpensiveChild rendered');
  return <input onChange={onChange} />;
});
```

### How React's Memoization Differs from General Memoization

| Aspect | General memoize() | useMemo / useCallback |
|---|---|---|
| Cache size | Unlimited (or LRU-bounded) | Only 1 entry — previous render only |
| Cache key | All arguments, arbitrary depth | Dependency array, shallow equality |
| Persistence | Process lifetime (or TTL) | Per component instance, cleared on unmount |
| Purpose | Avoid recomputation globally | Avoid recomputation per render |
| Cost if deps change | Cache miss, recompute | Recomputes on every render where deps changed |

**Common mistake:** Wrapping every computation in `useMemo`. React's docs are explicit: `useMemo` is a performance optimization, not a semantic guarantee. Overusing it adds overhead (dependency comparison) for cheap computations. Profile first.

---

## 11. Reselect / createSelector: Memoized Derived State in Redux

Reselect implements memoization for Redux selectors. It ensures derived state is only recomputed when the inputs that feed it actually change.

```js
import { createSelector } from 'reselect';

// Input selectors — simple, fast
const selectItems = (state) => state.cart.items;
const selectTaxRate = (state) => state.settings.taxRate;

// Memoized selector — only runs when selectItems or selectTaxRate output changes
const selectCartTotal = createSelector(
  [selectItems, selectTaxRate],
  (items, taxRate) => {
    console.log('Recomputing total...'); // only fires on actual change
    const subtotal = items.reduce((sum, item) => sum + item.price * item.qty, 0);
    return subtotal * (1 + taxRate);
  }
);

// Usage in component
const total = useSelector(selectCartTotal);
// If cart or taxRate hasn't changed, returns cached value — no recomputation
```

### How createSelector Works Internally

```js
// Simplified createSelector implementation
function createSelector(inputSelectors, resultFn) {
  let lastInputs = null;
  let lastResult = null;

  return function(state, ...args) {
    const inputs = inputSelectors.map(sel => sel(state, ...args));

    // Shallow equality check on each input
    const inputsChanged = lastInputs === null ||
      inputs.some((inp, i) => inp !== lastInputs[i]);

    if (!inputsChanged) {
      return lastResult; // memoized
    }

    lastInputs = inputs;
    lastResult = resultFn(...inputs);
    return lastResult;
  };
}
```

**Key detail:** Reselect uses strict reference equality (`===`) for each input selector's output. This is why Redux reducers must return new object references on change — the selector won't recompute if the reference is the same even if the contents differ (a mutation bug).

### Parameterized Selectors (Redux Toolkit pattern)

```js
import { createSelector } from '@reduxjs/toolkit';

// Factory function — each call creates an independent memoized selector
const makeSelectItemsByCategory = () =>
  createSelector(
    [(state) => state.items, (_, category) => category],
    (items, category) => items.filter(i => i.category === category)
  );

// In component — each component instance gets its own memoized selector
const selectElectronics = useMemo(makeSelectItemsByCategory, []);
const electronics = useSelector(state => selectElectronics(state, 'electronics'));
```

---

## 12. When NOT to Memoize

Memoization has real costs. Apply it only when it pays.

### 1. Cheap computations

```js
// Wrong: memoizing a trivial operation
const add = memoize((a, b) => a + b);
// Cache lookup (Map.has + Map.get) costs more than a + b
// You've made it slower

// Right: just call it
const sum = (a, b) => a + b;
```

### 2. Frequently changing inputs

```js
// Wrong: inputs change on nearly every call — near-zero cache hit rate
const processTimestamp = memoize((ts) => new Date(ts).toISOString());
setInterval(() => processTimestamp(Date.now()), 100); // cache grows forever, never hits
```

### 3. Equality checks cost more than recomputation

```js
// Deep structural equality to compare complex objects
const expensiveKey = memoize(
  (obj) => computeSomething(obj),
  { equalityFn: deepEqual } // deepEqual itself is O(n) — may cost more than computeSomething
);
```

### 4. Side-effectful functions

```js
// Wrong: memoize will suppress the side effect on repeated calls
const createUserAndLog = memoize(async (userId) => {
  await db.createUser(userId);   // side effect
  logger.info(`created ${userId}`); // side effect
  return userId;
});
// Second call with same userId: skips DB write and logging silently
```

### 5. Large argument surface area

```js
// Wrong: every call has a unique key because the data set is different each time
const processReport = memoize((rows) => aggregateRows(rows));
// If rows is a fresh array from each network response, cache never hits
// but grows indefinitely — pure memory waste
```

### Decision checklist

- Is the function pure? If no — do not memoize.
- Is the computation measurably expensive? Profile first.
- Do inputs repeat across calls within the cache lifetime? If rarely — skip it.
- Is the result set bounded? Unbounded inputs need LRU or TTL.
- Do equality checks cost less than recomputation? Shallow equality is cheap; deep is often not.

---

## Interview Q&A

---

**Q1: Implement a memoize function that handles any number of arguments.**

```js
function memoize(fn) {
  const cache = new Map();
  return function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}
```

*Follow-up: "What are the weaknesses of using JSON.stringify?"*
- Throws on circular references
- Functions and Symbols serialize to undefined/nothing
- Two object args with same shape but different references map to the same key — may be wrong if the function cares about identity

*Stronger answer:* Use a trie keyed by argument identity (WeakMap for objects, Map for primitives), as shown in section 4. Each node in the trie corresponds to one argument in position.

---

**Q2: Implement an LRU cache with get and put, O(1) time complexity.**

The key constraint is O(1) for both operations. A plain Map gives O(1) lookup but O(n) for finding the least recently used entry. We need a doubly-linked list to track access order in O(1).

```js
class LRUCache {
  constructor(capacity) {
    this.capacity = capacity;
    this.map = new Map();
    this.head = { prev: null, next: null }; // sentinel MRU end
    this.tail = { prev: null, next: null }; // sentinel LRU end
    this.head.next = this.tail;
    this.tail.prev = this.head;
  }

  get(key) {
    if (!this.map.has(key)) return -1;
    const node = this.map.get(key);
    this._remove(node);
    this._addFront(node);
    return node.value;
  }

  put(key, value) {
    if (this.map.has(key)) this._remove(this.map.get(key));
    const node = { key, value };
    this._addFront(node);
    this.map.set(key, node);
    if (this.map.size > this.capacity) {
      const lru = this.tail.prev;
      this._remove(lru);
      this.map.delete(lru.key);
    }
  }

  _remove(node) {
    node.prev.next = node.next;
    node.next.prev = node.prev;
  }

  _addFront(node) {
    node.next = this.head.next;
    node.prev = this.head;
    this.head.next.prev = node;
    this.head.next = node;
  }
}
```

*Explain:* `Map` gives O(1) key-to-node lookup. The doubly-linked list maintains order — head side is MRU, tail side is LRU. Removing and re-inserting a node at the front is O(1) pointer surgery.

---

**Q3: How would you memoize an async function and prevent duplicate in-flight requests?**

```js
function memoizeAsync(fn) {
  const cache = new Map();
  return function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);

    const promise = fn.apply(this, args).then(
      (val) => { cache.set(key, Promise.resolve(val)); return val; },
      (err) => { cache.delete(key); return Promise.reject(err); }
    );

    cache.set(key, promise); // cache BEFORE resolve — this deduplicates in-flight
    return promise;
  };
}
```

*Key points to mention:*
- We cache the Promise itself, not just the resolved value. This means concurrent callers for the same key all receive the same Promise object — a single network request.
- On rejection, we delete the cache entry so the next caller retries rather than getting a permanently cached failure.
- After resolution, we replace the pending Promise with `Promise.resolve(value)` — future callers still get a Promise (consistent interface), but one that resolves synchronously on the next tick.

---

**Q4: Why does wrapping `fib` with a generic memoize function not speed it up?**

```js
const fib = memoize(function fib(n) {
  if (n <= 1) return n;
  return fib(n - 1) + fib(n - 2); // calls the original fib, not memoFib
});
```

The `fib` symbol inside the function body refers to the function *before* it was wrapped. The wrapper only intercepts calls made through the `memoize`-returned function. Internal recursive calls bypass the cache entirely.

*Fix: close over the cache explicitly so all recursive calls go through the same memoized path.*

```js
const fib = (() => {
  const cache = new Map();
  function fib(n) {
    if (n <= 1) return n;
    if (cache.has(n)) return cache.get(n);
    const result = fib(n - 1) + fib(n - 2); // now calls this memoized fib
    cache.set(n, result);
    return result;
  }
  return fib;
})();
```

---

**Q5: When should you NOT memoize, and what are the failure modes?**

*Structured answer:*

1. **Impure functions** — memoization suppresses side effects silently. A logging function, DB write, or API call that appears to succeed will not re-execute on cached calls.

2. **Cheap computations** — the Map.has + Map.get overhead can exceed the cost of a simple arithmetic expression. Always profile before adding memoization.

3. **High input cardinality with low repetition** — if inputs rarely repeat (e.g., timestamps, unique IDs), cache hit rate is near zero but the cache grows without bound. Use LRU with a capacity limit or skip memoization.

4. **Deep equality key comparison** — if arguments are objects that you want to compare by value (not reference), you need deep equality. That is O(n) per comparison and often costs more than just recomputing.

5. **Memory pressure** — unbounded caches can exhaust heap memory in long-running processes. Always pair memoization with LRU eviction or TTL expiry in production code.

*Rule of thumb:* measure first. `console.time`, `performance.mark`, or a profiler should confirm that the function is actually a bottleneck before adding memoization complexity.
