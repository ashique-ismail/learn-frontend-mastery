# Map, Set, WeakMap, WeakSet, and WeakRef

## The Idea

**In plain English:** JavaScript gives you special containers for storing data in smarter ways than a plain list or object — some keep only unique items, some let you use any type as a label, and some are "weak," meaning they quietly let go of items the rest of your program no longer needs, so memory is not wasted.

**Real-world analogy:** Imagine a school lost-and-found office. The desk has labelled cubbyholes (Map), a bin that never stores two identical items (Set), sticky notes on student backpacks that fall off automatically once a student leaves the school (WeakMap/WeakSet), and a slip of paper with a locker number that may be empty if the locker was reassigned (WeakRef).

- The labelled cubbyholes = `Map` (any kind of label, stores associated item)
- The no-duplicates bin = `Set` (each unique item appears only once)
- The self-removing sticky note = `WeakMap`/`WeakSet` (data tied to an object; disappears when the object is gone)
- The locker-number slip = `WeakRef` (a reference that may point to nothing if the object was cleaned up)

---

## Overview

ES6 introduced four keyed collection types — `Map`, `Set`, `WeakMap`, and `WeakSet` — as purpose-built alternatives to using plain objects as dictionaries and arrays as unique-value stores. ES2021 added `WeakRef` and `FinalizationRegistry` for fine-grained garbage-collection awareness. Each collection type occupies a distinct niche: strong vs weak references, ordered vs unordered, iterable vs non-iterable. Understanding when to use each — and why the "weak" variants deliberately lack iteration — is a recurring senior interview topic.

## Map

### What is Map

`Map` is a keyed collection where keys can be any value — objects, functions, primitives, or symbols — and insertion order is preserved. Unlike plain objects, `Map` does not conflate string and symbol keys with inherited prototype properties.

```javascript
const map = new Map();

// Any type of key
map.set("string key", 1);
map.set(42, "number key");
map.set(true, "boolean key");
map.set({ id: 1 }, "object key");
map.set(Symbol("sym"), "symbol key");
map.set(null, "null key");

map.size; // 6
map.get(42); // "number key"
map.has(true); // true
map.delete(null);
map.size; // 5
```

### Construction

```javascript
// From iterable of [key, value] pairs
const map = new Map([
  ["a", 1],
  ["b", 2],
  ["c", 3]
]);

// From another Map
const copy = new Map(map);

// From Object.entries
const fromObj = new Map(Object.entries({ x: 10, y: 20 }));
```

### Iteration

Map maintains insertion order and is iterable:

```javascript
const map = new Map([["a", 1], ["b", 2], ["c", 3]]);

for (const [key, value] of map) {
  console.log(key, value); // "a" 1, "b" 2, "c" 3
}

[...map.keys()];    // ["a", "b", "c"]
[...map.values()];  // [1, 2, 3]
[...map.entries()]; // [["a",1], ["b",2], ["c",3]]

map.forEach((value, key) => console.log(key, value));
```

### Object Keys in Map

Map uses **SameValueZero** equality (like `===` but treats `NaN === NaN`). When using object keys, the same reference must be used to retrieve the value:

```javascript
const map = new Map();
const obj = { id: 1 };

map.set(obj, "found me");
map.get(obj);           // "found me"
map.get({ id: 1 });     // undefined — different object reference

// NaN works as a key
map.set(NaN, "not a number");
map.get(NaN);           // "not a number"
```

### Map vs Object

```javascript
// Object — property keys coerced to strings
const obj = {};
obj[1] = "one";
Object.keys(obj); // ["1"] — number coerced to string
obj[true] = "bool";
Object.keys(obj); // ["1", "true"]

// Object inherits from Object.prototype — potential key collisions
obj["constructor"]; // [Function: Object] — inherited!

// Map — no coercion, no prototype pollution
const map = new Map();
map.set(1, "one");
map.set(true, "bool");
[...map.keys()]; // [1, true] — original types preserved
```

### Practical Patterns

```javascript
// Frequency counter
function frequencies(arr) {
  return arr.reduce((map, item) => {
    map.set(item, (map.get(item) ?? 0) + 1);
    return map;
  }, new Map());
}

frequencies(["a", "b", "a", "c", "b", "a"]);
// Map { "a" => 3, "b" => 2, "c" => 1 }

// Index objects by property
function indexBy(arr, key) {
  return new Map(arr.map(item => [item[key], item]));
}

const users = [{ id: 1, name: "Alice" }, { id: 2, name: "Bob" }];
const byId = indexBy(users, "id");
byId.get(1); // { id: 1, name: "Alice" }
```

## Set

### What is Set

`Set` stores unique values in insertion order. Any value type is accepted, with `SameValueZero` equality for uniqueness checks.

```javascript
const set = new Set([1, 2, 3, 2, 1]);
set.size; // 3 — duplicates removed

set.add(4);
set.has(2);   // true
set.delete(1);
set.size;     // 3

// Iterating
for (const v of set) console.log(v); // 2, 3, 4
[...set]; // [2, 3, 4]
```

### Set Operations (ES2025 — or manual)

ES2025 adds native set methods. Until then, implement manually:

```javascript
// Union
const union = (a, b) => new Set([...a, ...b]);

// Intersection
const intersection = (a, b) => new Set([...a].filter(x => b.has(x)));

// Difference (a - b)
const difference = (a, b) => new Set([...a].filter(x => !b.has(x)));

// Symmetric difference
const symmetricDiff = (a, b) => {
  const result = new Set(a);
  for (const x of b) {
    if (result.has(x)) result.delete(x);
    else result.add(x);
  }
  return result;
};

const a = new Set([1, 2, 3, 4]);
const b = new Set([3, 4, 5, 6]);

union(a, b);          // Set {1,2,3,4,5,6}
intersection(a, b);   // Set {3,4}
difference(a, b);     // Set {1,2}
symmetricDiff(a, b);  // Set {1,2,5,6}
```

### Native Set Methods (ES2025)

```javascript
const a = new Set([1, 2, 3]);
const b = new Set([2, 3, 4]);

a.union(b);           // Set {1,2,3,4}
a.intersection(b);    // Set {2,3}
a.difference(b);      // Set {1}
a.symmetricDifference(b); // Set {1,4}
a.isSubsetOf(b);      // false
a.isSupersetOf(b);    // false
a.isDisjointFrom(b);  // false
```

### Deduplication

```javascript
const dedup = arr => [...new Set(arr)];
dedup([1, 2, 1, 3, 2]); // [1, 2, 3]

// Object dedup by reference (not by value)
const items = [obj1, obj2, obj1];
const unique = [...new Set(items)]; // [obj1, obj2] — reference equality
```

## WeakMap

### What is WeakMap

`WeakMap` is a Map where keys must be objects (or non-registered symbols in ES2023+). The key references are **weak** — if no other reference to the key object exists, the garbage collector can collect it, and the entry is automatically removed from the WeakMap.

```javascript
let obj = { name: "Alice" };
const wm = new WeakMap();

wm.set(obj, { accessCount: 0, lastAccess: null });
wm.get(obj); // { accessCount: 0, lastAccess: null }
wm.has(obj); // true

obj = null; // Remove the only strong reference
// At some point, GC will collect the object and remove the entry
// wm.get(deadRef) would return undefined
```

### Why No Iteration

`WeakMap` deliberately has no `size`, `keys()`, `values()`, `entries()`, or `forEach`. If iteration were possible, you could observe garbage collection (by noticing entries disappear), which would create non-deterministic program behavior and defeat GC optimizations. The non-iteration constraint ensures consistency.

```javascript
const wm = new WeakMap();
wm.size;    // undefined
[...wm];    // TypeError: wm is not iterable
```

### Pattern: Private Data via WeakMap

Before ES2022 private fields, WeakMap was the standard approach for truly private instance data:

```javascript
const _private = new WeakMap();

class Widget {
  constructor(id, element) {
    _private.set(this, {
      id,
      element,
      handlers: []
    });
  }

  on(event, handler) {
    const data = _private.get(this);
    data.handlers.push({ event, handler });
    data.element.addEventListener(event, handler);
    return this;
  }

  destroy() {
    const data = _private.get(this);
    data.handlers.forEach(({ event, handler }) => {
      data.element.removeEventListener(event, handler);
    });
    _private.delete(this);
  }

  get id() {
    return _private.get(this).id;
  }
}
```

When the `Widget` instance is garbage collected, the WeakMap entry is too — no memory leak.

### Pattern: Caching / Memoization

```javascript
const cache = new WeakMap();

function processNode(node) {
  if (cache.has(node)) {
    return cache.get(node);
  }
  const result = heavyComputation(node);
  cache.set(node, result);
  return result;
}

// When DOM nodes are removed, cache entries are automatically freed
// No need to manually clean up — avoids memory leaks
```

### Pattern: Metadata Without Pollution

```javascript
const metadata = new WeakMap();

function annotate(obj, data) {
  metadata.set(obj, { ...metadata.get(obj), ...data });
}

function getAnnotation(obj) {
  return metadata.get(obj);
}

const user = { name: "Alice" };
annotate(user, { role: "admin", lastLogin: new Date() });
getAnnotation(user); // { role: "admin", lastLogin: ... }
// The metadata is not on the object — invisible to enumeration
```

## WeakSet

### What is WeakSet

`WeakSet` stores weakly-held object references. Like WeakMap, it holds weak references to its members, has no iteration, and no `size`. Membership can be tested with `has`.

```javascript
const seen = new WeakSet();

function processOnce(obj) {
  if (seen.has(obj)) {
    return false;
  }
  seen.add(obj);
  doWork(obj);
  return true;
}

// When obj is GC'd, it's automatically removed from seen
```

### Pattern: Preventing Circular Reference Traversal

```javascript
function deepClone(obj, visited = new WeakSet()) {
  if (obj === null || typeof obj !== "object") return obj;
  if (visited.has(obj)) return "[Circular]";

  visited.add(obj);

  const clone = Array.isArray(obj) ? [] : {};
  for (const key of Object.keys(obj)) {
    clone[key] = deepClone(obj[key], visited);
  }
  return clone;
}
```

### Pattern: Capability / Branding

```javascript
const authorized = new WeakSet();

function grantAccess(user) {
  authorized.add(user);
}

function restrictedAction(user) {
  if (!authorized.has(user)) {
    throw new Error("Unauthorized");
  }
  // Perform sensitive action
}
```

## WeakRef

### What is WeakRef

`WeakRef` (ES2021) wraps an object with a weak reference. The garbage collector may collect the referenced object if no strong references remain. The wrapped object is retrieved with `.deref()`, which returns `undefined` if it has been collected.

```javascript
let target = { data: "important" };
const ref = new WeakRef(target);

ref.deref(); // { data: "important" }

target = null; // Remove strong reference

// At some point after GC:
ref.deref(); // undefined — object was collected
```

### FinalizationRegistry

`FinalizationRegistry` runs a callback after an object is garbage collected — useful for cleanup, logging, or releasing external resources:

```javascript
const registry = new FinalizationRegistry((heldValue) => {
  console.log(`Object with id=${heldValue} was collected`);
  // Release associated resources (file handles, network connections, etc.)
});

function createManagedResource(id) {
  const resource = { id, data: new ArrayBuffer(1024 * 1024) };
  registry.register(resource, id); // Register with cleanup value
  return resource;
}

let r = createManagedResource(1);
r = null; // Eligible for GC
// Eventually: "Object with id=1 was collected"
```

### WeakRef Use Case: Caching With Expiration

```javascript
class Cache {
  #entries = new Map(); // Map<key, WeakRef<value>>

  set(key, value) {
    this.#entries.set(key, new WeakRef(value));
  }

  get(key) {
    const ref = this.#entries.get(key);
    if (!ref) return undefined;

    const value = ref.deref();
    if (value === undefined) {
      // Entry was GC'd — clean up stale Map entry
      this.#entries.delete(key);
      return undefined;
    }
    return value;
  }
}
```

### WeakRef Cautions

WeakRef should be used sparingly. Its behavior is non-deterministic (GC timing varies by engine and runtime conditions). Never write code that is correct only if a WeakRef is still alive — always handle the `undefined` case.

## Comparison Table

| Feature | Map | Set | WeakMap | WeakSet | WeakRef |
| --- | --- | --- | --- | --- | --- |
| Key/value or value | Both | Value only | Both (weak key) | Value only (weak) | Wrapped object |
| Key types | Any | N/A | Objects (+ unregistered symbols ES2023) | Objects | Objects |
| Strong references | Yes | Yes | Keys: No / Values: Yes | No | No |
| Size property | Yes | Yes | No | No | N/A |
| Iterable | Yes | Yes | No | No | No |
| Garbage collection | N/A | N/A | Keys collected | Members collected | Object collected |
| Use case | Any keyed data | Unique values | Private data, cache | Membership | Optional object ref |

## Best Practices

### 1. Use Map Instead of Object for Dynamic Key Sets

```javascript
// Bad — prototype collision risk, string coercion
const index = {};
index["constructor"] = data; // Overwrites Object.prototype.constructor!

// Good
const index = new Map();
index.set("constructor", data);
```

### 2. Use WeakMap for Object Metadata, Not WeakSet for Non-Membership Checks

WeakMap stores associated data. WeakSet only answers "is this object a member?" — it cannot store associated values.

### 3. Prefer class Private Fields Over WeakMap in ES2022+

Class `#fields` are cleaner, faster, and engine-optimized. WeakMap-based privacy is still useful for attaching data to objects you don't control (third-party instances, DOM nodes, etc.).

### 4. Never Rely on WeakRef Timing

If code correctness depends on an object still being alive via `WeakRef`, you have a design problem. `WeakRef` is for optional optimization (caches) and cleanup registration, not for keeping objects alive.

## Interview Questions

### Q1: Why can't WeakMap and WeakSet be iterated?

**Answer:** Iteration would require enumerating the set of live entries. But because the GC can collect entries at any time (between any two observable operations), the set of entries is non-deterministic from the program's perspective. Allowing iteration would mean the results could change between two sequential iterations in unpredictable ways, making GC behavior observable. This would force the engine to either never collect those objects (defeating the purpose) or produce non-deterministic code. By prohibiting iteration, the spec ensures that WeakMap/WeakSet are purely a performance and memory-management tool with no observable side effects.

### Q2: What is the key difference between Map and a plain object used as a dictionary?

**Answer:** Map allows any value as a key (objects, numbers, symbols) without coercion to strings, guarantees insertion-order iteration, has a reliable `size` property, does not inherit from `Object.prototype` (no key collision with `constructor`, `hasOwnProperty`, etc.), and has purpose-built iteration methods. Plain objects coerce all keys to strings or symbols, inherit prototype properties (potential collisions), have no guaranteed order for all key types, and no built-in `size`. Map is the correct choice for dynamic, programmatic key-value storage.

### Q3: What problem does WeakMap solve that a regular Map does not?

**Answer:** With a regular `Map`, keeping an entry prevents the key object from being garbage collected — the Map holds a strong reference to it. This causes memory leaks when the key objects are meant to be short-lived (e.g., DOM nodes, request contexts). `WeakMap` holds weak references to keys, allowing the GC to collect them when no strong references remain. The entry is automatically removed. This makes WeakMap appropriate for associating metadata or private state with objects without causing memory leaks.

### Q4: When would you choose WeakRef over WeakMap?

**Answer:** WeakMap stores associated data keyed on an object. WeakRef is when you want to optionally hold a reference to an object itself — for example, a cache of computed objects where you want to reuse an existing instance if it hasn't been GC'd, but create a new one if it has. WeakRef is also used with `FinalizationRegistry` for cleanup callbacks. If you need to associate metadata with an object, use WeakMap. If you need an optional handle to the object itself, use WeakRef.

### Q5: How does Set handle equality for complex values?

**Answer:** Set uses SameValueZero equality — essentially `===` with the special case that `NaN` is equal to itself. This means two different object references are always distinct entries even if they are deeply equal in content. Two primitive values are deduplicated only if they are strictly equal. There is no way to customize Set's equality algorithm (unlike Java's `equals`/`hashCode`). For content-based deduplication of objects, you would serialize to a string key and use a Map instead.

## Common Pitfalls

### 1. Expecting Object Equality in Set/Map

```javascript
const set = new Set();
set.add({ id: 1 });
set.add({ id: 1 });
set.size; // 2 — different references, not equal by SameValueZero
```

### 2. Converting Map to Object Naively

```javascript
const map = new Map([["a", 1], [42, 2]]);

// Bad — Object.fromEntries coerces keys to strings
Object.fromEntries(map); // { a: 1, "42": 2 } — number key coerced

// If you need string keys, ensure all keys are strings before converting
```

### 3. Using WeakRef When a Strong Reference Is Needed

```javascript
// Bug — registerCallback may fire after object is GC'd
function watch(element) {
  const ref = new WeakRef(element);
  setInterval(() => {
    const el = ref.deref();
    if (el) el.update(); // Fine — el may be undefined
  }, 100);
}
```

If the element is supposed to stay alive for the duration of the interval, hold a strong reference. WeakRef is not a replacement for normal references.

### 4. Manually Clearing Map as a "WeakMap"

```javascript
// Anti-pattern — developers sometimes do this
const cache = new Map();
window.addEventListener("beforeunload", () => cache.clear());

// Better: if keys are objects, use WeakMap — entries self-clean
// If keys are strings/primitives, manage cache lifecycle explicitly
```

## Resources

### Official Documentation

- [MDN: Map](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map)
- [MDN: Set](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set)
- [MDN: WeakMap](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakMap)
- [MDN: WeakSet](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakSet)
- [MDN: WeakRef](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakRef)
- [TC39: Set Methods Proposal](https://github.com/tc39/proposal-set-methods)
- [TC39: WeakRefs and FinalizationRegistry](https://github.com/tc39/proposal-weakrefs)

### Articles and Guides

- [JavaScript Collections In-Depth (2ality)](https://2ality.com/2015/01/es6-maps-sets.html)
- [WeakMap and WeakSet (javascript.info)](https://javascript.info/weakmap-weakset)
- [WeakRef and FinalizationRegistry (V8 Blog)](https://v8.dev/features/weak-references)
- [When to Use Map vs Object (MDN)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map#objects_vs._maps)
