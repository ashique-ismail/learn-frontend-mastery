# Implement Function.prototype.bind Polyfill

## The Idea

**In plain English:** `bind` is a way to lock a function to a specific object so that no matter how or when you call the function later, it always behaves as if it belongs to that object. A "polyfill" means writing that feature yourself from scratch, in case the browser does not have it built in.

**Real-world analogy:** Imagine a restaurant chef who normally cooks for whichever kitchen hires them that day. Now the chef signs a permanent contract with one specific restaurant — no matter where they physically show up, they always cook that restaurant's menu with that restaurant's ingredients.

- The chef = the function
- The restaurant they sign a contract with = the `this` object passed to `bind`
- The signed contract = the new bound function returned by `bind`

---

## What `bind` Does

`bind` returns a **new function** that, when called, has its `this` permanently fixed to a given value, with optional pre-filled (partially applied) arguments.

```js
function greet(greeting, punctuation) {
  return `${greeting}, ${this.name}${punctuation}`;
}

const alice = { name: 'Alice' };
const helloAlice = greet.bind(alice, 'Hello');
helloAlice('!');   // "Hello, Alice!"
helloAlice('...');  // "Hello, Alice..."
```

---

## Basic Polyfill

```js
Function.prototype.myBind = function(thisArg, ...presetArgs) {
  const fn = this;  // the function bind was called on

  return function(...callArgs) {
    return fn.apply(thisArg, [...presetArgs, ...callArgs]);
  };
};

function add(a, b) { return a + b; }
const add5 = add.myBind(null, 5);
add5(3);  // 8
```

---

## Handling `new` Invocation

When a bound function is used with `new`, the `this` binding should be ignored — `new` creates a new object and `this` refers to it.

```js
Function.prototype.myBind = function(thisArg, ...presetArgs) {
  const fn = this;

  function BoundFunction(...callArgs) {
    // If called with `new`, `this` is a new instance — use it, not thisArg
    const context = this instanceof BoundFunction ? this : thisArg;
    return fn.apply(context, [...presetArgs, ...callArgs]);
  }

  // Preserve the prototype chain so `instanceof` works correctly
  BoundFunction.prototype = Object.create(fn.prototype);
  BoundFunction.prototype.constructor = BoundFunction;

  return BoundFunction;
};

function Point(x, y) {
  this.x = x;
  this.y = y;
}

const BoundPoint = Point.myBind(null, 10);
const p = new BoundPoint(20);
console.log(p.x, p.y);  // 10, 20
console.log(p instanceof Point);  // true
```

---

## Preserving `.length` and `.name`

The native `bind` sets the returned function's `.length` to `Math.max(0, fn.length - presetArgs.length)` and its `.name` to `"bound " + fn.name`.

```js
Function.prototype.myBind = function(thisArg, ...presetArgs) {
  const fn = this;
  const boundLength = Math.max(0, fn.length - presetArgs.length);

  const BoundFunction = function(...callArgs) {
    const context = new.target ? this : thisArg;
    return fn.apply(context, [...presetArgs, ...callArgs]);
  };

  // Set metadata (non-enumerable, non-writable like native)
  Object.defineProperty(BoundFunction, 'length', { value: boundLength });
  Object.defineProperty(BoundFunction, 'name', { value: 'bound ' + fn.name });

  BoundFunction.prototype = Object.create(fn.prototype);
  return BoundFunction;
};
```

Note: `new.target` (ES2015) is cleaner than the `instanceof` check for detecting `new` invocation.

---

## Real-World Uses of `bind`

```js
// 1. Fixing `this` for event handlers
class Timer {
  constructor() {
    this.count = 0;
    this.tick = this.tick.bind(this); // bind once in constructor
  }
  tick() { this.count++; }
  start() { setInterval(this.tick, 1000); }
}

// 2. Partial application
function multiply(a, b) { return a * b; }
const double = multiply.bind(null, 2);
[1, 2, 3].map(double); // [2, 4, 6]

// 3. Method borrowing
const arrayLike = { 0: 'a', 1: 'b', length: 2 };
const toArray = Array.prototype.slice.bind(arrayLike);
toArray(); // ['a', 'b']
```

---

## Common Interview Questions

**Q: What's the difference between `call`, `apply`, and `bind`?**
- `call(ctx, a, b)` — invokes immediately, args listed
- `apply(ctx, [a, b])` — invokes immediately, args as array
- `bind(ctx, a, b)` — returns new function with fixed `this` and optional preset args

**Q: Can you rebind a bound function?**
No. Once bound, the `this` is permanent. `boundFn.bind(other)` returns a new function but `this` is still the original binding.

**Q: Why does `bind` return a new function instead of modifying the original?**
Functions are immutable in JS regarding their `this` context. `bind` creates a wrapper that intercepts calls and redirects `this`. The original function is unchanged.

**Q: When would you use `bind` in modern React?**
In class components to ensure `this` inside event handlers refers to the component instance. Function components with hooks don't need this since arrow functions capture `this` lexically (or don't use `this` at all).
