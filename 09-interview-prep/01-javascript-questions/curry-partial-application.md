# Currying & Partial Application

## Overview

Both patterns transform multi-argument functions into more reusable forms. Interviewers use these to test functional programming knowledge and closure mechanics.

---

## Currying

**Transforms** `f(a, b, c)` into `f(a)(b)(c)` — a chain of single-argument functions, each returning the next until all arguments are collected.

### Manual Curry (Fixed Arity)

```js
// Transform a 2-arg function manually
const add = (a, b) => a + b;
const curriedAdd = a => b => a + b;

curriedAdd(1)(2); // 3
const add5 = curriedAdd(5); // partially applied
add5(3); // 8
```

### Generic `curry()` Implementation

```js
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    }
    return function(...moreArgs) {
      return curried.apply(this, args.concat(moreArgs));
    };
  };
}

const add = curry((a, b, c) => a + b + c);
add(1)(2)(3);     // 6
add(1, 2)(3);     // 6 — can pass multiple args at once
add(1)(2, 3);     // 6
add(1, 2, 3);     // 6
```

**Key insight:** `fn.length` is the number of declared parameters. Once we have enough, call the original.

### Real-World Use Case

```js
const multiply = curry((factor, value) => factor * value);
const double = multiply(2);
const triple = multiply(3);

[1, 2, 3, 4].map(double);  // [2, 4, 6, 8]
[1, 2, 3, 4].map(triple);  // [3, 6, 9, 12]
```

---

## Partial Application

**Fixes some arguments** of a function, returning a new function that takes the remaining arguments. Unlike currying, you can fix multiple args at once.

### Using `Function.prototype.bind`

```js
function add(a, b, c) { return a + b + c; }
const add10 = add.bind(null, 10);      // fixes a=10
const add10and5 = add.bind(null, 10, 5); // fixes a=10, b=5

add10(5, 3);    // 18
add10and5(3);   // 18
```

### Manual `partial()`

```js
function partial(fn, ...presetArgs) {
  return function(...laterArgs) {
    return fn(...presetArgs, ...laterArgs);
  };
}

const log = (level, message) => console.log(`[${level}] ${message}`);
const warn = partial(log, 'WARN');
const error = partial(log, 'ERROR');

warn('Disk space low');    // [WARN] Disk space low
error('Connection lost');  // [ERROR] Connection lost
```

---

## Currying vs Partial Application

| | Currying | Partial Application |
|---|---|---|
| Transformation | `f(a,b,c)` → `f(a)(b)(c)` | `f(a,b,c)` → `f(b,c)` (a fixed) |
| Args per call | One at a time | Any number |
| Returns | New unary function | New function with fewer params |
| Used for | Point-free composition | Configuring functions with defaults |

```js
// Currying — one arg at a time
const curried = curry(add);
const step1 = curried(1);  // takes b
const step2 = step1(2);    // takes c → executes

// Partial application — fix what you know now
const addTo10 = partial(add, 10); // takes b and c
addTo10(2, 3); // 15
```

---

## Point-Free Style with Curried Functions

```js
const pipe = (...fns) => x => fns.reduce((v, f) => f(v), x);

const isOdd = n => n % 2 !== 0;
const double = n => n * 2;
const clamp = curry((min, max, n) => Math.min(max, Math.max(min, n)));

const transform = pipe(
  double,
  clamp(0, 100),
);

[1, 2, 3, 50, 100].map(transform); // [2, 4, 6, 100, 100]
```

---

## Common Interview Questions

**Q: What does `fn.length` return for `function f(a, b = 1, ...rest) {}`?**
`1` — only parameters before the first default value or rest parameter are counted.

**Q: What's the difference between currying and partial application?**
Currying is a transformation: `f(a, b, c)` becomes `f(a)(b)(c)`, always returning unary functions. Partial application fixes one or more arguments, returning a function with fewer parameters — it's application of some arguments now, rest later.

**Q: Does `bind` do currying or partial application?**
Partial application. `fn.bind(ctx, a, b)` fixes `this` and two arguments. It doesn't force one-argument-at-a-time.

**Q: Why is currying useful in functional programming?**
It enables function composition and point-free style. Every function can be treated as a unary function that takes one argument and returns either a result or another function. This makes functions composable with utilities like `pipe`/`compose`.
