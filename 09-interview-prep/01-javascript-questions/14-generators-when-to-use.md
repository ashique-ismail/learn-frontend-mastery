# Generators & WeakMap/WeakSet Use Cases

## The Idea

**In plain English:** A generator is a special kind of function that can pause itself in the middle and hand you one result at a time, instead of computing everything all at once. A WeakMap and WeakSet are like normal collections that hold objects, except they automatically forget an item the moment nothing else in your program is using it anymore.

**Real-world analogy:** Imagine a vending machine that dispenses one snack each time you press a button, and you can press as many or as few times as you want — it does not load up every snack in advance.

- The button press = calling `.next()` on a generator
- The machine pausing between presses = the `yield` keyword stopping execution
- The snack that comes out = the value the generator hands back to you

---

## Generators

### What They Are

Generator functions return an **iterator** — an object that produces values on demand, pausing execution between each `yield`.

```js
function* count(start = 0) {
  while (true) {
    yield start++;
  }
}

const counter = count(10);
counter.next(); // { value: 10, done: false }
counter.next(); // { value: 11, done: false }
counter.next(); // { value: 12, done: false }
// Never done — infinite sequence with zero memory overhead
```

### Syntax

```js
function* generator() {
  const x = yield 'first';   // yields 'first', pauses
                              // x = value passed to next(value)
  yield 'second';
  return 'done';             // { value: 'done', done: true }
}

const gen = generator();
gen.next();          // { value: 'first', done: false }
gen.next('hello');   // x = 'hello', { value: 'second', done: false }
gen.next();          // { value: 'done', done: true }
gen.next();          // { value: undefined, done: true }
```

### `yield*` — Delegate to Another Iterable

```js
function* flatten(arr) {
  for (const item of arr) {
    if (Array.isArray(item)) yield* flatten(item);
    else yield item;
  }
}

[...flatten([1, [2, [3, 4]], 5])]; // [1, 2, 3, 4, 5]
```

---

## Real Use Cases

### 1. Infinite Sequences

```js
function* fibonacci() {
  let [a, b] = [0, 1];
  while (true) {
    yield a;
    [a, b] = [b, a + b];
  }
}

function take(n, iterable) {
  const result = [];
  for (const x of iterable) {
    result.push(x);
    if (result.length === n) return result;
  }
}

take(8, fibonacci()); // [0, 1, 1, 2, 3, 5, 8, 13]
```

### 2. Lazy Transformations (No Intermediate Arrays)

```js
function* map(iterable, fn) {
  for (const x of iterable) yield fn(x);
}

function* filter(iterable, pred) {
  for (const x of iterable) if (pred(x)) yield x;
}

function* take(n, iterable) {
  let i = 0;
  for (const x of iterable) {
    if (i++ >= n) break;
    yield x;
  }
}

// Process 1 million items, only materialise the 5 we need
const result = [...take(5, filter(map(bigDataSource(), x => x * 2), x => x > 10))];
```

### 3. Custom Iterators

```js
class Range {
  constructor(start, end) { this.start = start; this.end = end; }
  [Symbol.iterator]() {
    let current = this.start;
    const end = this.end;
    return {
      next() {
        return current <= end
          ? { value: current++, done: false }
          : { value: undefined, done: true };
      }
    };
  }
}

[...new Range(1, 5)]; // [1, 2, 3, 4, 5]
for (const n of new Range(1, 3)) console.log(n); // 1, 2, 3
```

### 4. Async Control Flow (Legacy — now replaced by async/await)

Libraries like `co` used generators to write async code before async/await existed. Understanding this helps explain WHY async/await was designed the way it was.

---

## WeakMap Use Cases

`WeakMap` holds **weak** references to keys — keys must be objects, and they don't prevent garbage collection.

### 1. Private Data for Class Instances

```js
const _private = new WeakMap();

class Account {
  constructor(balance) {
    _private.set(this, { balance });
  }
  deposit(amount) {
    _private.get(this).balance += amount;
  }
  get balance() {
    return _private.get(this).balance;
  }
}

const acc = new Account(100);
acc.balance; // 100
// acc._private → undefined — truly private
// When acc is GC'd, _private entry is also GC'd
```

### 2. DOM Node Metadata (No Memory Leak)

```js
const metadata = new WeakMap();

document.querySelectorAll('button').forEach(btn => {
  metadata.set(btn, { clickCount: 0, lastClicked: null });
});

document.addEventListener('click', (e) => {
  const data = metadata.get(e.target);
  if (data) {
    data.clickCount++;
    // When button is removed from DOM, metadata is GC'd automatically
  }
});
```

### 3. Memoization Without Memory Leaks

```js
const cache = new WeakMap();

function processElement(el) {
  if (cache.has(el)) return cache.get(el);
  const result = expensiveComputation(el);
  cache.set(el, result);
  return result;
}
// When el is removed from DOM, cache entry is GC'd
```

---

## WeakSet Use Cases

`WeakSet` stores **weak** object references, with only `has`, `add`, `delete`.

### Tracking Visited Objects (No Memory Leak)

```js
const visited = new WeakSet();

function deepProcess(obj) {
  if (visited.has(obj)) return; // circular reference guard
  visited.add(obj);
  // ... process
  for (const key of Object.keys(obj)) {
    if (typeof obj[key] === 'object') deepProcess(obj[key]);
  }
}
```

### Branding / Tagging Objects

```js
const initialized = new WeakSet();

class Component {
  init() { initialized.add(this); }
  render() {
    if (!initialized.has(this)) throw new Error('Not initialized');
    // ...
  }
}
```

---

## Common Interview Questions

**Q: When would you use a generator over async/await?**
Generators are lower-level. Use them for lazy sequences, infinite iterators, and custom iteration protocols. For async control flow, async/await is simpler. Generators are still relevant in Redux-Saga (which models side effects as generator-based sagas).

**Q: Why can't WeakMap keys be primitives?**
Weak references only make sense for objects (which are heap-allocated and can be GC'd). Primitives are value types — there's no object identity to track.

**Q: What's the practical difference between Map and WeakMap?**
`Map` holds strong references (prevents GC), can have any key type, is iterable. `WeakMap` holds weak references (allows GC when key is unreachable), only object keys, not iterable (you can't enumerate entries — by design, since entries may vanish).
