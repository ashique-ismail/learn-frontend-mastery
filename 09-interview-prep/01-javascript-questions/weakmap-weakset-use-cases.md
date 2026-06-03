# WeakMap & WeakSet — Use Cases and Internals

## Table of Contents
- [Map/Set vs WeakMap/WeakSet — The Core Difference](#mapset-vs-weakmapweakset--the-core-difference)
- [Weak References Explained](#weak-references-explained)
- [Why No Iteration, No Size, No clear()](#why-no-iteration-no-size-no-clear)
- [Key Constraint: Objects Only (and Registered Symbols)](#key-constraint-objects-only-and-registered-symbols)
- [Use Case 1: Private Instance Data Without Memory Leaks](#use-case-1-private-instance-data-without-memory-leaks)
- [Use Case 2: Memoization / Caching by Object Identity](#use-case-2-memoization--caching-by-object-identity)
- [Use Case 3: Tracking DOM Nodes Without Preventing GC](#use-case-3-tracking-dom-nodes-without-preventing-gc)
- [Use Case 4: Associating Metadata with Third-Party Objects](#use-case-4-associating-metadata-with-third-party-objects)
- [Use Case 5: WeakSet — Visited Marking in Graph Traversal](#use-case-5-weakset--visited-marking-in-graph-traversal)
- [WeakRef and FinalizationRegistry (ES2021)](#weakref-and-finalizationregistry-es2021)
- [Comparison Table: WeakMap vs Map](#comparison-table-weakmap-vs-map)
- [Interview Q&A](#interview-qa)

---

## Map/Set vs WeakMap/WeakSet — The Core Difference

`Map` and `Set` are general-purpose keyed/value collections. They hold **strong references** to everything they contain — an object stored as a `Map` key cannot be garbage-collected as long as the `Map` itself is alive, regardless of whether any other code still needs that object.

`WeakMap` and `WeakSet` hold **weak references** to their keys (or members, in the case of `WeakSet`). The garbage collector is allowed to reclaim the object even if it is still a key in a `WeakMap` — the map's hold on it does not count as "still reachable."

```javascript
// Strong reference — Map keeps obj alive forever
const map = new Map();
let obj = { id: 1 };
map.set(obj, 'some data');
obj = null; // You released your reference...
// ...but map still holds a strong reference.
// obj is NOT eligible for GC. map.size === 1.

// Weak reference — WeakMap does not prevent GC
const wmap = new WeakMap();
let obj2 = { id: 2 };
wmap.set(obj2, 'some data');
obj2 = null; // You released the only strong reference.
// obj2 is now eligible for GC.
// When collected, the WeakMap entry disappears automatically.
```

| Feature          | Map / Set              | WeakMap / WeakSet          |
|------------------|------------------------|----------------------------|
| Reference type   | Strong                 | Weak (keys/members only)   |
| Key types        | Any value              | Objects + registered symbols |
| Iterable         | Yes (`for...of`, etc.) | No                         |
| `size` property  | Yes                    | No                         |
| `clear()` method | Yes                    | No                         |
| GC behaviour     | Prevents collection    | Allows collection          |

---

## Weak References Explained

A **weak reference** is a reference that the garbage collector ignores when deciding whether an object is still "reachable." In JavaScript's mark-and-sweep GC, the collector starts from roots (global scope, stack frames, etc.) and follows strong references. Anything not reachable through a strong-reference chain is eligible for collection.

`WeakMap` keys participate in this process:

```
Strong-reference chain:

  [root] → myWeakMap     (strong, keeps the WeakMap alive)
  [root] → obj           (strong, keeps obj alive)

  myWeakMap ---(weak key reference)---> obj

If we do: obj = null
  [root] → myWeakMap     (still alive)
  [root] → obj           GONE — no more strong references to the object

  The GC sees the object is unreachable via strong references.
  It reclaims the object.
  The corresponding WeakMap entry is removed automatically.
```

The value side of a `WeakMap` is held **strongly** — `wmap.get(key)` returns the value only while the key object is alive. Once the key is collected, the value is released too (because the entry itself disappears).

---

## Why No Iteration, No Size, No clear()

This is a deliberate design choice rooted in GC non-determinism.

The JavaScript specification does not mandate *when* the garbage collector runs or *in what order* it collects objects. The same program could observe different GC timings across:
- V8 vs SpiderMonkey vs JavaScriptCore
- Different Node.js versions
- Low-memory vs high-memory devices
- Different load conditions at runtime

If `WeakMap` supported iteration or a `size` property, the visible state of the collection would depend on GC timing — a form of **observable non-determinism**. Two programs with identical logic could see different iteration results based solely on when the engine chose to run a GC cycle.

```javascript
// Hypothetical (does NOT exist) — why this would be dangerous:
const wmap = new WeakMap();
let a = {};
let b = {};
wmap.set(a, 1);
wmap.set(b, 2);

a = null; // eligible for GC — but when?

// On one machine: wmap.size === 1  (GC already ran)
// On another:     wmap.size === 2  (GC hasn't run yet)
// This is determinism-breaking: same code, different observable outcomes.
```

By making `WeakMap` and `WeakSet` non-iterable and size-less, the spec ensures that **the only visible behaviour is per-key lookup** — and lookup results are consistent: either the key is alive and you get the value, or the key is gone and you get `undefined` / `false`. GC timing never affects observable state.

This also prevents a subtle security risk: if you could enumerate a `WeakMap`, you could detect which objects the GC had collected, enabling timing attacks or covert channels across security boundaries (relevant in browser iframe scenarios).

---

## Key Constraint: Objects Only (and Registered Symbols)

Primitive values (`string`, `number`, `boolean`, `null`, `undefined`, `bigint`) cannot be `WeakMap` keys. The reason is fundamental to how weak references work.

A weak reference is meaningful only when there is a concept of **object identity** — a unique, addressable thing in memory that the GC manages. Primitives in JavaScript are **value-typed**: `"foo"` is not an object with an address; it is just the value `"foo"`. There is no heap allocation to hold weakly; there is nothing for the GC to reclaim.

```javascript
const wmap = new WeakMap();

// All of these throw TypeError: Invalid value used as weak map key
wmap.set('string', 1);
wmap.set(42, 1);
wmap.set(true, 1);
wmap.set(null, 1);

// Objects work because they have a unique heap address
wmap.set({}, 1);        // OK
wmap.set([], 1);        // OK
wmap.set(function(){}, 1); // OK
```

**ES2023 addition — registered symbols as keys:**

`Symbol.for('key')` creates a symbol in the global symbol registry. These *registered* symbols are now allowed as `WeakMap` keys because the registry gives them a stable, addressable identity. Plain `Symbol()` values (unregistered) are still not allowed.

```javascript
const sym = Symbol.for('my.key'); // registered symbol — has stable identity
const wmap = new WeakMap();
wmap.set(sym, 'metadata');   // OK in ES2023+

const plainSym = Symbol('plain');
// wmap.set(plainSym, 'x'); // TypeError — unregistered symbol
```

---

## Use Case 1: Private Instance Data Without Memory Leaks

Before the `#privateField` syntax (introduced in ES2022), developers needed a way to attach genuinely private data to class instances — data that:
1. Could not be accessed from outside the class
2. Did not prevent GC when the instance was discarded

`WeakMap` is the canonical pre-`#` solution. Using a closure over a module-scoped `WeakMap`, each instance gets associated data that only the class methods can reach.

```javascript
// Module scope — not exported, not reachable from outside
const _private = new WeakMap();

class BankAccount {
  constructor(owner, initialBalance) {
    // Private data stored in the WeakMap, keyed by the instance
    _private.set(this, {
      owner,
      balance: initialBalance,
      transactions: [],
    });
  }

  deposit(amount) {
    if (amount <= 0) throw new RangeError('Amount must be positive');
    const data = _private.get(this);
    data.balance += amount;
    data.transactions.push({ type: 'deposit', amount, at: Date.now() });
    return this;
  }

  withdraw(amount) {
    const data = _private.get(this);
    if (amount > data.balance) throw new Error('Insufficient funds');
    data.balance -= amount;
    data.transactions.push({ type: 'withdrawal', amount, at: Date.now() });
    return this;
  }

  get balance() {
    return _private.get(this).balance;
  }

  get history() {
    // Return a copy — never expose the internal array directly
    return [..._private.get(this).transactions];
  }
}

let account = new BankAccount('Alice', 1000);
account.deposit(500).withdraw(200);
console.log(account.balance); // 1300
console.log(account.history.length); // 2

// _private is inaccessible outside this module — true encapsulation

// Memory: when `account` goes out of scope, its WeakMap entry is
// automatically removed. No manual cleanup required.
account = null;
// The { owner, balance, transactions } object is now eligible for GC.
// If this had been a plain Map, the entry would persist forever.
```

**Why not use a closure per instance?** A per-instance closure creates a new function object per method per instance — expensive at scale. The `WeakMap` approach keeps a single set of prototype methods while storing per-instance state externally, with automatic cleanup.

---

## Use Case 2: Memoization / Caching by Object Identity

When a computation depends on an object's contents and you want to cache the result, using the object itself as a cache key is natural. `WeakMap` makes this safe: when the input object is discarded, the cached result is discarded too — no bounded cache size management required.

```javascript
const computeCache = new WeakMap();

/**
 * Expensive analysis of an object's structure.
 * Result is cached per object instance.
 */
function analyzeSchema(obj) {
  if (computeCache.has(obj)) {
    return computeCache.get(obj);
  }

  // Simulate expensive work
  const result = {
    keys: Object.keys(obj),
    types: Object.fromEntries(
      Object.entries(obj).map(([k, v]) => [k, typeof v])
    ),
    depth: getObjectDepth(obj),
    computedAt: Date.now(),
  };

  computeCache.set(obj, result);
  return result;
}

function getObjectDepth(obj, current = 0) {
  if (typeof obj !== 'object' || obj === null) return current;
  return Math.max(...Object.values(obj).map(v => getObjectDepth(v, current + 1)));
}

const config = { host: 'localhost', port: 3000, db: { name: 'mydb', pool: 5 } };
const r1 = analyzeSchema(config); // computed
const r2 = analyzeSchema(config); // cache hit — same object reference
console.log(r1 === r2); // true

// ---
// Compare with a Map-based cache — the dangerous version:
const mapCache = new Map();

function memoize(fn) {
  return function(obj) {
    if (mapCache.has(obj)) return mapCache.get(obj);
    const result = fn(obj);
    mapCache.set(obj, result); // obj is now pinned in memory as long as mapCache lives
    return result;
  };
}
// Every object ever passed to this memoized function lives forever.
// WeakMap-based version has zero memory management overhead.
```

**Limitation to be aware of:** `WeakMap` is keyed by *object identity*, not value equality. Two separate objects with the same contents will get separate cache entries. This is appropriate when the cache should be invalidated if you construct a fresh object — it is not appropriate for value-based deduplication.

---

## Use Case 3: Tracking DOM Nodes Without Preventing GC

A common front-end pattern is attaching extra data to DOM elements — event handler cleanup functions, render state, animation progress, component references. Storing this in a plain `Map` means you must manually delete entries when the element is removed from the DOM, or you leak the element's memory.

```javascript
// Attach cleanup / metadata to DOM elements without holding them alive
const domMeta = new WeakMap();

function registerElement(el, options = {}) {
  const cleanup = [];

  if (options.resizeHandler) {
    const observer = new ResizeObserver(options.resizeHandler);
    observer.observe(el);
    cleanup.push(() => observer.disconnect());
  }

  if (options.clickHandler) {
    el.addEventListener('click', options.clickHandler);
    cleanup.push(() => el.removeEventListener('click', options.clickHandler));
  }

  domMeta.set(el, {
    registeredAt: Date.now(),
    cleanup,
    ...options.data,
  });
}

function teardown(el) {
  const meta = domMeta.get(el);
  if (!meta) return;
  meta.cleanup.forEach(fn => fn());
  // No need to delete from domMeta — it's a WeakMap.
  // When el is removed from the DOM and all strong references are dropped,
  // the WeakMap entry (including the cleanup array) is collected automatically.
}

// Usage
const card = document.querySelector('.product-card');
registerElement(card, {
  clickHandler: () => console.log('card clicked'),
  data: { productId: 42, category: 'electronics' },
});

// Later: remove element from DOM
card.parentNode.removeChild(card);
// Any code that held a reference to `card` releases it.
// WeakMap entry is eligible for GC — no manual domMeta.delete(card) needed.
```

**Contrast with using a plain Map:**

```javascript
// DANGEROUS: Map keeps the detached element in memory
const elementMap = new Map();
elementMap.set(card, { productId: 42 });

card.parentNode.removeChild(card);
// card is now "detached" — removed from the DOM tree,
// but elementMap still holds a strong reference.
// The entire DOM subtree rooted at `card` cannot be collected.
// This is a classic detached DOM node memory leak.
```

---

## Use Case 4: Associating Metadata with Third-Party Objects

When you use a third-party library, you often receive objects you do not own. You cannot safely add properties to them (that may conflict with the library's own properties, enumerable keys, or future additions). `WeakMap` lets you associate arbitrary data with these objects without touching them.

```javascript
// Imagine a third-party graph library that returns node objects
// import { createNode } from 'graph-lib';

// You want to track your own application-level metadata per node,
// but you cannot (safely) do: node._myAppData = { ... }

const nodeMetadata = new WeakMap();

function annotateNode(node, annotation) {
  const existing = nodeMetadata.get(node) ?? {};
  nodeMetadata.set(node, { ...existing, ...annotation });
}

function getAnnotation(node) {
  return nodeMetadata.get(node) ?? null;
}

// Usage with third-party objects
const thirdPartyNode = someGraphLib.createNode({ label: 'Start' });

annotateNode(thirdPartyNode, {
  visitedAt: Date.now(),
  color: '#ff6b6b',
  selected: false,
  renderOrder: 1,
});

// Later retrieval
const meta = getAnnotation(thirdPartyNode);
console.log(meta.color); // '#ff6b6b'

// You have not mutated the third-party object at all.
// When the library discards the node, your metadata is discarded too.
```

**A real-world variant — React internals:** React's fiber reconciler uses a `WeakMap`-like approach to associate internal fiber data with host DOM nodes (the `__reactFiber$...` property on DOM nodes is a simplified version of this). When a DOM node is removed, React can clean up internal state without maintaining a parallel data structure that would require manual purging.

---

## Use Case 5: WeakSet — Visited Marking in Graph Traversal

`WeakSet` stores a collection of objects, allowing membership testing (`has()`), but holds each member weakly. The primary use case is **marking** objects as having been seen/processed without preventing their GC.

Graph traversal with cycle detection is the canonical example:

```javascript
/**
 * Deep clone an object graph that may contain circular references.
 * Uses WeakSet to track already-visited objects.
 */
function deepClone(value, visited = new WeakSet()) {
  // Primitives — nothing to clone
  if (value === null || typeof value !== 'object') return value;

  // Cycle detected — return a placeholder or throw
  if (visited.has(value)) {
    // In a real implementation you'd track the clone and return it here.
    // For illustration, we signal the cycle.
    return '[Circular]';
  }

  visited.add(value);

  if (Array.isArray(value)) {
    return value.map(item => deepClone(item, visited));
  }

  const clone = Object.create(Object.getPrototypeOf(value));
  for (const key of Reflect.ownKeys(value)) {
    clone[key] = deepClone(value[key], visited);
  }
  return clone;
}

// Test with circular reference
const obj = { a: 1, b: { c: 2 } };
obj.self = obj; // circular

const cloned = deepClone(obj);
console.log(cloned.a);       // 1
console.log(cloned.b.c);     // 2
console.log(cloned.self);    // '[Circular]'
```

**Why WeakSet and not Set here?**

If you use a `Set`, the visited set holds strong references to every node in the graph. The set must remain alive for the duration of traversal — which is fine for a single call. But if `visited` is stored somewhere longer-lived (e.g., an incremental traversal or a memoized traversal helper), the `Set` would prevent all visited nodes from being GC-collected. `WeakSet` automatically releases them once they are no longer referenced by the graph itself.

```javascript
// Another real-world pattern: tracking objects that have been initialised
const initialised = new WeakSet();

class Plugin {
  init(context) {
    if (initialised.has(context)) {
      console.warn('Already initialised with this context');
      return;
    }
    // ... do initialisation work ...
    initialised.add(context);
  }
}

// When `context` is GC-collected, the WeakSet entry vanishes.
// No bookkeeping required — no risk of the set growing unboundedly.
```

**WeakSet vs Set for "seen" marking:**

```javascript
// Set — leaks all visited nodes for the lifetime of the set
const visitedSet = new Set();
function traverseWithSet(graph) {
  visitedSet.clear(); // Must manually clear or nodes accumulate
  // ...
}

// WeakSet — automatically cleans up as nodes are released
const visitedWeak = new WeakSet(); // no need to ever clear()
function traverseWithWeakSet(node) {
  if (visitedWeak.has(node)) return;
  visitedWeak.add(node);
  for (const neighbour of node.neighbours) {
    traverseWithWeakSet(neighbour);
  }
}
```

---

## WeakRef and FinalizationRegistry (ES2021)

`WeakRef` and `FinalizationRegistry` are lower-level primitives that expose weak references more directly. They are related to `WeakMap`/`WeakSet` but serve different purposes.

### WeakRef

`WeakRef` wraps a single object with a weak reference. You call `.deref()` to get the object back — or `undefined` if it has been collected.

```javascript
let heavyObject = { data: new Array(1_000_000).fill('x') };
const ref = new WeakRef(heavyObject);

// Release the strong reference
heavyObject = null;

// Later... (after GC may or may not have run)
function useIfAlive() {
  const obj = ref.deref();
  if (obj === undefined) {
    console.log('Object was collected — nothing to do');
    return;
  }
  console.log('Object still alive, length:', obj.data.length);
}
```

**Important:** You must always check `deref()` for `undefined`. The GC can run between any two JavaScript statements, and you cannot predict when an object referenced only by a `WeakRef` will be collected.

### FinalizationRegistry

`FinalizationRegistry` lets you register a callback that fires *after* an object is collected.

```javascript
const registry = new FinalizationRegistry((heldValue) => {
  // heldValue is what you passed as the third arg to register()
  // It is NOT the collected object (that's gone) — it's your bookkeeping token.
  console.log(`Resource cleaned up for token: ${heldValue}`);
  releaseNativeResource(heldValue);
});

function acquireResource(id) {
  const resource = new ExternalResource(id); // wraps a native handle
  registry.register(resource, id); // when resource is GC'd, callback fires with `id`
  return resource;
}
```

**Caveats — when to use (and when not to):**

- GC callbacks are **non-deterministic**: the spec does not guarantee they will ever fire (e.g., in a short-lived Node.js process that exits cleanly). Never use them for correctness-critical cleanup.
- They run in a **microtask** after collection, so timing is unpredictable.
- They are appropriate for: logging/diagnostics, releasing optional native resources, cache eviction notifications.
- They are **not** a substitute for explicit lifecycle management (`.dispose()`, `using` declarations in TC39 Explicit Resource Management).

```javascript
// Appropriate: optional diagnostic logging
const leakDetector = new FinalizationRegistry((tag) => {
  console.warn(`[LeakDetector] Object "${tag}" was GC'd.`);
});

function track(obj, tag) {
  leakDetector.register(obj, tag);
  return obj;
}

// Not appropriate: correctness-critical cleanup
// BAD — don't rely on this to close a file handle reliably:
const fileRegistry = new FinalizationRegistry((fd) => {
  fs.closeSync(fd); // may never run, or run too late
});
// GOOD — explicit close in a finally block / using declaration instead.
```

---

## Comparison Table: WeakMap vs Map

| Dimension                        | Map                                     | WeakMap                                           |
|----------------------------------|-----------------------------------------|---------------------------------------------------|
| Key types                        | Any value (including primitives)        | Objects + registered symbols only                |
| GC behaviour                     | Strong reference — key is never collected while map lives | Weak reference — key can be collected             |
| Iterable                         | Yes (`keys()`, `values()`, `entries()`, `for...of`) | No                                                |
| `size`                           | Available                               | Not available                                     |
| `clear()`                        | Available                               | Not available                                     |
| Memory profile                   | Grows indefinitely unless manually pruned | Self-pruning — entries removed when keys are collected |
| Use when key lifetime            | Is managed by you / matches map lifetime | Is managed externally / shorter than map lifetime |
| Typical use cases                | General lookup tables, caches with explicit eviction, counters | Private data, DOM metadata, memoization, third-party object annotations |
| JSON-serialisable                | No (natively), but iterable so manually possible | No — and cannot be iterated to build a serialization |
| Thread safety                    | N/A — JS single-threaded               | N/A                                               |
| Debug introspection              | Easy — iterate to inspect             | Hard by design — you cannot enumerate entries     |

**Decision rule (simple heuristic):**

```
Is the key an object whose lifetime I don't fully control?
  YES → WeakMap (let the GC decide when the entry goes away)
  NO  → Map (you control cleanup, or keys are primitives)
```

---

## Interview Q&A

### Q1: "Why would you use WeakMap over Map?"

**Model answer:**

The core reason is **memory safety when key lifetime is externally managed.** When you use an object as a `Map` key, the `Map` holds a strong reference to that object — it will never be garbage-collected as long as the `Map` is alive, even if every other reference to the object has been dropped. This causes subtle memory leaks in long-running applications.

`WeakMap` solves this by holding weak references to keys. When the last strong reference to a key object is released, the GC is free to collect it, and the corresponding entry in the `WeakMap` disappears automatically. You get automatic cleanup with zero bookkeeping.

Concrete scenarios where this matters:
1. **Caching computed results** keyed by input objects — the cache self-prunes as objects go out of scope.
2. **DOM node metadata** — detaching a node from the DOM and releasing all strong references allows the node to be collected along with any associated data.
3. **Private class data** — WeakMap ensures instance data is cleaned up when the instance itself is collected.

The trade-off: `WeakMap` is not enumerable and has no `size`. If you need to iterate, count entries, or use primitive keys, `Map` is the right choice.

---

### Q2: "Why can't you iterate a WeakMap?"

**Model answer:**

Iteration is intentionally omitted because `WeakMap` entries can disappear at any moment — whenever the GC decides to collect a key. The garbage collector's schedule is **non-deterministic**: it varies by engine, runtime environment, and memory pressure.

If `WeakMap` supported `keys()` or `forEach()`, the set of entries returned would depend on *when* you call the iteration method relative to when the GC last ran. Two semantically identical programs could observe different iteration results based solely on GC timing. That is a form of observable non-determinism — your program's output would depend on implementation details of the runtime, not just the logic you wrote.

By prohibiting iteration, the spec guarantees that `WeakMap`'s behaviour is consistent: `get(key)` returns a value if the key is alive, or `undefined` if it isn't. GC timing affects neither result in a user-visible, logic-breaking way.

There is also a security dimension: if you could enumerate a `WeakMap`, you could observe which objects were still alive — potentially leaking information across security boundaries in environments like multi-origin browser contexts.

---

### Q3: "Describe a real-world bug that WeakMap prevents."

**Model answer:**

Consider a front-end component system where each DOM element gets a rich component object attached to it via a `Map`:

```javascript
// Problematic pattern
const componentMap = new Map();

function mount(el, ComponentClass) {
  const instance = new ComponentClass(el);
  componentMap.set(el, instance); // strong reference
  return instance;
}

function unmount(el) {
  const inst = componentMap.get(el);
  inst?.destroy();
  el.parentNode?.removeChild(el);
  // BUG: forgot componentMap.delete(el)
  // el is still strongly referenced by componentMap.
  // The element and its entire DOM subtree remain in memory — a detached DOM leak.
}
```

In a single-page application where components mount and unmount hundreds of times, this gradually exhausts the heap. The leak is hard to spot because `componentMap` is in module scope — heap snapshots show many "Detached HTMLElement" nodes retained by an unexpected Map.

The `WeakMap` version requires no manual deletion:

```javascript
const componentMap = new WeakMap();

function unmount(el) {
  const inst = componentMap.get(el);
  inst?.destroy();
  el.parentNode?.removeChild(el);
  // When all strong references to el are dropped, the WeakMap entry
  // and the component instance are eligible for GC automatically.
}
```

The bug class is eliminated structurally — there is no `delete` call to forget.

---

### Q4: "When would WeakRef and FinalizationRegistry be more appropriate than WeakMap?"

**Model answer:**

`WeakMap` is the right tool when you want to **associate data with an object** and have that data disappear when the object does. It covers 95% of weak-reference use cases.

`WeakRef` and `FinalizationRegistry` are appropriate in two narrower scenarios:

**1. You need to hold a weak reference to a single object, not as a map key.**

If you want to maintain an optional reference to an object — use it if it's still alive, skip it if not — `WeakRef` is the right primitive. A `WeakMap` requires a separate "thing" to use as the key; `WeakRef` is just the reference itself.

Example: a caching layer that optionally holds the last-computed result, but will not prevent GC of a large object if memory pressure demands it.

**2. You need a callback when an object is collected.**

`FinalizationRegistry` lets you run cleanup code after GC. This is useful for releasing external (non-JS) resources tied to a JS object — native file handles, GPU buffers, WebAssembly memory — where you want a safety-net cleanup in case explicit `dispose()` is not called.

The critical caveat: `FinalizationRegistry` callbacks are **not guaranteed to run** (e.g., in short-lived processes, or if the engine's GC never decides to run). Use them only as a best-effort safety net, never for correctness-critical cleanup. For reliable resource cleanup, always use explicit lifecycle management — `using` declarations (TC39 Explicit Resource Management, Stage 4) or explicit `.destroy()` / `.close()` calls in `finally` blocks.
