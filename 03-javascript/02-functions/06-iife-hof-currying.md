# IIFEs, Higher-Order Functions, Currying, and Partial Application

## The Idea

**In plain English:** These are patterns for writing functions that are smarter and more reusable — some run themselves the moment they are created (IIFE), some accept or return other functions as values (higher-order functions), and some let you "pre-fill" parts of a function so you can finish calling it later (currying and partial application).

**Real-world analogy:** Imagine a coffee machine at a cafe. The barista programs it once with a specific recipe (like "double espresso"), and from then on any staff member just presses a button and it fills a cup — no need to re-enter the recipe each time.

- The pre-programmed recipe button = a curried or partially applied function (configuration locked in up front)
- The act of pressing the button and choosing a cup size = supplying the remaining argument(s)
- The machine running automatically the moment it's plugged in = an IIFE (executes immediately, no separate call needed)

---

## Overview

This cluster of functional programming concepts transforms how JavaScript programs are structured. IIFEs (Immediately Invoked Function Expressions) isolate scope and bootstrap modules. Higher-order functions (HOFs) are the cornerstone of JavaScript's array API and callback-driven asynchrony. Currying and partial application enable constructing reusable, composable function pipelines. Together, these techniques form the backbone of idiomatic functional JavaScript and appear extensively in senior-level interviews, code challenges, and production codebases.

## IIFEs (Immediately Invoked Function Expressions)

### What Is an IIFE

An IIFE is a function expression that is defined and invoked in one step. It creates an isolated scope and executes immediately without polluting the surrounding scope.

```javascript
(function() {
  const secret = "not accessible outside";
  console.log("IIFE runs immediately");
})();

// Arrow function IIFE
(() => {
  console.log("arrow IIFE");
})();

// Named IIFE (name only accessible inside)
(function init() {
  console.log("named IIFE");
})();
```

### Why the Wrapping Parentheses

The function keyword at the start of a statement creates a declaration, which cannot be immediately invoked. Wrapping in parentheses (or using any expression context operator) turns it into an expression.

```javascript
// SyntaxError — declaration, not expression
function() {}();

// Valid — expression context
(function() {})();
(function() {}());  // Crockford style — parens inside
!function() {}();   // Unary operator creates expression context
+function() {}();   // Also valid
void function() {}(); // Also valid — returns undefined
```

### Primary Use Cases

#### 1. Scope Isolation (Pre-ES6)

Before block scoping with `let` and `const`, IIFEs were the only way to create isolated scope.

```javascript
// ES5: IIFE for variable isolation
(function() {
  var localVar = "isolated";
  // localVar is not accessible outside
})();

// ES6+: block scope with let/const makes IIFEs less necessary
{
  const localVar = "isolated";
}
```

#### 2. Module Pattern

IIFEs enable the module pattern — encapsulating private state and exposing a public API.

```javascript
const counter = (function() {
  // Private state
  let count = 0;

  // Private helper
  function validate(n) {
    return typeof n === "number" && !isNaN(n);
  }

  // Public API
  return {
    increment(n = 1) {
      if (!validate(n)) throw new TypeError("Expected a number");
      count += n;
      return this;
    },
    decrement(n = 1) {
      count -= n;
      return this;
    },
    reset() {
      count = 0;
      return this;
    },
    value() {
      return count;
    }
  };
})();

counter.increment().increment(5).value(); // 6
// count is inaccessible directly
```

#### 3. Avoiding Global Variable Conflicts

```javascript
// jQuery plugin pattern
(function($, window, document) {
  // $ is guaranteed to be jQuery here
  // window and document are local references for minification
  $.fn.myPlugin = function() { /* ... */ };
})(jQuery, window, document);
```

#### 4. Initialization Code

```javascript
const config = (function() {
  const env = process.env.NODE_ENV || "development";
  const isDev = env === "development";

  return Object.freeze({
    env,
    apiUrl: isDev ? "http://localhost:3000" : "https://api.example.com",
    debug: isDev,
    version: "2.1.0"
  });
})();
```

#### 5. Capturing Loop Variables (Classic ES5 Closure Problem)

```javascript
// ES5 problem — all callbacks close over the same 'i'
var handlers = [];
for (var i = 0; i < 3; i++) {
  handlers.push(function() { return i; });
}
handlers.map(fn => fn()); // [3, 3, 3] — wrong!

// Fix with IIFE — captures 'i' at each iteration
var handlers = [];
for (var i = 0; i < 3; i++) {
  handlers.push((function(captured) {
    return function() { return captured; };
  })(i));
}
handlers.map(fn => fn()); // [0, 1, 2] — correct

// ES6 solution — let creates a new binding per iteration
const handlers = [];
for (let i = 0; i < 3; i++) {
  handlers.push(() => i);
}
handlers.map(fn => fn()); // [0, 1, 2]
```

## Higher-Order Functions

### Definition

A higher-order function is a function that either accepts one or more functions as arguments, returns a function, or both. Functions are first-class values in JavaScript — they can be stored in variables, passed as arguments, and returned like any other value.

### Built-in Array Higher-Order Functions

#### map

Transforms each element, returning a new array of the same length.

```javascript
const prices   = [10, 25, 5, 50, 30];
const withTax  = prices.map(p => p * 1.1);
// [11, 27.5, 5.5, 55, 33]

// map always returns same length — use filter for different lengths
const users = [
  { id: 1, name: "Alice", active: true },
  { id: 2, name: "Bob",   active: false }
];

const names = users.map(u => u.name); // ["Alice", "Bob"]
```

#### filter

Returns a new array of elements that pass the predicate.

```javascript
const activeUsers = users.filter(u => u.active);
// [{ id: 1, name: "Alice", active: true }]

// Chaining
const activePremiumIds = users
  .filter(u => u.active && u.plan === "premium")
  .map(u => u.id);
```

#### reduce

Accumulates a single value by applying a function to each element and a running accumulator.

```javascript
const sum = [1, 2, 3, 4, 5].reduce((acc, n) => acc + n, 0); // 15

// Grouping with reduce
const grouped = users.reduce((acc, user) => {
  const key = user.active ? "active" : "inactive";
  (acc[key] = acc[key] || []).push(user);
  return acc;
}, {});

// Building a lookup map
const byId = users.reduce((map, user) => {
  map[user.id] = user;
  return map;
}, {});
```

#### forEach, find, some, every, flatMap

```javascript
// find — returns first match or undefined
const admin = users.find(u => u.role === "admin");

// some — true if any element passes
const hasAdmin = users.some(u => u.role === "admin");

// every — true if all elements pass
const allActive = users.every(u => u.active);

// flatMap — map + flatten one level
const words = ["hello world", "foo bar"];
const chars = words.flatMap(s => s.split(" "));
// ["hello", "world", "foo", "bar"]
```

### Custom Higher-Order Functions

```javascript
// Function that accepts a function and returns a function
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

const fibonacci = memoize(function fib(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
});

fibonacci(40); // fast — results cached

// Retry wrapper
function withRetry(fn, maxAttempts = 3, delay = 1000) {
  return async function(...args) {
    let lastError;
    for (let attempt = 1; attempt <= maxAttempts; attempt++) {
      try {
        return await fn(...args);
      } catch (err) {
        lastError = err;
        if (attempt < maxAttempts) {
          await new Promise(res => setTimeout(res, delay * attempt));
        }
      }
    }
    throw lastError;
  };
}

const fetchWithRetry = withRetry(fetch, 3);

// Decorator pattern using HOFs
function readonly(fn) {
  return function(...args) {
    const result = fn(...args);
    return Object.freeze(result);
  };
}

function logged(fn, name = fn.name) {
  return function(...args) {
    console.log(`[${name}] called with`, args);
    const result = fn(...args);
    console.log(`[${name}] returned`, result);
    return result;
  };
}

function timed(fn, name = fn.name) {
  return function(...args) {
    const start = performance.now();
    const result = fn(...args);
    console.log(`[${name}] took ${(performance.now() - start).toFixed(2)}ms`);
    return result;
  };
}
```

### Function Composition

```javascript
// compose — right to left: compose(f, g)(x) === f(g(x))
const compose = (...fns) => x => fns.reduceRight((acc, fn) => fn(acc), x);

// pipe — left to right: pipe(f, g)(x) === g(f(x))
const pipe = (...fns) => x => fns.reduce((acc, fn) => fn(acc), x);

const trim     = s => s.trim();
const toLower  = s => s.toLowerCase();
const replace  = (pattern, sub) => s => s.replace(pattern, sub);
const slugify  = pipe(trim, toLower, replace(/\s+/g, "-"));

slugify("  Hello World  "); // "hello-world"

// Multi-arg compose
const add    = (a, b) => a + b;
const square = n => n * n;
const double = n => n * 2;

const transform = pipe(double, square); // first double, then square
transform(3); // square(double(3)) = square(6) = 36
```

## Currying

### Definition

Currying transforms a function that takes multiple arguments into a sequence of functions, each taking a single argument. A curried function `f(a, b, c)` becomes `f(a)(b)(c)`.

### Manual Currying

```javascript
// Uncurried
function add(a, b) {
  return a + b;
}

// Manually curried
function curriedAdd(a) {
  return function(b) {
    return a + b;
  };
}

curriedAdd(2)(3);   // 5
const addFive = curriedAdd(5);
addFive(10);        // 15
addFive(20);        // 25
```

### Automatic curry Utility

```javascript
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) {
      // Enough arguments — invoke the original function
      return fn.apply(this, args);
    }
    // Not enough — return a function collecting more args
    return function(...moreArgs) {
      return curried.apply(this, [...args, ...moreArgs]);
    };
  };
}

const add  = curry((a, b) => a + b);
const mul  = curry((a, b, c) => a * b * c);

add(1)(2);        // 3
add(1, 2);        // 3 — also works
mul(2)(3)(4);     // 24
mul(2, 3)(4);     // 24 — partial application mixed in
mul(2)(3, 4);     // 24

const add10  = add(10);
const triple = mul(3)(1); // == mul(3, 1) == function waiting for c
triple(5);                // 15
```

### Real-World Currying Patterns

```javascript
// Configuration first, data last — typical functional style
const getProperty = curry((key, obj) => obj[key]);
const getName = getProperty("name");

const users = [{ name: "Alice" }, { name: "Bob" }];
users.map(getName); // ["Alice", "Bob"]

// Validation
const minLength = curry((min, str) => str.length >= min);
const maxLength = curry((max, str) => str.length <= max);
const isEmail   = curry((re, str) => re.test(str));

const validations = [
  minLength(3),
  maxLength(20),
  isEmail(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)
];

function validate(value, validators) {
  return validators.every(v => v(value));
}

// Predicate factories
const gt  = curry((a, b) => b > a);
const lt  = curry((a, b) => b < a);
const eq  = curry((a, b) => b === a);

const isAdult     = gt(17);       // b > 17
const isChild     = lt(18);       // b < 18
const isAdmin     = eq("admin");  // b === "admin"

users.filter(u => isAdult(u.age));
roles.filter(isAdmin);
```

## Partial Application

### Definition

Partial application pre-fills some arguments of a function, returning a new function that expects the remaining arguments. Unlike currying (which always produces unary functions), partial application can pre-fill any number of arguments at once.

```javascript
function partial(fn, ...presetArgs) {
  return function(...laterArgs) {
    return fn(...presetArgs, ...laterArgs);
  };
}

function log(level, timestamp, message) {
  return `[${level}] ${timestamp}: ${message}`;
}

const info  = partial(log, "INFO");
const error = partial(log, "ERROR");

info(new Date().toISOString(), "Server started");
// "[INFO] 2024-01-01T00:00:00.000Z: Server started"

// Partial application with placeholders (Ramda/lodash style)
function partialWithPlaceholders(fn, ...args) {
  const _ = partialWithPlaceholders.placeholder = Symbol("_");

  return function(...newArgs) {
    let newArgIndex = 0;
    const combined = args.map(arg =>
      arg === _ ? newArgs[newArgIndex++] : arg
    );
    return fn(...combined, ...newArgs.slice(newArgIndex));
  };
}
```

### bind as Partial Application

`Function.prototype.bind` is the built-in partial application mechanism.

```javascript
const multiply = (a, b) => a * b;

const double  = multiply.bind(null, 2);
const triple  = multiply.bind(null, 3);

double(5);  // 10
triple(5);  // 15

// bind for method partial application
const arr = [1, 2, 3, 4, 5];
const joinWith = Array.prototype.join.bind(arr);

joinWith("-"); // "1-2-3-4-5"
joinWith(", "); // "1, 2, 3, 4, 5"
```

### Currying vs Partial Application

```javascript
// Currying — always unary, strict one-at-a-time
const curriedAdd = a => b => c => a + b + c;
curriedAdd(1)(2)(3);    // 6 — must supply one at a time

// Partial application — any number of pre-filled args
const partialAdd = partial((a, b, c) => a + b + c, 1, 2);
partialAdd(3);           // 6 — pre-filled two, supply one

// Auto-curry bridges the gap
const flexAdd = curry((a, b, c) => a + b + c);
flexAdd(1)(2)(3);        // 6 — one at a time
flexAdd(1, 2)(3);        // 6 — mixed
flexAdd(1)(2, 3);        // 6 — mixed
flexAdd(1, 2, 3);        // 6 — all at once
```

## Comparison Table

| Concept | Definition | Returns | Argument pattern | Primary benefit |
|---|---|---|---|---|
| IIFE | Immediately invoked function expression | Result of function body | N/A | Isolated scope, initialization |
| Higher-Order Function | Takes/returns functions | Varies | Function as arg or return value | Abstraction, reuse |
| Currying | Multi-arg function → chain of unary functions | Function (until all args supplied) | One at a time | Composability, point-free style |
| Partial Application | Pre-filling some arguments | Function with fewer params remaining | Any number at once | Specialization, reuse |

## Best Practices

### 1. Prefer Native Array Methods Over Manual Loops

```javascript
// Avoid
const doubled = [];
for (let i = 0; i < nums.length; i++) {
  doubled.push(nums[i] * 2);
}

// Prefer — declarative, composable
const doubled = nums.map(n => n * 2);
```

### 2. Put Configuration Before Data in Curried Functions

Functions designed for currying should accept fixed/stable arguments first and variable/data arguments last. This makes partial application natural.

```javascript
// Good: config first, data last
const filterByRole = curry((role, users) => users.filter(u => u.role === role));
const getAdmins = filterByRole("admin");
getAdmins(userList);

// Bad: data first — partial application is awkward
const filterUsers = curry((users, role) => users.filter(u => u.role === role));
// Can't partially apply without placeholder mechanism
```

### 3. Use IIFEs Sparingly in Modern Code

ES modules and block-scoped `let`/`const` solve the scope isolation problem IIFEs addressed. Reserve IIFEs for: initialization code, polyfills, config objects that need computation, and environments without module support.

### 4. Compose Small, Pure Functions

```javascript
// Hard to reuse or test
function processUsers(users) {
  return users
    .filter(u => u.active && u.age >= 18)
    .map(u => ({ ...u, label: `${u.name} (${u.role})` }))
    .sort((a, b) => a.name.localeCompare(b.name));
}

// Small, reusable pieces
const isActive    = u => u.active;
const isAdult     = u => u.age >= 18;
const withLabel   = u => ({ ...u, label: `${u.name} (${u.role})` });
const byName      = (a, b) => a.name.localeCompare(b.name);

const processUsers = users =>
  users
    .filter(u => isActive(u) && isAdult(u))
    .map(withLabel)
    .sort(byName);
```

### 5. Watch Function Arity When Currying

Functions used with `curry` must have a known, fixed `length`. Rest parameters and default parameters reduce apparent arity.

```javascript
const fn = curry((...args) => args.reduce((a, b) => a + b, 0));
fn.length; // 0 — curry cannot determine when enough args are supplied

// Fix: use explicit arity or a custom curry
const sum3 = curry((a, b, c) => a + b + c); // length is 3
```

## Interview Questions

**Q: What is the difference between currying and partial application?**

A: Currying transforms a function into a chain of unary (single-argument) functions. `f(a, b, c)` becomes `f(a)(b)(c)`. Each step returns a function waiting for the next argument. Partial application pre-fills one or more arguments of a function and returns a new function that expects the remaining ones. The number of pre-filled and remaining arguments can be any combination. Currying is a specific form where each application handles exactly one argument; partial application is more general. A practical auto-curry implementation (like Ramda's `R.curry`) merges both: it produces a curried function that also accepts multiple arguments at once.

**Q: Why do functional programmers put configuration before data in function argument order?**

A: Partial application and currying become most useful when stable/configurable arguments are pre-filled first, leaving the variable "data" argument to be supplied later. If data comes first, you'd need to supply it before you can partially apply the config — which defeats the purpose. With config-first ordering, you create a family of specialized functions by partially applying the config, then feed different data to each: `filterByRole("admin")(userList)`, reusing `filterByRole("admin")` across many different lists.

**Q: What problem does an IIFE solve, and is it still relevant in ES6+?**

A: IIFEs historically solved scope isolation — before `let`, `const`, and ES modules, all code shared the global scope, and IIFEs were the only way to create private variables. In modern JavaScript, `let`/`const` provide block scoping and ES modules provide file-level scope. IIFEs are less necessary but still useful for: immediately-executed initialization blocks, computed constant initialization, environments without module support, and code that must run in a clean scope before the module system is available (e.g., polyfills).

**Q: Implement `memoize` — a higher-order function that caches results.**

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
```

Key considerations: key serialization (JSON.stringify fails for circular objects and functions — a WeakMap or custom serializer may be needed), cache eviction strategy for memory management, and handling `this` for method memoization.

**Q: What is function composition and how does `pipe` differ from `compose`?**

A: Function composition combines multiple functions so the output of one becomes the input of the next. `compose` applies functions right to left (mathematical convention): `compose(f, g)(x)` is `f(g(x))`. `pipe` applies them left to right: `pipe(f, g)(x)` is `g(f(x))`. `pipe` reads more naturally in imperative-style thinking ("do this, then this"), while `compose` reflects mathematical notation. Both require that each function (except the first) be unary — accepting the output of the previous step.

## Common Pitfalls

### 1. Assuming curry Works with Variadic Functions

```javascript
const sum = curry((...args) => args.reduce((a, b) => a + b, 0));
// fn.length is 0 — curry can never know when to call the original
sum(1)(2)(3); // returns a function, not 6
```

### 2. Forgetting map Passes Index as Second Argument

```javascript
// Dangerous — parseInt's second arg is radix, not array index
["1", "2", "3"].map(parseInt); // [1, NaN, NaN]

// Fix — wrap in arrow to control argument count
["1", "2", "3"].map(s => parseInt(s, 10)); // [1, 2, 3]

// This is why curried functions must be careful about arity
```

### 3. IIFE Result Confusion

```javascript
const value = (function() {
  const x = 42;
  // forgot to return
})();

console.log(value); // undefined, not 42
```

### 4. Mutating Accumulator in reduce

```javascript
// Bad — mutating the accumulator across iterations
const grouped = users.reduce((acc, user) => {
  acc[user.role] = acc[user.role] || [];
  acc[user.role].push(user); // mutation is fine here since acc is the same object
  return acc;
}, {});

// But watch for accidental sharing when the initial value is reused
// This is safe because {} is created fresh each call to reduce
```

### 5. Over-Currying for Simple Cases

```javascript
// Overkill
const double = curry((a, b) => a * b)(2);

// Just use a closure or arrow
const double = n => n * 2;
```

## Resources

### Documentation
- [MDN: Array.prototype.reduce](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce)
- [MDN: Array.prototype.map](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map)
- [MDN: IIFE](https://developer.mozilla.org/en-US/docs/Glossary/IIFE)

### Articles and Books
- [JavaScript.info: Function Expressions](https://javascript.info/function-expressions)
- [Professor Frisby's Mostly Adequate Guide to FP in JavaScript](https://mostly-adequate.gitbook.io/mostly-adequate-guide/)
- [Ramda Documentation — Currying and Composition](https://ramdajs.com/docs/)
- [Eric Elliott: Composing Software](https://medium.com/javascript-scene/composing-software-an-introduction-27b72500d6ea)
