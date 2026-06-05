# Deep Clone vs Shallow Clone

## The Idea

**In plain English:** When you copy an object in JavaScript, you can either make a "surface copy" (shallow clone) that shares inner parts with the original, or a fully independent copy (deep clone) where nothing is shared. Changing a deep clone never affects the original.

**Real-world analogy:** Imagine photocopying a binder of documents. A shallow copy is like copying only the binder's cover page and then putting sticky notes pointing to the original pages inside — the pages themselves are shared. A deep copy is like photocopying every single page inside too, so you end up with a completely separate binder.

- The binder cover = the top-level object
- The sticky notes pointing to original pages = references to nested objects
- Photocopying every page = deep cloning all nested values

---

## Why It Matters

Mutation bugs from unintended reference sharing are one of the most common JavaScript bugs. Interviewers use this question to test object model understanding and practical cloning knowledge.

---

## Shallow Copy

Copies top-level properties only. Nested objects/arrays still share references.

```js
const original = { a: 1, nested: { b: 2 } };

// All three are shallow:
const c1 = Object.assign({}, original);
const c2 = { ...original };
const c3 = Array.from(arr);  // for arrays

c1.a = 99;          // safe — original.a unchanged
c1.nested.b = 99;   // MUTATES original.nested.b too
```

**When shallow is fine:** Immutable primitives at top level, or when you intend to share nested references (Redux reducers using spread).

---

## Deep Clone Options

### 1. `structuredClone()` — Modern Standard (ES2022)

```js
const clone = structuredClone(original);
```

**Handles:** nested objects, arrays, `Date`, `Map`, `Set`, `RegExp`, `ArrayBuffer`, circular references.

**Does NOT handle:** functions, DOM nodes, class instances (loses prototype), `undefined` values in objects (kept), `Symbol` keys (dropped).

```js
const data = {
  date: new Date(),
  map: new Map([['a', 1]]),
  nested: { arr: [1, 2, 3] },
};
const clone = structuredClone(data);
// All deeply cloned, date is a real Date object
```

### 2. `JSON.parse(JSON.stringify(obj))` — Fast but Lossy

```js
const clone = JSON.parse(JSON.stringify(obj));
```

**Loses:** `undefined`, functions, `Date` (becomes string), `Map`/`Set` (become `{}`/`[]`), `Symbol` keys, circular references (throws).

**Use when:** data is pure JSON (numbers, strings, booleans, plain objects, arrays) and you know the shape.

### 3. Recursive Implementation

```js
function deepClone(value, seen = new WeakMap()) {
  if (value === null || typeof value !== 'object') return value;
  if (seen.has(value)) return seen.get(value);   // handle circular refs

  if (value instanceof Date) return new Date(value);
  if (value instanceof RegExp) return new RegExp(value);
  if (value instanceof Map) {
    const clone = new Map();
    seen.set(value, clone);
    value.forEach((v, k) => clone.set(deepClone(k, seen), deepClone(v, seen)));
    return clone;
  }
  if (value instanceof Set) {
    const clone = new Set();
    seen.set(value, clone);
    value.forEach(v => clone.add(deepClone(v, seen)));
    return clone;
  }

  const clone = Array.isArray(value) ? [] : Object.create(Object.getPrototypeOf(value));
  seen.set(value, clone);
  for (const key of Reflect.ownKeys(value)) {
    clone[key] = deepClone(value[key], seen);
  }
  return clone;
}
```

Key implementation details:
- `WeakMap` for seen objects handles circular references and is GC-friendly
- `Reflect.ownKeys` catches Symbol keys
- `Object.getPrototypeOf` preserves prototype chain

### 4. lodash `_.cloneDeep()`

Production-ready, handles edge cases. Add as a dependency when you need it rather than maintaining your own.

---

## Comparison Table

| Method | Deep | Circular Refs | Date/Map/Set | Functions | Speed |
|--------|------|---------------|--------------|-----------|-------|
| Spread / Object.assign | No | — | — | Yes | Fastest |
| JSON.parse/stringify | Yes | No (throws) | No (lossy) | No | Fast |
| structuredClone | Yes | Yes | Yes | No | Medium |
| Recursive custom | Yes | Yes (with WeakMap) | Depends on impl | No | Slow |
| lodash cloneDeep | Yes | Yes | Yes | No | Medium |

---

## Common Interview Questions

**Q: When would you use `structuredClone` vs JSON.parse/stringify?**
Use `structuredClone` by default for modern environments. Use JSON round-trip only for pure JSON data where you need maximum compatibility or want to deliberately drop non-JSON values.

**Q: Why does `const clone = {...obj}` not deep clone?**
Spread copies own enumerable properties one level. Property values that are objects are copied by reference, so both original and clone point to the same nested object.

**Q: How do you handle circular references in a recursive clone?**
Use a `WeakMap` as a "seen" cache: before processing an object, check if it's in the map. If yes, return the already-cloned version. If no, add the new clone to the map before recursing.

**Q: What does `structuredClone` throw on?**
It throws a `DataCloneError` for: functions, DOM nodes (in most environments), and class instances with methods that can't be serialized.
