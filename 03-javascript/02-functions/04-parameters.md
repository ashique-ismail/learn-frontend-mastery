# Parameters: Default, Rest, and Destructured

## The Idea

**In plain English:** Parameters are the named slots in a function that receive values when you call it. JavaScript gives you three powerful upgrades: you can set a fallback value if nothing is passed (default), collect any number of extra values into a list (rest), and unpack specific pieces from an object or array right inside the function's signature (destructured).

**Real-world analogy:** Imagine ordering a custom sandwich at a deli counter. You hand the server a printed order card that says: "Name: Turkey, Bread: Sourdough (default: white), Extras: pickles, olives, jalapenos."

- The named fields on the card (Name, Bread) = named parameters
- The pre-filled "(default: white)" = a default parameter (used only when you leave Bread blank)
- The open-ended "Extras" line where you can list as many toppings as you want = a rest parameter (collects everything remaining)
- The server reading specific fields off the card by name instead of guessing by position = destructured parameters

---

## Overview

JavaScript functions have evolved significantly in how they handle parameters. ES6 introduced default parameters, rest parameters, and destructuring assignment, replacing verbose ES5 patterns with clean, expressive syntax. Mastering these features is essential for writing idiomatic modern JavaScript and is a frequent topic in senior-level interviews, where examiners assess not only syntax knowledge but understanding of edge cases and performance implications.

## Default Parameters

### Syntax

```javascript
function greet(name = "World", greeting = "Hello") {
  return `${greeting}, ${name}!`;
}

console.log(greet());              // "Hello, World!"
console.log(greet("Alice"));      // "Hello, Alice!"
console.log(greet("Bob", "Hi"));  // "Hi, Bob!"
```

### How Defaults Are Evaluated

Default parameter values are evaluated at call time, not at function definition time. This is a critical distinction.

```javascript
// Each call gets a fresh array — evaluated at call time
function addItem(item, list = []) {
  list.push(item);
  return list;
}

console.log(addItem("a")); // ["a"]
console.log(addItem("b")); // ["b"] — not ["a", "b"]

// Compare to the ES5 anti-pattern:
function addItemBad(item, list) {
  list = list || []; // list is shared across calls if passed as reference
  list.push(item);
  return list;
}
```

### Defaults Can Reference Earlier Parameters

```javascript
function createRange(start, end = start + 10, step = 1) {
  const result = [];
  for (let i = start; i < end; i += step) {
    result.push(i);
  }
  return result;
}

console.log(createRange(0));         // [0, 1, 2, ..., 9]
console.log(createRange(0, 5));      // [0, 1, 2, 3, 4]
console.log(createRange(0, 10, 2));  // [0, 2, 4, 6, 8]
```

Parameters are available from left to right. Referencing a later parameter in an earlier default is a TDZ error.

```javascript
// TDZ error — b not yet initialized when a's default is evaluated
function bad(a = b, b = 1) {}  // ReferenceError
```

### Defaults Can Be Expressions or Function Calls

```javascript
function generateId() {
  return Math.random().toString(36).slice(2);
}

function createRecord(data, id = generateId(), timestamp = Date.now()) {
  return { id, timestamp, ...data };
}
```

### Triggering Defaults: undefined vs null

Defaults are only triggered by `undefined`, not `null`. This is a frequent interview question.

```javascript
function greet(name = "Guest") {
  return `Hello, ${name}`;
}

greet(undefined);  // "Hello, Guest" — default triggered
greet(null);       // "Hello, null"  — default NOT triggered
greet("");         // "Hello, "      — default NOT triggered
greet(0);          // "Hello, 0"     — default NOT triggered
```

### ES5 Equivalents (and Their Flaws)

```javascript
// Common ES5 pattern — broken for falsy values
function connect(host, port) {
  host = host || "localhost";
  port = port || 3000;
  return `${host}:${port}`;
}

connect("", 0);         // "localhost:3000" — wrong!

// Safer ES5 pattern
function connect(host, port) {
  host = (host !== undefined) ? host : "localhost";
  port = (port !== undefined) ? port : 3000;
  return `${host}:${port}`;
}

// ES6 default parameters handle this correctly
function connectES6(host = "localhost", port = 3000) {
  return `${host}:${port}`;
}

connectES6("", 0);  // ":0" — correct
```

## Rest Parameters

### Syntax

The rest parameter collects all remaining arguments into a real array.

```javascript
function sum(...numbers) {
  return numbers.reduce((acc, n) => acc + n, 0);
}

console.log(sum(1, 2, 3));        // 6
console.log(sum(1, 2, 3, 4, 5)); // 15
console.log(sum());               // 0
```

### Rest vs arguments Object

Rest parameters produce a real `Array` instance. The `arguments` object is array-like but not an array.

```javascript
function usingArguments() {
  // Must convert to array first
  const arr = Array.prototype.slice.call(arguments);
  // Or: const arr = Array.from(arguments);
  return arr.reduce((acc, n) => acc + n, 0);
}

function usingRest(...numbers) {
  // Already a real array — all array methods available
  return numbers.reduce((acc, n) => acc + n, 0);
}

// Arrow functions have no 'arguments' — rest is the only option
const sum = (...nums) => nums.reduce((a, b) => a + b, 0);
```

### Combining Named Parameters with Rest

```javascript
function log(level, ...messages) {
  const timestamp = new Date().toISOString();
  messages.forEach(msg => console.log(`[${timestamp}] [${level}] ${msg}`));
}

log("INFO", "Server started", "Listening on port 3000");
// [2024-01-01T00:00:00.000Z] [INFO] Server started
// [2024-01-01T00:00:00.000Z] [INFO] Listening on port 3000
```

### Rest Must Be Last

```javascript
// Error: Rest parameter must be last formal parameter
function bad(...args, last) {}  // SyntaxError

// Correct
function good(first, ...rest) {
  console.log(first); // first argument
  console.log(rest);  // array of remaining arguments
}
```

### Real-World Use Cases

```javascript
// Variadic function wrappers
function withLogging(fn, ...args) {
  console.log(`Calling ${fn.name} with`, args);
  const result = fn(...args);
  console.log(`Result:`, result);
  return result;
}

// Composing multiple event handlers
function combineHandlers(...handlers) {
  return function(event) {
    handlers.forEach(handler => handler(event));
  };
}

// Building APIs with optional trailing arguments
function createElement(tag, attrs = {}, ...children) {
  // similar to React.createElement
  return { tag, attrs, children };
}

createElement("div", { className: "container" }, child1, child2, child3);
```

## Destructured Parameters

### Object Destructuring in Parameters

```javascript
function displayUser({ name, age, role = "user" }) {
  console.log(`${name} (${age}) — ${role}`);
}

displayUser({ name: "Alice", age: 30 });              // "Alice (30) — user"
displayUser({ name: "Bob", age: 25, role: "admin" }); // "Bob (25) — admin"
```

### Renaming Destructured Parameters

```javascript
function connect({ host: hostname = "localhost", port: portNumber = 3000 }) {
  return `${hostname}:${portNumber}`;
}

connect({ host: "example.com", port: 443 }); // "example.com:443"
connect({});                                  // "localhost:3000"
```

### Nested Destructuring

```javascript
function processOrder({
  id,
  customer: { name, email },
  items: [firstItem, ...rest],
  metadata: { createdAt = Date.now() } = {}
}) {
  return {
    orderId: id,
    customerName: name,
    contactEmail: email,
    firstItem,
    remainingItems: rest,
    created: createdAt
  };
}
```

### Default for the Entire Parameter Object

Without a default for the whole object, calling the function with no argument throws an error.

```javascript
// Will throw if called with no argument
function greet({ name, greeting }) {
  return `${greeting}, ${name}!`;
}
greet(); // TypeError: Cannot destructure property 'name' of undefined

// Safe version — default the entire parameter to {}
function greet({ name = "World", greeting = "Hello" } = {}) {
  return `${greeting}, ${name}!`;
}
greet();  // "Hello, World!"
```

### Array Destructuring in Parameters

```javascript
function head([first, ...rest]) {
  return { first, rest };
}

function swap([a, b]) {
  return [b, a];
}

function useCoordinates([x = 0, y = 0, z = 0] = []) {
  return `(${x}, ${y}, ${z})`;
}

console.log(useCoordinates([1, 2]));  // "(1, 2, 0)"
console.log(useCoordinates());        // "(0, 0, 0)"
```

### Combining All Three Features

```javascript
function createRequest(
  url,
  {
    method = "GET",
    headers = {},
    body = null,
    timeout = 5000
  } = {},
  ...middlewares
) {
  return {
    url,
    options: { method, headers, body, timeout },
    middlewares
  };
}

createRequest(
  "https://api.example.com/users",
  { method: "POST", body: JSON.stringify({ name: "Alice" }) },
  authMiddleware,
  loggingMiddleware
);
```

## Comparison Table

| Feature | Default Parameters | Rest Parameters | Destructured Parameters |
|---|---|---|---|
| Purpose | Fallback values | Variadic arguments | Named/structured access |
| Triggered by | `undefined` only | Always collects | Structural matching |
| Return type | Individual values | `Array` instance | Extracted values |
| Position | Any position | Must be last | Any position |
| ES5 equivalent | `arg || default` | `arguments` object | Manual property access |
| Works with | Primitives, objects | Remaining args | Objects, arrays |

## Best Practices

### 1. Default the Entire Options Object

```javascript
// Fragile — breaks when called without argument
function init({ port, host }) { /* ... */ }

// Robust — safe to call with no argument
function init({ port = 3000, host = "localhost" } = {}) { /* ... */ }
```

### 2. Prefer Descriptive Parameter Objects Over Long Argument Lists

```javascript
// Hard to read at the call site — what does true mean?
function createUser("Alice", 30, true, false, "admin") { /* ... */ }

// Self-documenting call site
function createUser({ name, age, isActive = true, isVerified = false, role = "user" }) { /* ... */ }
createUser({ name: "Alice", age: 30, role: "admin" });
```

### 3. Use Rest for Variadic Functions, Not arguments

```javascript
// Avoid — not available in arrow functions, not a real array
function oldSum() {
  return [].reduce.call(arguments, (a, b) => a + b, 0);
}

// Prefer — works everywhere, real array
const sum = (...nums) => nums.reduce((a, b) => a + b, 0);
```

### 4. Document Parameter Shapes with TypeScript or JSDoc

```javascript
/**
 * @param {Object} options
 * @param {string} options.host
 * @param {number} [options.port=3000]
 * @param {boolean} [options.ssl=false]
 */
function connect({ host, port = 3000, ssl = false } = {}) { /* ... */ }
```

### 5. Avoid Side Effects in Default Expressions

```javascript
// Unpredictable — counter increments on each call
let callCount = 0;
function track(id = ++callCount) { /* ... */ }

// Better — keep defaults pure and predictable
function track(id = generateUUID()) { /* ... */ }
```

## Interview Questions

**Q: What is the difference between rest parameters and the `arguments` object?**

A: Rest parameters collect remaining function arguments into a real `Array` instance, supporting all array methods directly. The `arguments` object is array-like (has numeric indices and `length`) but is not an actual array — methods like `map`, `filter`, and `reduce` are unavailable without first converting it. Additionally, `arguments` is not available inside arrow functions, while rest parameters work everywhere. Rest parameters also only collect arguments beyond the named ones, whereas `arguments` captures all arguments regardless of named parameters.

**Q: When are default parameter values evaluated?**

A: Default values are evaluated each time the function is called and the parameter is `undefined` — not when the function is defined. This means default expressions like `[] ` or `{}` create new instances on each invocation, avoiding the shared-reference mutation problem common in languages where defaults are evaluated once at definition time.

**Q: Why does `greet(null)` not use the default value but `greet(undefined)` does?**

A: Default parameters are triggered exclusively by `undefined`. `null` is a deliberate "absence of value" and is a valid JavaScript value, so it passes through. `undefined` specifically signals "no value provided," which is why it triggers the default. This distinction matters when callers explicitly pass `null` to opt out of a value while still wanting other defaults to apply.

**Q: What happens when you destructure a parameter and the argument is not provided?**

A: Without a default for the entire parameter, the engine attempts to destructure `undefined`, throwing a `TypeError`. The fix is to provide a default for the whole parameter: `function f({ x } = {})`. This ensures that when no argument is given, destructuring runs on `{}` instead of `undefined`.

**Q: Can a default parameter reference another parameter?**

A: Yes, but only parameters that appear earlier in the list. Parameters are initialized left to right, and a default expression can reference any previously initialized parameter. Referencing a later parameter results in a `ReferenceError` because that parameter is in the temporal dead zone at evaluation time.

**Q: How would you write a function that accepts an unknown number of arguments and returns their average?**

```javascript
const average = (...nums) => {
  if (nums.length === 0) return 0;
  return nums.reduce((sum, n) => sum + n, 0) / nums.length;
};

console.log(average(1, 2, 3, 4, 5)); // 3
```

## Common Pitfalls

### 1. Mutable Objects as Defaults Are Safe in ES6, Not ES5

```javascript
// ES5 — dangerous: all calls share the same object
function bad(opts) {
  opts = opts || { retries: 3 }; // Same object reference!
}

// ES6 — safe: new object per call
function good(opts = { retries: 3 }) { /* fresh object each time */ }

// Even better — destructure defaults inline
function better({ retries = 3, timeout = 1000 } = {}) { /* ... */ }
```

### 2. Rest Parameter Position

```javascript
// SyntaxError — rest must be last
function wrong(a, ...b, c) {}

// Correct
function right(a, b, ...c) {}
```

### 3. Destructuring Without a Fallback

```javascript
function render({ title, body }) {
  return `<h1>${title}</h1><p>${body}</p>`;
}

render(); // TypeError: Cannot destructure property 'title' of undefined

// Fix
function render({ title = "Untitled", body = "" } = {}) { /* ... */ }
```

### 4. Using Default Parameters with Strict Mode

```javascript
// In non-strict mode, arguments reflects changes to named params
function sideEffect(x = 1) {
  "use strict"; // functions with defaults are always strict-mode
  arguments[0] = 99;
  console.log(x); // 1 — not 99. arguments is not linked in strict mode
}
```

### 5. Overusing Nested Destructuring

Deep destructuring is expressive but can make function signatures hard to read and refactor. Prefer a shallow destructure and handle nesting inside the function body when objects are deeply nested.

## Resources

### Documentation
- [MDN: Default Parameters](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Default_parameters)
- [MDN: Rest Parameters](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/rest_parameters)
- [MDN: Destructuring Assignment](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment)

### Articles
- [JavaScript.info: Function Parameters](https://javascript.info/function-basics#default-values)
- [JavaScript.info: Destructuring Assignment](https://javascript.info/destructuring-assignment)
- [Exploring ES6: Parameter Handling](https://exploringjs.com/es6/ch_parameter-handling.html)
