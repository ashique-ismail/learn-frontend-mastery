# Object Creation Patterns in JavaScript

## The Idea

**In plain English:** An object creation pattern is a recipe for building an "object" — a bundle of related information and actions grouped under one name. JavaScript gives you several different recipes to choose from, each with its own strengths, just like you might build a sandwich differently depending on whether you're making one for yourself or a hundred for a party.

**Real-world analogy:** Think of a cookie-cutter bakery. A baker can make a single one-off cookie by hand (sculpting it directly), stamp out dozens using a metal cookie cutter, follow a printed recipe card that any baker can replicate, or use a cookie-making machine that keeps the dough (ingredients) locked inside so no one can tamper with it.

- The hand-sculpted one-off cookie = the object literal (a unique, single object you write directly)
- The metal cookie cutter = `Object.create` (a shared mold/prototype that new cookies delegate their shape to)
- The printed recipe card = the class (a reusable blueprint that stamps out identical instances with `new`)
- The locked cookie machine = the factory function (a function that hides internal state and hands you a finished cookie)

---

## Overview

JavaScript is a prototype-based language that offers multiple strategies for creating objects. Unlike classical OOP languages, JavaScript did not originally have classes — objects were created by cloning prototypes, calling constructor functions, or assembling them from plain data. ES6 classes are syntactic sugar over the same prototype system, but factories and `Object.create` remain powerful alternatives. Mastering all four patterns — object literals, `Object.create`, classes, and factory functions — allows you to choose the right tool for the problem and understand what frameworks and libraries are doing under the hood.

## Object Literal Pattern

### Basic Usage

The simplest and most common way to create an object. Each property is an own, enumerable, configurable, and writable data descriptor by default.

```javascript
const user = {
  name: "Alice",
  age: 30,
  greet() {
    return `Hi, I'm ${this.name}`;
  }
};
```

### Prototype Chain

All object literals inherit from `Object.prototype`:

```javascript
const obj = {};
Object.getPrototypeOf(obj) === Object.prototype; // true

// Access inherited methods
obj.hasOwnProperty("name"); // true - method from Object.prototype
obj.toString();             // "[object Object]"
```

### Shorthand and Computed Properties

ES6 introduced shorthand property names and computed keys:

```javascript
const x = 10;
const y = 20;

// Shorthand properties
const point = { x, y }; // { x: 10, y: 20 }

// Computed property names
const key = "dynamic";
const config = {
  [key]: true,         // { dynamic: true }
  [`${key}_v2`]: false // { dynamic_v2: false }
};
```

### Limitations

Object literals do not share behavior across instances — every object is a one-off. They are ideal for configuration objects, data transfer objects, and singletons, not for creating many instances with shared methods.

## Object.create Pattern

### Explicit Prototype Delegation

`Object.create(proto)` creates a new object whose internal `[[Prototype]]` is set to `proto`. This is the most direct expression of JavaScript's prototype chain.

```javascript
const animalProto = {
  breathe() {
    return `${this.name} breathes`;
  },
  eat(food) {
    return `${this.name} eats ${food}`;
  }
};

const dog = Object.create(animalProto);
dog.name = "Rex";
dog.bark = function() { return "Woof!"; };

dog.breathe(); // "Rex breathes" — delegated to proto
dog.bark();    // "Woof!" — own method
```

### Object.create(null)

Pass `null` to create a dictionary with no prototype at all — no `toString`, `hasOwnProperty`, or any inherited property:

```javascript
const dict = Object.create(null);
dict.key = "value";

"toString" in dict;       // false — truly clean slate
dict.hasOwnProperty;      // undefined
dict instanceof Object;   // false

// Common use case: safe key-value stores (avoids prototype pollution attacks)
function createSafeMap() {
  return Object.create(null);
}
```

### Prototype Property Descriptor Form

`Object.create` accepts a second argument: property descriptors (same shape as `Object.defineProperties`):

```javascript
const base = { type: "base" };

const child = Object.create(base, {
  id: { value: 1, writable: true, enumerable: true, configurable: true },
  secret: { value: "hidden", enumerable: false }
});

Object.keys(child);      // ["id"] — "secret" is non-enumerable
JSON.stringify(child);   // '{"id":1}' — non-enumerable excluded
```

### Differential Inheritance

`Object.create` enables efficient object extension without copying — the child delegates to the parent:

```javascript
const base = {
  toString() { return `[Base: ${this.id}]`; }
};

const specialized = Object.create(base);
specialized.id = 99;
specialized.toString(); // "[Base: 99]" — no copy, just delegation
```

## Constructor Function Pattern

### Pre-ES6 Classes

Before ES6, constructors were plain functions invoked with `new`. The convention is a capital first letter:

```javascript
function Person(name, age) {
  this.name = name;
  this.age = age;
}

// Methods are shared via prototype — not copied per instance
Person.prototype.greet = function() {
  return `Hi, I'm ${this.name}, ${this.age} years old`;
};

Person.prototype.toString = function() {
  return `Person(${this.name})`;
};

const alice = new Person("Alice", 30);
const bob = new Person("Bob", 25);

alice.greet === bob.greet; // true — same function reference
```

### What `new` Does

Calling `new F()` performs four steps:

```javascript
// new Person("Alice", 30) is roughly equivalent to:
function simulateNew(Constructor, ...args) {
  // 1. Create an empty object linked to Constructor.prototype
  const obj = Object.create(Constructor.prototype);
  // 2. Call the constructor with `this` set to the new object
  const result = Constructor.apply(obj, args);
  // 3. If constructor returns an object, use it; otherwise use `obj`
  return result !== null && typeof result === "object" ? result : obj;
}
```

### Return Value Override

A constructor that explicitly returns an object overrides the default behavior:

```javascript
function Weird() {
  this.a = 1;
  return { b: 2 }; // Returns this object instead
}

const w = new Weird();
w.a; // undefined — the returned object has no 'a'
w.b; // 2

// Returning a primitive is ignored:
function Normal() {
  this.a = 1;
  return 42; // Ignored
}
new Normal().a; // 1
```

## Class Pattern

### ES6 Class Syntax

Classes are syntactic sugar over the prototype system, but they enforce stricter semantics (must use `new`, not hoisted as values, methods are non-enumerable):

```javascript
class Animal {
  #energy; // Private field (ES2022)

  constructor(name, energy) {
    this.name = name;
    this.#energy = energy;
  }

  eat(food) {
    this.#energy += 10;
    return `${this.name} eats ${food}`;
  }

  get currentEnergy() {
    return this.#energy;
  }

  static create(name, energy = 100) {
    return new Animal(name, energy);
  }
}

const cat = Animal.create("Whiskers");
cat.eat("fish"); // "Whiskers eats fish"
cat.currentEnergy; // 110
```

### Inheritance via extends

```javascript
class Dog extends Animal {
  #breed;

  constructor(name, energy, breed) {
    super(name, energy); // Must call super before accessing `this`
    this.#breed = breed;
  }

  bark() {
    return `${this.name} (${this.#breed}) says: Woof!`;
  }

  eat(food) {
    const result = super.eat(food); // Delegate to parent
    return `${result} (tail wagging)`;
  }
}

const rex = new Dog("Rex", 80, "Labrador");
rex.eat("kibble"); // "Rex eats kibble (tail wagging)"
rex.bark();        // "Rex (Labrador) says: Woof!"
rex instanceof Dog;    // true
rex instanceof Animal; // true
```

### Class Internals

Classes are mostly equivalent to their prototype counterparts, with key differences:

```javascript
class Foo {
  method() {}
  static staticMethod() {}
}

// What this produces:
typeof Foo;                             // "function"
Foo.prototype.method.enumerable;       // false (unlike manual assignment)
Object.getOwnPropertyDescriptor(Foo.prototype, "method");
// { value: [Function], writable: true, enumerable: false, configurable: true }

// You cannot call a class without `new`
Foo(); // TypeError: Class constructor Foo cannot be invoked without 'new'
```

## Factory Function Pattern

### What is a Factory

A factory is any function that creates and returns an object without using `new` or `this`. It leverages closure for private state:

```javascript
function createCounter(initial = 0) {
  let count = initial; // Private — not on the object

  return {
    increment() { count++; },
    decrement() { count--; },
    reset() { count = initial; },
    get value() { return count; }
  };
}

const counter = createCounter(10);
counter.increment();
counter.increment();
counter.value; // 12
counter.count; // undefined — truly private via closure
```

### Factory vs Class: True Privacy

Unlike class private fields (`#`), closure-based privacy works in all ES6 environments and cannot be accessed even via reflection:

```javascript
// Class private field
class BankAccount {
  #balance;
  constructor(balance) { this.#balance = balance; }
  deposit(amount) { this.#balance += amount; }
  get balance() { return this.#balance; }
}

// Can be accessed via special syntax in same-class methods,
// but not from outside

// Factory closure
function createBankAccount(initialBalance) {
  let balance = initialBalance;

  return {
    deposit(amount) { balance += amount; },
    withdraw(amount) {
      if (amount > balance) throw new Error("Insufficient funds");
      balance -= amount;
    },
    get balance() { return balance; }
  };
}

// Cannot access `balance` from outside in any way
```

### Composition Over Inheritance

Factories compose behavior naturally using object spread and mixin functions:

```javascript
const withLogging = (obj) => ({
  ...obj,
  log(message) {
    console.log(`[${new Date().toISOString()}] ${message}`);
  }
});

const withValidation = (obj) => ({
  ...obj,
  validate(value) {
    return value !== null && value !== undefined;
  }
});

function createService(name) {
  const base = { name };
  return withLogging(withValidation(base));
}

const svc = createService("UserService");
svc.log("started"); // Logs with timestamp
svc.validate(null); // false
```

### Memory Consideration

Each factory call creates new function objects for every method — unlike classes where methods live on the prototype:

```javascript
class Klass {
  method() {}
}
const a = new Klass();
const b = new Klass();
a.method === b.method; // true — shared via prototype

function factory() {
  return { method() {} };
}
const c = factory();
const d = factory();
c.method === d.method; // false — each object gets its own copy
```

For high-frequency object creation, classes or `Object.create` are more memory-efficient.

## Comparison Table

| Feature | Object Literal | Object.create | Class | Factory |
| --- | --- | --- | --- | --- |
| Prototype control | Implicit (Object.prototype) | Explicit | Via `extends` | Implicit / closure |
| Shared methods | No | Yes (manual) | Yes (auto) | No (unless shared ref) |
| Privacy | None | None | `#` fields | Closure |
| `new` required | No | No | Yes | No |
| Inheritance | None | Manual delegation | `extends` keyword | Composition/mixins |
| Memory (many instances) | Poor | Good | Good | Poor |
| TypeScript/type checking | Moderate | Poor | Excellent | Good |
| `instanceof` works | No | Partial | Yes | No |

## Best Practices

### 1. Match the Pattern to the Use Case

```javascript
// Single instance / config → literal
const config = { apiUrl: "/api", timeout: 5000 };

// Prototype chain control → Object.create
const base = Object.create(null); // Safe dictionary

// Multiple instances with shared behavior → class
class EventEmitter { /* ... */ }

// Encapsulation / composition / no `this` → factory
const createStore = (reducer) => { /* ... */ };
```

### 2. Avoid `new` in Factories

Factories should never require `new` — guard against it if needed:

```javascript
function createUser(name) {
  if (this instanceof createUser) {
    // Called with new accidentally
    throw new Error("createUser is a factory, not a constructor");
  }
  return { name };
}
```

### 3. Prefer Composition Over Deep Inheritance

Deep class hierarchies are brittle. Prefer composable factories or mixins for behavior reuse.

### 4. Use `Object.create(null)` for True Dictionaries

When using an object as a hash map where keys may collide with prototype names (`constructor`, `toString`, etc.), prefer `Object.create(null)`.

## Interview Questions

### Q1: What is the difference between `Object.create(proto)` and `new Constructor()`?

**Answer:** Both create an object with a given prototype. `Object.create(proto)` directly sets `[[Prototype]]` to `proto` and does not run any initialization code. `new Constructor()` calls the constructor function (runs initialization), sets `[[Prototype]]` to `Constructor.prototype`, and returns the new object. `Object.create` is more explicit about prototype delegation; `new` pairs initialization with prototype setup.

### Q2: Why do factories not work with `instanceof`?

**Answer:** `instanceof` checks whether `Constructor.prototype` exists anywhere in an object's prototype chain. Factory functions return plain objects whose `[[Prototype]]` is `Object.prototype` (or whatever was used in the literal), not a factory-specific prototype. Since factories don't set a prototype, `instanceof` has nothing to check against.

### Q3: What are the four steps performed by the `new` keyword?

**Answer:** (1) Create a new empty object whose `[[Prototype]]` is set to `Constructor.prototype`. (2) Execute the constructor function with `this` bound to the new object. (3) If the constructor explicitly returns a non-null object, use that; otherwise use the object created in step 1. (4) Return the object.

### Q4: How does a class method differ from one added via `prototype` assignment?

**Answer:** Class methods are non-enumerable by default (`enumerable: false`), so they don't appear in `for...in` loops or `Object.keys`. Manual prototype assignment creates enumerable methods by default. Both end up on the same prototype object, but the descriptor differs.

### Q5: When would you choose a factory over a class?

**Answer:** Factories are preferable when: you need true closure-based privacy that predates ES2022; you want to avoid `this` binding bugs in callbacks; you want to compose behavior from multiple sources without inheritance hierarchies; you don't need `instanceof` checks; or you are building functional-style code where `new` feels out of place.

## Common Pitfalls

### 1. Forgetting `new` With Constructor Functions

```javascript
function Person(name) {
  this.name = name;
}

const p = Person("Alice"); // Forgot new!
p; // undefined — no return value
window.name; // "Alice" — accidentally set on global in non-strict mode

// In strict mode, `this` is undefined → TypeError
```

### 2. Defining Methods Inside the Constructor

```javascript
// Bad — new function per instance
class Bad {
  constructor() {
    this.method = function() { /* ... */ }; // Not on prototype!
  }
}

// Good — single shared reference on prototype
class Good {
  method() { /* ... */ }
}
```

### 3. Mutating `Object.prototype`

```javascript
// NEVER do this
Object.prototype.exists = function() { return true; };

// Now every object gets this method — including plain data objects
// This breaks `for...in` loops and third-party libraries
```

### 4. Using `Object.create` Without Initializing Own Properties

```javascript
const proto = { greet() { return this.name; } };
const obj = Object.create(proto);
obj.greet(); // undefined — forgot to set this.name = "..."
```

## Resources

### Official Documentation

- [MDN: Object.create](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/create)
- [MDN: Classes](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes)
- [ECMAScript: The new Operator](https://tc39.es/ecma262/#sec-new-operator)
- [TC39: Class fields proposal](https://github.com/tc39/proposal-class-fields)

### Articles and Books

- [You Don't Know JS: this & Object Prototypes](https://github.com/getify/You-Dont-Know-JS/tree/2nd-ed/objects-classes)
- [JavaScript: The Good Parts — Chapter on Objects](https://www.oreilly.com/library/view/javascript-the-good/9780596517748/)
- [Composition vs Inheritance (Eric Elliott)](https://medium.com/javascript-scene/composition-vs-inheritance-f0f9be00c17e)
- [Factory Functions vs Classes (Eric Elliott)](https://medium.com/javascript-scene/javascript-factory-functions-vs-constructor-functions-vs-classes-2f22ceddf33e)
