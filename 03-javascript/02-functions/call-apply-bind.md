# `call`, `apply`, and `bind`

## Overview

`call`, `apply`, and `bind` are methods on `Function.prototype` that give developers explicit control over what `this` refers to when a function executes. They are the mechanism behind Rule 3 (explicit binding) in the `this` binding precedence hierarchy. While modern code increasingly relies on arrow functions and class fields for `this` management, understanding `call`, `apply`, and `bind` is essential for reading legacy code, working with dynamic dispatch, borrowing methods, and understanding how higher-level constructs like `bind`-heavy event systems and partial application work internally.

## `Function.prototype.call`

### Syntax

```javascript
fn.call(thisArg, arg1, arg2, ...argN)
```

`call` invokes the function immediately with `this` set to `thisArg`. Arguments are passed individually.

### Basic Usage

```javascript
function greet(greeting, punctuation) {
  return `${greeting}, ${this.name}${punctuation}`;
}

const alice = { name: "Alice" };
const bob   = { name: "Bob" };

greet.call(alice, "Hello", "!"); // "Hello, Alice!"
greet.call(bob, "Hey", ".");     // "Hey, Bob."
```

### Borrowing Methods

A primary use case for `call` is borrowing methods from one object to use on another.

```javascript
// Borrowing Array methods for array-like objects
function logAll() {
  // 'arguments' is array-like, not a real array
  const arr = Array.prototype.slice.call(arguments);
  return arr.map(x => x * 2);
}

logAll(1, 2, 3); // [2, 4, 6]

// Borrowing hasOwnProperty for objects with no prototype
const obj = Object.create(null);
obj.key = "value";
// obj.hasOwnProperty("key"); // TypeError — no inherited methods

Object.prototype.hasOwnProperty.call(obj, "key"); // true (safe borrow)
```

### Calling Superclass Constructors

```javascript
function Animal(name, sound) {
  this.name = name;
  this.sound = sound;
}

function Dog(name) {
  Animal.call(this, name, "Woof"); // invoke super constructor with 'this'
  this.type = "dog";
}

const rex = new Dog("Rex");
console.log(rex.name);  // "Rex"
console.log(rex.sound); // "Woof"
console.log(rex.type);  // "dog"
```

### null and undefined as thisArg

Passing `null` or `undefined` as `thisArg` results in the global object being used in sloppy mode, or `undefined` in strict mode.

```javascript
function showThis() {
  "use strict";
  console.log(this);
}

showThis.call(null);      // null (strict mode preserves null)
showThis.call(undefined); // undefined (strict mode preserves undefined)

function showThisSloppy() {
  console.log(this === globalThis);
}

showThisSloppy.call(null);      // true — defaults to global
showThisSloppy.call(undefined); // true — defaults to global
```

## `Function.prototype.apply`

### Syntax

```javascript
fn.apply(thisArg, [argsArray])
```

`apply` is identical to `call` except arguments are passed as an array (or array-like object). It also invokes the function immediately.

### Basic Usage

```javascript
function sum(a, b, c) {
  return a + b + c;
}

sum.apply(null, [1, 2, 3]); // 6
```

### Spreading Arguments from an Array

Before the spread operator, `apply` was the primary way to spread an array into positional arguments.

```javascript
const numbers = [5, 3, 9, 1, 7];

// ES5 — apply to spread an array
const max = Math.max.apply(null, numbers); // 9
const min = Math.min.apply(null, numbers); // 1

// ES6 equivalent — prefer this today
const max2 = Math.max(...numbers); // 9
```

### apply with Dynamic Argument Arrays

`apply` is still useful when arguments are built dynamically at runtime.

```javascript
function buildQuery(table, ...conditions) {
  const clauses = conditions.join(" AND ");
  return `SELECT * FROM ${table} WHERE ${clauses}`;
}

const conditions = ["age > 18", "active = true", "role = 'admin'"];
buildQuery.apply(null, ["users", ...conditions]);
// or
buildQuery.apply(null, ["users"].concat(conditions));
```

### Detecting Types Safely

A classic use of `call`/`apply` is invoking `Object.prototype.toString` to get accurate type tags.

```javascript
function typeOf(value) {
  return Object.prototype.toString.call(value).slice(8, -1).toLowerCase();
}

typeOf([]);          // "array"
typeOf(null);        // "null"
typeOf(/regex/);     // "regexp"
typeOf(new Date());  // "date"
typeOf(undefined);   // "undefined"
typeOf(42);          // "number"
```

## `Function.prototype.bind`

### Syntax

```javascript
const boundFn = fn.bind(thisArg, arg1, arg2, ...argN)
```

`bind` does not invoke the function. It returns a new function with `this` permanently set to `thisArg`. Optionally, arguments can be pre-filled (partial application).

### Basic Usage

```javascript
function greet(greeting) {
  return `${greeting}, ${this.name}!`;
}

const alice = { name: "Alice" };
const greetAlice = greet.bind(alice);

greetAlice("Hello"); // "Hello, Alice!"
greetAlice("Hey");   // "Hey, Alice!"
```

### Preserving this for Callbacks

The classic use case: preventing `this` loss when passing methods as callbacks.

```javascript
class Timer {
  constructor() {
    this.elapsed = 0;
    this.tick = this.tick.bind(this); // bind once in constructor
  }

  tick() {
    this.elapsed++;
    console.log(`${this.elapsed}s elapsed`);
  }

  start() {
    return setInterval(this.tick, 1000);
  }
}

const t = new Timer();
t.start(); // logs 1s, 2s, 3s... correctly
```

### Partial Application with bind

`bind` pre-fills leading arguments, producing a function that expects fewer arguments.

```javascript
function multiply(a, b) {
  return a * b;
}

const double  = multiply.bind(null, 2);  // a=2 pre-filled
const triple  = multiply.bind(null, 3);  // a=3 pre-filled
const tenTimes = multiply.bind(null, 10);

double(5);   // 10
triple(5);   // 15
tenTimes(7); // 70
```

```javascript
// Practical partial application
function request(method, url, data) {
  return fetch(url, { method, body: JSON.stringify(data) });
}

const get    = request.bind(null, "GET");
const post   = request.bind(null, "POST");
const put    = request.bind(null, "PUT");
const del    = request.bind(null, "DELETE");

get("/api/users");
post("/api/users", { name: "Alice" });
```

### bind Behavior with new

As noted in `this` binding rules, `new` overrides a `bind`-provided `this`. The pre-filled arguments are still honored.

```javascript
function Point(x, y) {
  this.x = x;
  this.y = y;
}

const BoundPoint = Point.bind({}, 10); // pre-fill x=10
const p = new BoundPoint(20);          // y=20

console.log(p.x); // 10 — pre-filled arg honored
console.log(p.y); // 20
// 'this' is the new object, not {}
console.log(p instanceof Point); // true
```

### Bound Function Characteristics

```javascript
function foo() {}

const bound = foo.bind({});

// Bound functions have a fixed length property
console.log(foo.length);          // 0 (no params)
console.log(bound.length);        // 0

function bar(a, b, c) {}
const boundBar = bar.bind(null, 1); // pre-fill 1 arg
console.log(boundBar.length);      // 2 (3 - 1 pre-filled)

// Bound functions have no own prototype
console.log(typeof bound.prototype); // "undefined"

// name property reflects binding
console.log(bound.name); // "bound foo"
```

## Comparison Table

| Feature | `call` | `apply` | `bind` |
|---|---|---|---|
| Invokes function | Immediately | Immediately | Never (returns new function) |
| Arguments format | Individual (`a, b, c`) | Array `[a, b, c]` | Individual (pre-filled) |
| Returns | Function result | Function result | New bound function |
| `this` effect | One-time override | One-time override | Permanent (hard binding) |
| Overrides arrow `this` | No | No | No |
| Primary use case | Method borrowing, super calls | Spreading dynamic arrays | Callbacks, partial application |
| ES6 alternative | N/A | Spread operator `...` | Arrow functions, class fields |

## Implementing bind from Scratch

A common interview question. Understanding this implementation reveals how `bind` handles partial application and `new`.

```javascript
Function.prototype.myBind = function(thisArg, ...partialArgs) {
  const originalFn = this;

  function BoundFn(...callArgs) {
    const allArgs = [...partialArgs, ...callArgs];
    // If called with 'new', use the newly created 'this', not the bound one
    if (this instanceof BoundFn) {
      return new originalFn(...allArgs);
    }
    return originalFn.call(thisArg, ...allArgs);
  }

  // Preserve prototype chain for instanceof checks
  if (originalFn.prototype) {
    BoundFn.prototype = Object.create(originalFn.prototype);
  }

  // Reflect correct length (original length minus pre-filled args)
  Object.defineProperty(BoundFn, "length", {
    value: Math.max(0, originalFn.length - partialArgs.length)
  });

  Object.defineProperty(BoundFn, "name", {
    value: `bound ${originalFn.name}`
  });

  return BoundFn;
};
```

## Implementing call from Scratch

```javascript
Function.prototype.myCall = function(thisArg, ...args) {
  // Normalize thisArg
  const ctx = (thisArg === null || thisArg === undefined)
    ? globalThis
    : Object(thisArg);

  // Attach the function as a temporary property to invoke with correct 'this'
  const sym = Symbol("__call__");
  ctx[sym] = this;

  const result = ctx[sym](...args);
  delete ctx[sym];

  return result;
};
```

## Implementing apply from Scratch

```javascript
Function.prototype.myApply = function(thisArg, argsArray) {
  const ctx = (thisArg === null || thisArg === undefined)
    ? globalThis
    : Object(thisArg);

  const args = argsArray ? [...argsArray] : [];
  const sym = Symbol("__apply__");
  ctx[sym] = this;

  const result = ctx[sym](...args);
  delete ctx[sym];

  return result;
};
```

## Real-World Patterns

### Mixin Functions

```javascript
const Serializable = {
  serialize() {
    return JSON.stringify(this);
  },
  deserialize(json) {
    return Object.assign(this, JSON.parse(json));
  }
};

class User {
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }
}

// Borrow serialize onto User prototype
User.prototype.serialize = Serializable.serialize;

const u = new User("Alice", 30);
u.serialize(); // '{"name":"Alice","age":30}'
```

### Function Composition with bind

```javascript
const pipe = (...fns) => x => fns.reduce((v, fn) => fn(v), x);

const double  = n => n * 2;
const addTen  = n => n + 10;
const square  = n => n * n;

const transform = pipe(double, addTen, square);
transform(3); // (3*2 + 10)^2 = 256
```

### Throttle Using bind

```javascript
function throttle(fn, delay) {
  let lastCall = 0;
  return function(...args) {
    const now = Date.now();
    if (now - lastCall >= delay) {
      lastCall = now;
      return fn.apply(this, args); // preserve 'this' from the call site
    }
  };
}

class Scroller {
  constructor() {
    this.handleScroll = throttle(this.handleScroll.bind(this), 100);
  }

  handleScroll(event) {
    console.log("scrolled", this);
  }
}
```

### Safe Invocation Utilities

```javascript
// Utility to call a method on an object only if it exists
function safeCall(obj, method, ...args) {
  if (typeof obj[method] === "function") {
    return obj[method].apply(obj, args);
  }
}

safeCall(console, "log", "hello");     // "hello"
safeCall({}, "nonExistent", "data");   // undefined (no throw)
```

## Best Practices

### 1. Prefer Arrow Functions for this Preservation in Modern Code

```javascript
// Modern — arrow class field
class Button {
  handleClick = () => {
    this.doWork();
  };
}

// Legacy — bind in constructor
class Button {
  constructor() {
    this.handleClick = this.handleClick.bind(this);
  }
  handleClick() { this.doWork(); }
}
```

### 2. Use call/apply for Method Borrowing, Not Workarounds

```javascript
// Good use: borrowing toString for type detection
const type = Object.prototype.toString.call(value);

// Good use: applying Math methods to arrays
const max = Math.max.apply(null, arrayOfNumbers);

// Modern equivalent (prefer when available)
const max2 = Math.max(...arrayOfNumbers);
```

### 3. Use bind for Partial Application

```javascript
// Good: creates reusable partially-applied functions
const withBase = (base, n) => Math.log(n) / Math.log(base);
const log2  = withBase.bind(null, 2);
const log10 = withBase.bind(null, 10);

log2(8);   // 3
log10(100); // 2
```

### 4. Store Bound References When Removing Event Listeners

```javascript
class Component {
  constructor(element) {
    this.element = element;
    // Store the bound reference to remove it later
    this._boundHandler = this.handleEvent.bind(this);
    element.addEventListener("click", this._boundHandler);
  }

  handleEvent(e) { /* ... */ }

  destroy() {
    // Works: same function reference
    this.element.removeEventListener("click", this._boundHandler);
  }
}
```

### 5. Validate thisArg Type in Borrowing Scenarios

```javascript
// Defensive method borrowing
function safeToString(value) {
  try {
    return Object.prototype.toString.call(value);
  } catch {
    return "[unknown]";
  }
}
```

## Interview Questions

**Q: What is the difference between `call` and `apply`?**

A: Both invoke the function immediately with a specified `this` value. The only difference is how subsequent arguments are provided: `call` accepts them as individual comma-separated values, while `apply` accepts them as a single array or array-like object. The mnemonic is "A for array, C for commas." In modern JavaScript, the spread operator (`fn.call(ctx, ...arr)`) largely replaces the need for `apply`, but `apply` is still common in older codebases.

**Q: What does `bind` return, and how is it different from `call` and `apply`?**

A: `bind` returns a new function with `this` permanently set to the provided value — it does not call the function. `call` and `apply` invoke immediately. The returned bound function can be called later any number of times, always using the fixed `this`. Additionally, `bind` supports partial application: pre-provided arguments are prepended to any arguments passed at invocation time.

**Q: Can you override the `this` value of a function returned by `bind`?**

A: Mostly no. A bound function's `this` cannot be changed by `call`, `apply`, or another `bind` invocation. The one exception is the `new` operator — calling a bound function with `new` ignores the bound `this` and creates a fresh object, though pre-filled arguments are still honored.

**Q: How would you implement `Function.prototype.bind` from scratch?**

A: The core idea is to capture `thisArg` and any partial arguments in a closure, then return a new function that, when called, invokes the original function via `call` with the captured `this` and a concatenation of partial and new arguments. The `new` case requires checking whether `this instanceof` the bound function, and if so, invoking the original with `new` instead. The implementation also copies the prototype relationship and adjusts the `length` property.

**Q: When would you use `apply` over the spread operator?**

A: The spread operator is generally preferred in modern code. `apply` is still appropriate when: working in ES5 environments, dealing with actual `arguments` objects that need to be passed without converting, or when dynamically forwarding all arguments through a proxy or wrapper function where the argument count is unknown at write time. `Function.prototype.apply.call(fn, ctx, args)` is also a safe pattern for calling possibly-overridden `apply`.

**Q: What is `Object.prototype.hasOwnProperty.call(obj, key)` protecting against?**

A: Objects can override `hasOwnProperty` as an own property, either intentionally or via `Object.create(null)` (which produces an object with no prototype at all). Calling `obj.hasOwnProperty(key)` on such objects either invokes the wrong function or throws. `Object.prototype.hasOwnProperty.call(obj, key)` directly invokes the original method regardless of what `obj.hasOwnProperty` is.

## Common Pitfalls

### 1. Forgetting bind Returns, Not Invokes

```javascript
// Bug — bind does not call the function
button.addEventListener("click", handler.bind(ctx)); // works — returns bound fn
handler.bind(ctx); // does nothing useful — return value discarded
```

### 2. Rebinding an Already-Bound Function

```javascript
function greet() { return this.name; }
const boundAlice = greet.bind({ name: "Alice" });
const boundBob   = boundAlice.bind({ name: "Bob" }); // has no effect on 'this'

boundBob(); // "Alice" — the first bind wins
```

### 3. Forgetting to Store the Bound Reference

```javascript
class Component {
  mount(el) {
    // Creating a new bound function each time — cannot remove later
    el.addEventListener("click", this.onClick.bind(this));
  }
  // destroy() { el.removeEventListener("click", ???) }
}
```

### 4. Using apply with Non-Array Second Argument

```javascript
function add(a, b) { return a + b; }

add.apply(null, 1, 2);    // TypeError: CreateListFromArrayLike called on non-object
add.apply(null, [1, 2]);  // 2 — correct
```

### 5. Assuming bind Copies the Function

```javascript
function original() { return this.value; }
const bound = original.bind({ value: 42 });

// They are different function objects
console.log(original === bound); // false

// Modifying original.prototype does not affect bound (no prototype)
original.prototype.extra = "added";
console.log(bound.prototype); // undefined
```

## Resources

### Documentation
- [MDN: Function.prototype.call](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/call)
- [MDN: Function.prototype.apply](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply)
- [MDN: Function.prototype.bind](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind)

### Books and Articles
- [You Don't Know JS: this & Object Prototypes — Chapter 2](https://github.com/getify/You-Dont-Know-JS/blob/1st-ed/this%20%26%20object%20prototypes/ch2.md)
- [JavaScript.info: Decorators and Forwarding](https://javascript.info/call-apply-decorators)
- [JavaScript.info: Function Binding](https://javascript.info/bind)
