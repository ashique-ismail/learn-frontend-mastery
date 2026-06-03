# Pure Functions and Side Effects

## Overview

Pure functions are the foundation of functional programming in JavaScript. A function is pure if, given the same inputs, it always returns the same output (referential transparency), and it has no observable side effects on state outside its own scope. Side effects — mutations of external state, I/O operations, network calls, DOM manipulation — are not inherently wrong, but isolating them from pure logic makes programs dramatically easier to reason about, test, and optimize. Senior engineers are expected not just to define pure functions, but to architect systems where side effects are contained, explicit, and manageable.

## What Is a Pure Function

### Formal Definition

A function `f` is pure if:

1. **Deterministic**: For the same inputs, it always returns the same output.
2. **No side effects**: It does not mutate external state, perform I/O, or interact with anything outside its own scope.

```javascript
// Pure
function add(a, b) {
  return a + b;
}

add(2, 3); // 5 — always
add(2, 3); // 5 — always, regardless of when or how many times called

// Impure — depends on external state
let multiplier = 2;
function scale(x) {
  return x * multiplier; // result changes if multiplier changes
}

// Impure — has a side effect (mutation of external variable)
let callCount = 0;
function impureAdd(a, b) {
  callCount++; // side effect
  return a + b;
}
```

### Referential Transparency

A pure function call can be replaced with its return value without changing program behavior. This property is called referential transparency.

```javascript
// Referentially transparent
function square(x) { return x * x; }

const result = square(5) + square(5);
// Can be replaced with: 25 + 25
// Both expressions are equivalent

// NOT referentially transparent
let n = 0;
function increment() { return ++n; }

const r = increment() + increment();
// First call returns 1, second returns 2 — result is 3
// Cannot replace with any fixed value
```

## Side Effects: A Catalog

Understanding what constitutes a side effect is as important as knowing how to avoid them.

### Mutating External State

```javascript
// Side effect: mutating a variable in outer scope
let total = 0;
function addToTotal(n) {
  total += n; // side effect
  return total;
}

// Side effect: mutating an input argument
function appendItem(arr, item) {
  arr.push(item); // mutates the caller's array
  return arr;
}

// Pure version: return a new array
function appendItemPure(arr, item) {
  return [...arr, item];
}
```

### Mutating Object Properties

```javascript
// Impure — mutates the user object
function activateUser(user) {
  user.active = true; // side effect
  return user;
}

// Pure — returns a new object
function activateUserPure(user) {
  return { ...user, active: true };
}

// Deep copy when nested objects are involved
function updateAddress(user, newAddress) {
  return {
    ...user,
    address: { ...user.address, ...newAddress }
  };
}
```

### I/O Operations

```javascript
// Impure — console.log is a side effect (I/O)
function add(a, b) {
  console.log(`Adding ${a} + ${b}`); // side effect
  return a + b;
}

// I/O operations are inherently impure:
// - console.log / console.error
// - localStorage.getItem / setItem
// - document.querySelector, element.style.color = ...
// - fetch(), XMLHttpRequest
// - fs.readFile(), fs.writeFile()
// - process.exit()
```

### Randomness and Time

```javascript
// Impure — depends on current time
function getTimestamp() {
  return Date.now(); // different every call
}

// Impure — random
function rollDie() {
  return Math.floor(Math.random() * 6) + 1; // unpredictable
}

// Making impure code testable: inject the dependency
function createTimestampedRecord(data, getTime = Date.now) {
  return { ...data, createdAt: getTime() };
}

// In tests
createTimestampedRecord({ name: "Alice" }, () => 0);
// { name: "Alice", createdAt: 0 } — deterministic in tests
```

### Throwing Exceptions

```javascript
// Impure — throws can alter control flow (observable side effect)
function divide(a, b) {
  if (b === 0) throw new Error("Division by zero");
  return a / b;
}

// Pure alternative — return a result type
function safeDiv(a, b) {
  if (b === 0) return { ok: false, error: "Division by zero" };
  return { ok: true, value: a / b };
}

// Or use a Maybe/Option pattern
function safeDivOption(a, b) {
  return b === 0 ? null : a / b;
}
```

## Pure Functions in Practice

### Working with Arrays

```javascript
const items = [3, 1, 4, 1, 5, 9, 2, 6];

// Pure — returns new array, original unchanged
const sorted     = [...items].sort((a, b) => a - b);
const reversed   = [...items].reverse(); // reverse is mutating — spread first
const doubled    = items.map(n => n * 2);
const evens      = items.filter(n => n % 2 === 0);
const sum        = items.reduce((acc, n) => acc + n, 0);
const unique     = [...new Set(items)];
const withoutOne = items.filter(n => n !== 1);

// Array.prototype methods that MUTATE (impure in terms of original array):
// push, pop, shift, unshift, splice, sort, reverse, fill, copyWithin

// Their pure equivalents:
const append  = (arr, item) => [...arr, item];
const prepend = (arr, item) => [item, ...arr];
const remove  = (arr, index) => [...arr.slice(0, index), ...arr.slice(index + 1)];
const insert  = (arr, index, item) => [
  ...arr.slice(0, index),
  item,
  ...arr.slice(index)
];
const update  = (arr, index, item) => arr.map((el, i) => i === index ? item : el);
```

### Working with Objects

```javascript
const user = { id: 1, name: "Alice", role: "user", settings: { theme: "dark" } };

// Pure object updates — always return new objects
const withRole  = { ...user, role: "admin" };
const withTheme = { ...user, settings: { ...user.settings, theme: "light" } };

// Deep clone for deeply nested structures (use structuredClone in modern environments)
const deepCopy = structuredClone(user);

// Pure delete
const { role, ...userWithoutRole } = user;

// Immutable update patterns
function setIn(obj, path, value) {
  const keys = path.split(".");
  if (keys.length === 1) return { ...obj, [path]: value };

  const [head, ...tail] = keys;
  return {
    ...obj,
    [head]: setIn(obj[head] ?? {}, tail.join("."), value)
  };
}

setIn(user, "settings.theme", "light");
// { id:1, name:"Alice", ..., settings: { theme: "light" } }
```

### Pure Transformations Pipeline

```javascript
// Each step is pure — no mutation, no side effects
const processOrders = orders =>
  orders
    .filter(o => o.status === "paid")
    .map(o => ({ ...o, total: o.items.reduce((s, i) => s + i.price, 0) }))
    .filter(o => o.total > 0)
    .sort((a, b) => b.total - a.total)
    .slice(0, 10);

// All pure: filter, map, reduce, sort (on copy), slice
```

## Referential Transparency and Memoization

Because pure functions always return the same output for the same input, their results can be safely cached.

```javascript
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

// Only works correctly if fn is pure
const fibonacci = memoize(function fib(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
});

fibonacci(40); // computed once and cached — O(n) instead of O(2^n)

// Memoizing an impure function gives wrong results
const getTimeMemoized = memoize(() => Date.now());
getTimeMemoized(); // 1704067200000
getTimeMemoized(); // 1704067200000 — wrong, but that's what memoize does
```

## Isolating Side Effects

Side effects are unavoidable — real programs need I/O, network calls, and DOM manipulation. The strategy is to isolate side effects at the boundaries of the system and keep the core logic pure.

### The Functional Core / Imperative Shell Pattern

```javascript
// PURE CORE — business logic with no side effects
function calculateDiscount(cart, discountRules) {
  return discountRules.reduce((total, rule) => {
    return rule.applies(cart) ? total * (1 - rule.rate) : total;
  }, cart.subtotal);
}

function buildInvoice(cart, discount, tax) {
  return {
    items: cart.items,
    subtotal: cart.subtotal,
    discount,
    tax: (cart.subtotal - discount) * tax,
    total: cart.subtotal - discount + (cart.subtotal - discount) * tax
  };
}

// IMPERATIVE SHELL — side effects at the edges
async function checkoutHandler(cartId, userId) {
  // Side effect: I/O
  const cart = await db.carts.findById(cartId);
  const rules = await db.discountRules.findActiveForUser(userId);
  const taxRate = await taxService.getRate(cart.shippingAddress);

  // Pure core
  const discount = calculateDiscount(cart, rules);
  const invoice  = buildInvoice(cart, discount, taxRate);

  // Side effect: I/O
  await db.invoices.create(invoice);
  await emailService.send(userId, invoice);

  return invoice;
}
```

### Dependency Injection for Testability

```javascript
// Impure — hardwired side effects
function saveUser(user) {
  const id = uuid(); // impure
  const record = { ...user, id, createdAt: Date.now() }; // impure
  database.save(record); // side effect
  return record;
}

// Pure core + injected dependencies
function createUserRecord(user, { generateId, getTimestamp }) {
  return {
    ...user,
    id: generateId(),
    createdAt: getTimestamp()
  };
}

// Shell: side effects isolated
async function saveUser(user, deps = {
  generateId: () => crypto.randomUUID(),
  getTimestamp: Date.now,
  save: db.users.create.bind(db.users)
}) {
  const record = createUserRecord(user, deps);
  await deps.save(record);
  return record;
}

// In tests — fully deterministic
const testRecord = createUserRecord(
  { name: "Alice" },
  { generateId: () => "test-id", getTimestamp: () => 0 }
);
// { name: "Alice", id: "test-id", createdAt: 0 }
```

## Testing Pure Functions

Pure functions are trivially testable: no mocks, no stubs, no setup, no teardown.

```javascript
// Pure — test is straightforward
function slugify(str) {
  return str
    .trim()
    .toLowerCase()
    .replace(/[^a-z0-9\s-]/g, "")
    .replace(/\s+/g, "-");
}

// Tests: input → output, no mocking needed
assert.equal(slugify("Hello World"),      "hello-world");
assert.equal(slugify("  Trim Me  "),      "trim-me");
assert.equal(slugify("Café au lait"),     "caf-au-lait");
assert.equal(slugify("Multiple   Spaces"), "multiple-spaces");

// Compare to impure version that touches DOM or calls fetch — requires mocks
```

## Comparison Table

| Property | Pure Function | Impure Function |
|---|---|---|
| Same inputs → same output | Always | Not guaranteed |
| Modifies external state | Never | May do so |
| Safe to memoize | Yes | Only if deterministic |
| Easy to test | Yes — no setup | Requires mocks/stubs |
| Parallelizable | Yes | Race conditions possible |
| Referencially transparent | Yes | No |
| Composable | Freely | With care |
| Predictable in isolation | Yes | Depends on context |

## Best Practices

### 1. Treat Input Data as Read-Only

```javascript
// Bad — mutates input
function sortUsers(users) {
  return users.sort((a, b) => a.name.localeCompare(b.name)); // mutates original
}

// Good — copy first
function sortUsers(users) {
  return [...users].sort((a, b) => a.name.localeCompare(b.name));
}
```

### 2. Return New Objects Instead of Mutating

```javascript
// Bad
function setActive(user) {
  user.active = true;
  return user;
}

// Good
function setActive(user) {
  return { ...user, active: true };
}
```

### 3. Push Side Effects to the Edges

Move logging, persistence, network calls, and randomness to the outermost layer of the call stack. Pure functions handle the computation; the shell handles the I/O.

### 4. Use const Aggressively

`const` does not make objects immutable, but it signals intent and prevents reassignment. Combine with `Object.freeze` for shallow immutability.

```javascript
const config = Object.freeze({
  apiUrl: "https://api.example.com",
  timeout: 5000
});

// config.timeout = 10000; // silently fails (or throws in strict mode)
```

### 5. Prefer Expressions Over Statements

Expressions produce values without side effects. Statements often imply mutation or control flow.

```javascript
// Statement style (imperative)
let result;
if (condition) {
  result = "a";
} else {
  result = "b";
}

// Expression style (more functional)
const result = condition ? "a" : "b";

// Conditional rendering in arrays
const items = [
  isAdmin && adminItem,
  userItem
].filter(Boolean);
```

### 6. Document Side Effects Explicitly

When a function must have side effects, make it obvious in the name or documentation.

```javascript
// Name signals side effect
async function fetchAndCacheUser(id) { /* ... */ }
function logAndReturn(label, value) { console.log(label, value); return value; }
function updateDOMAndNotify(el, msg) { /* ... */ }
```

## Interview Questions

**Q: Define a pure function. Give an example of a function that looks pure but is not.**

A: A pure function always returns the same output for the same inputs and produces no observable side effects. A function that looks pure but is not: `function first(arr) { return arr[0]; }` — this is actually pure. A deceptive example: `function getUser(id) { return users[id]; }` — if `users` is a variable from outer scope, the result changes if `users` is mutated elsewhere, making it impure despite having no visible side effect.

**Q: What is referential transparency, and why does it matter?**

A: Referential transparency means a function call can be replaced with its return value without changing program semantics. It matters because: it enables safe memoization (the cached value is equivalent to recomputing), it makes reasoning easier (no hidden state to track), it enables compiler optimizations (dead code elimination, constant folding), and it simplifies testing (no environment setup required).

**Q: Is `Array.prototype.sort` pure? What about `Array.prototype.map`?**

A: `sort` is not pure — it sorts the array in place, mutating the original. `map` is pure with respect to the original array — it returns a new array and does not mutate the source. However, `map` will propagate any side effects in the callback, so `arr.map(fn)` is only as pure as `fn`.

**Q: How would you refactor an impure function to be testable without fully purifying it?**

A: Use dependency injection to pass in the impure dependencies (e.g., `Date.now`, `Math.random`, `fetch`, `db.save`) as parameters with production defaults. The core logic becomes a pure function of its inputs including the injected functions. Tests supply deterministic stubs for those dependencies, giving full control without mocking the module system.

**Q: What is the "functional core, imperative shell" pattern?**

A: It is an architecture strategy where pure functions handle all business logic and computation (the core), and a thin outer layer handles all I/O, side effects, and orchestration (the shell). The shell reads from external sources, passes pure data to the core, then writes the core's output back to external sinks. This maximizes testability — the core needs no mocks, and the shell is so thin it barely needs testing.

**Q: Can a function that throws an exception be considered pure?**

A: Technically no, because throwing an exception is an observable side effect — it alters control flow in a way that is not reflected in the return value. Pure functions handle errors by returning a value representing failure (e.g., `null`, a Result/Either type, or a tagged union) rather than throwing. However, many practitioners accept throwing as a "pure enough" behavior in JavaScript since the language does not have a native Result type, and properly threading error types through every call site is impractical without type system support.

## Common Pitfalls

### 1. sort Mutates In Place

```javascript
const nums = [3, 1, 2];
const sorted = nums.sort(); // mutates nums!
console.log(nums); // [1, 2, 3] — original changed

// Fix: copy before sorting
const sorted = [...nums].sort();
```

### 2. Shallow Spread Does Not Deep Copy

```javascript
const user = { name: "Alice", prefs: { theme: "dark" } };
const copy = { ...user };

copy.prefs.theme = "light"; // mutates original user.prefs!
console.log(user.prefs.theme); // "light" — oops

// Fix: spread nested objects too
const copy = { ...user, prefs: { ...user.prefs } };
// Or: structuredClone(user) for arbitrary depth
```

### 3. Relying on Object Identity in "Pure" Functions

```javascript
// Looks pure — same inputs, same output?
function merge(a, b) {
  return { ...a, ...b };
}

merge({ x: 1 }, { y: 2 }); // { x: 1, y: 2 }
merge({ x: 1 }, { y: 2 }); // { x: 1, y: 2 } — same value, different reference

// If downstream code uses === to compare results, it will see "changes"
// even when values are identical — relevant for React, Redux, memoization
```

### 4. Forgetting That Closures Capture References, Not Values

```javascript
function makeAdder(x) {
  return function(y) {
    return x + y; // 'x' is captured by reference
  };
}

// This is fine because 'x' is immutable (const/parameter)

// But this is a trap:
let base = 10;
const addBase = n => n + base; // captures 'base' by reference

addBase(5); // 15
base = 20;
addBase(5); // 25 — result changed even though inputs to addBase didn't
// addBase is NOT pure — it depends on external mutable state
```

### 5. Mutating Through Array/Object References in reduce

```javascript
// Bug: accumulator is mutated but that's the same object being passed forward
// This is actually OK here since the acc is the object we own and returned
const result = items.reduce((acc, item) => {
  acc[item.id] = item; // mutation of accumulator — generally avoid
  return acc;
}, {});

// Immutable style — creates new object each step (slower for large collections)
const result = items.reduce((acc, item) => ({
  ...acc,
  [item.id]: item
}), {});

// For performance-sensitive code, the mutating version is acceptable
// since the accumulator is not shared outside reduce
```

## Resources

### Documentation
- [MDN: Array — Mutating vs Non-Mutating Methods](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array)
- [MDN: structuredClone](https://developer.mozilla.org/en-US/docs/Web/API/structuredClone)
- [MDN: Object.freeze](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/freeze)

### Books and Articles
- [Professor Frisby's Mostly Adequate Guide to FP in JavaScript — Chapter 3](https://mostly-adequate.gitbook.io/mostly-adequate-guide/ch03)
- [Eric Elliott: What is a Pure Function?](https://medium.com/javascript-scene/master-the-javascript-interview-what-is-a-pure-function-d1c076bec976)
- [Gary Bernhardt: Functional Core, Imperative Shell](https://www.destroyallsoftware.com/screencasts/catalog/functional-core-imperative-shell)
- [JavaScript.info: Closures](https://javascript.info/closure)
