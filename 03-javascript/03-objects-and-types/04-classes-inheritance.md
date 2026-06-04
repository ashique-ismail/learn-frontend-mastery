# Classes, Inheritance, super, and Private Fields

## Overview

ES6 classes gave JavaScript a syntactic layer over its prototype system that aligns with classical OOP idioms while preserving the underlying prototype delegation model. ES2022 added private class fields (`#`) and static class blocks, completing a feature set that covers most real-world OOP needs. Understanding classes deeply means understanding both the sugar — what the syntax does for you — and the mechanics — how prototypes, `super`, descriptors, and the `new` operator interact under the hood. This file covers the full class system at senior-engineer depth, including tricky behavior around `this`, `super` binding, private fields, and inheritance edge cases.

## Class Declaration Basics

### Syntax

```javascript
class Animal {
  // Static property
  static kingdom = "Animalia";

  // Private instance field (ES2022)
  #id;

  // Public instance field
  species = "unknown"; // Initialized per instance

  constructor(name, age) {
    this.#id = Math.random().toString(36).slice(2);
    this.name = name;
    this.age = age;
  }

  // Prototype method (non-enumerable)
  describe() {
    return `${this.name} (${this.species}), age ${this.age}`;
  }

  // Getter on prototype
  get info() {
    return `[${this.constructor.name} #${this.#id}]`;
  }

  // Static method
  static create(name, age) {
    return new this(name, age); // `this` is the class itself in static context
  }
}
```

### What a Class Actually Is

```javascript
typeof Animal; // "function" — a class is a constructor function
Animal.prototype.describe; // [Function: describe]
Object.getOwnPropertyDescriptor(Animal.prototype, "describe");
// { value: [Function], writable: true, enumerable: false, configurable: true }
// Key difference from manual prototype assignment: enumerable: false
```

Class methods are non-enumerable — they don't appear in `for...in` loops. Manually assigned prototype methods (`Foo.prototype.bar = function() {}`) are enumerable.

### Class Expression

Classes can be named or anonymous expressions, useful for dynamic class creation:

```javascript
const Foo = class NamedFoo {
  whoAmI() { return NamedFoo.name; } // Internal name only
};

const Bar = class {
  whoAmI() { return "anonymous"; }
};

// Dynamic class factory
function makeClass(greeting) {
  return class {
    greet() { return greeting; }
  };
}
const HelloClass = makeClass("Hello!");
new HelloClass().greet(); // "Hello!"
```

### No Hoisting

Unlike function declarations, class declarations are not hoisted as values (though the binding is, entering the TDZ):

```javascript
const a = new Foo(); // ReferenceError: Cannot access 'Foo' before initialization
class Foo {}
```

### Strict Mode by Default

All code inside a class body runs in strict mode automatically, regardless of the surrounding scope.

## Public and Private Fields

### Public Instance Fields

Public fields declared in the class body are initialized per instance in the constructor, before the constructor body runs:

```javascript
class Counter {
  count = 0;                   // Public instance field
  label = this.defaultLabel(); // Can reference other methods

  defaultLabel() { return "Counter"; }
}

const c = new Counter();
c.hasOwnProperty("count"); // true — own property, not on prototype
c.hasOwnProperty("defaultLabel"); // false — method is on prototype
```

### Private Instance Fields

Private fields use a `#` prefix. They are hard-private — not accessible outside the class body, not on the prototype, not visible via `Object.keys`, `for...in`, or `Reflect.ownKeys` (they are listed by `Object.getOwnPropertyNames` as strings starting with `#`, but still inaccessible from outside).

```javascript
class BankAccount {
  #balance;
  #owner;
  #transactions = [];

  constructor(owner, initial) {
    this.#owner = owner;
    this.#balance = initial;
  }

  deposit(amount) {
    if (amount <= 0) throw new RangeError("Amount must be positive");
    this.#balance += amount;
    this.#transactions.push({ type: "deposit", amount });
    return this;
  }

  withdraw(amount) {
    if (amount > this.#balance) throw new Error("Insufficient funds");
    this.#balance -= amount;
    this.#transactions.push({ type: "withdrawal", amount });
    return this;
  }

  get balance() { return this.#balance; }
  get owner() { return this.#owner; }
  get history() { return [...this.#transactions]; }
}

const account = new BankAccount("Alice", 1000);
account.deposit(500).withdraw(200);
account.balance; // 1300
account.#balance; // SyntaxError: Private field '#balance' must be declared in an enclosing class
```

### Private in `instanceof` Alternative

Private fields enable a brand check — verifying an object is a genuine instance of the class (not a duck-typed impostor):

```javascript
class Token {
  #secret;

  constructor(value) {
    this.#secret = value;
  }

  static isToken(obj) {
    try {
      obj.#secret; // Throws if obj is not a Token instance
      return true;
    } catch {
      return false;
    }
  }

  // Or with the `in` operator (ES2022):
  static isTokenV2(obj) {
    return #secret in obj;
  }
}

Token.isToken(new Token("x")); // true
Token.isToken({});             // false
```

### Static Fields and Private Static Fields

```javascript
class IdGenerator {
  static #nextId = 1;
  static #instances = new Set();

  #id;

  constructor() {
    this.#id = IdGenerator.#nextId++;
    IdGenerator.#instances.add(this);
  }

  get id() { return this.#id; }

  static get count() { return IdGenerator.#instances.size; }
}

const a = new IdGenerator(); // id: 1
const b = new IdGenerator(); // id: 2
IdGenerator.count; // 2
```

## Inheritance and extends

### Basic Inheritance

```javascript
class Animal {
  constructor(name) {
    this.name = name;
  }

  speak() {
    return `${this.name} makes a sound`;
  }
}

class Dog extends Animal {
  #breed;

  constructor(name, breed) {
    super(name); // Must come before any `this` access
    this.#breed = breed;
  }

  speak() {
    return `${this.name} barks`;
  }

  get breed() { return this.#breed; }
}

const dog = new Dog("Rex", "Labrador");
dog.speak(); // "Rex barks"
dog instanceof Dog;    // true
dog instanceof Animal; // true
```

### Prototype Chain Established by extends

`extends` sets up two prototype chains:
1. Instance chain: `dog.__proto__ === Dog.prototype` and `Dog.prototype.__proto__ === Animal.prototype`
2. Constructor chain: `Dog.__proto__ === Animal` (for static inheritance)

```javascript
Object.getPrototypeOf(Dog.prototype) === Animal.prototype; // true — instance chain
Object.getPrototypeOf(Dog) === Animal;                     // true — static chain

// Static methods are inherited
class Base {
  static staticMethod() { return "from Base"; }
}
class Child extends Base {}
Child.staticMethod(); // "from Base" — inherited via static chain
```

### super in Methods

`super.method()` calls the method from the parent class's prototype:

```javascript
class Logger {
  log(message) {
    return `[LOG] ${message}`;
  }
}

class TimestampLogger extends Logger {
  log(message) {
    const base = super.log(message); // Delegates to parent
    return `${new Date().toISOString()} ${base}`;
  }
}

new TimestampLogger().log("hello");
// "2024-01-15T10:30:00.000Z [LOG] hello"
```

`super` is not a variable — it's a special syntax that uses the internal `[[HomeObject]]` slot of the method to determine which prototype to look up:

```javascript
// This is why super only works inside methods, not regular functions
const obj = {
  method() {
    return super.toString(); // Works — method has [[HomeObject]]
  }
};

// Extracting the method loses [[HomeObject]]
const m = obj.method;
m(); // ReferenceError or wrong behavior
```

### super in the Constructor

`super()` in a derived class constructor calls the parent's constructor and initializes the instance. You must call `super()` before accessing `this`:

```javascript
class Shape {
  constructor(color) {
    this.color = color;
  }
}

class Circle extends Shape {
  constructor(color, radius) {
    // this.radius = radius; // ReferenceError: Must call super constructor first
    super(color);
    this.radius = radius; // OK now
  }
}
```

If the derived class has no constructor, a default one is synthesized:

```javascript
class Derived extends Base {
  // Implicitly:
  // constructor(...args) { super(...args); }
}
```

### Calling Parent's Constructor Without super

If a derived class constructor returns an object explicitly, it skips `this` initialization entirely (no need for `super`). This is used for proxy / mixin patterns but is generally an antipattern.

## Method Overriding and Polymorphism

### Override with Super Delegation

```javascript
class Collection {
  #items = [];

  add(item) {
    this.#items.push(item);
    return this;
  }

  remove(item) {
    const idx = this.#items.indexOf(item);
    if (idx !== -1) this.#items.splice(idx, 1);
    return this;
  }

  get size() { return this.#items.length; }

  [Symbol.iterator]() {
    return this.#items[Symbol.iterator]();
  }
}

class ValidatedCollection extends Collection {
  #validator;

  constructor(validator) {
    super();
    this.#validator = validator;
  }

  add(item) {
    if (!this.#validator(item)) {
      throw new TypeError(`Invalid item: ${item}`);
    }
    return super.add(item); // Delegate to parent
  }
}

const nums = new ValidatedCollection(n => typeof n === "number");
nums.add(1).add(2);
nums.add("x"); // TypeError: Invalid item: x
```

## Mixins

JavaScript's single-prototype chain does not support multiple inheritance. Mixins compose behavior:

```javascript
const Serializable = (Base) => class extends Base {
  serialize() {
    return JSON.stringify(this);
  }

  static deserialize(json) {
    return Object.assign(new this(), JSON.parse(json));
  }
};

const Timestamped = (Base) => class extends Base {
  constructor(...args) {
    super(...args);
    this.createdAt = new Date().toISOString();
    this.updatedAt = this.createdAt;
  }

  touch() {
    this.updatedAt = new Date().toISOString();
    return this;
  }
};

class User {
  constructor(name, email) {
    this.name = name;
    this.email = email;
  }
}

class EnhancedUser extends Serializable(Timestamped(User)) {}

const u = new EnhancedUser("Alice", "alice@example.com");
u.serialize(); // JSON string
u.touch();     // Updates updatedAt
```

## Static Class Block

ES2022 static class blocks run once when the class is evaluated, useful for complex static initialization:

```javascript
class Config {
  static #data;
  static #initialized = false;

  static {
    // Complex initialization logic
    Config.#data = Object.freeze({
      version: "1.0.0",
      features: ["auth", "payments", "analytics"]
    });
    Config.#initialized = true;
    console.log("Config initialized");
  }

  static get(key) {
    return Config.#data[key];
  }

  static isReady() {
    return Config.#initialized;
  }
}
```

## Comparison Table

| Feature | ES6 Class | ES5 Constructor | Factory |
|---|---|---|---|
| Syntax clarity | High | Low | High |
| `new` required | Yes (enforced) | Yes (not enforced) | No |
| Prototype chain | Auto via extends | Manual | None by default |
| Private state | `#` fields (native) | Convention only | Closure |
| `instanceof` | Works | Works | Doesn't work |
| Static inheritance | Automatic | Manual | N/A |
| Method sharing | Prototype | Prototype | Per instance |
| `super` | Built-in | Manual | N/A |

## Best Practices

### 1. Prefer Private Fields for Internal State

```javascript
// Bad — convention-only privacy
class Bad {
  constructor() {
    this._internal = 42; // Anyone can read/write
  }
}

// Good — enforced privacy
class Good {
  #internal = 42;
}
```

### 2. Limit Inheritance Depth

More than two or three levels of inheritance creates brittle code. Prefer composition via mixins for shared behavior.

### 3. Keep Constructors Thin

Constructors should assign fields and call `super`. Complex logic belongs in factory methods or separate initialization functions.

### 4. Use static create() for Complex Construction

```javascript
class Connection {
  #socket;
  #config;

  constructor(socket, config) {
    this.#socket = socket;
    this.#config = config;
  }

  static async create(url, options = {}) {
    const socket = await openSocket(url);
    const config = await loadConfig(options);
    return new Connection(socket, config);
  }
}

// Async construction can't go in a constructor
const conn = await Connection.create("ws://localhost:8080");
```

## Interview Questions

### Q1: What does `super` do in a constructor vs in a method?

**Answer:** In a constructor of a derived class, `super(args)` calls the parent class's constructor, creates and initializes the instance, and makes `this` available. You must call it before accessing `this`. In a method, `super.methodName()` performs a property lookup starting from the parent class's prototype, bypassing the current class's override. It uses the method's `[[HomeObject]]` internal slot (set at class definition time) to determine which prototype to start from — this is why `super` is bound to the method, not to `this`.

### Q2: What is the difference between class fields and constructor assignments?

**Answer:** Class fields (`x = value`) are initialized per instance in the specification-defined initialization phase, which occurs before the constructor body runs. They appear as own properties. Constructor assignments (`this.x = value`) happen during the constructor body. The behavior is largely the same for most use cases, but private fields (`#x`) can only be declared as class fields — they cannot be added dynamically in the constructor (though they can be assigned in the constructor after being declared).

### Q3: How does private field access differ from WeakMap-based privacy?

**Answer:** Before `#` fields, private state was implemented with a WeakMap keyed on `this`. Both approaches are truly private (no external access without cooperation of the class). Private fields are syntactically cleaner, have better performance (engine-level support vs WeakMap lookup), and support the brand check pattern (`#field in obj`). WeakMap-based privacy works in ES6+ without ES2022 and doesn't require class syntax.

### Q4: Can a private field be accessed by a subclass?

**Answer:** No. Private fields are scoped to the class body where they are declared. A subclass cannot access `#field` from the parent even though it inherits the parent's public interface. If the parent class needs to share data with subclasses, it must expose it via a protected-style method or getter.

### Q5: What is [[HomeObject]] and why does super break when you extract a method?

**Answer:** `[[HomeObject]]` is an internal slot set on a method when it is defined in a class or object literal. It holds a reference to the object on which the method was defined (the prototype or class). `super` uses `[[HomeObject]]` to find the parent prototype by walking up the chain from it. When you extract a method (`const m = obj.method`), the extracted function still has the original `[[HomeObject]]` — `super` still resolves relative to the original object. However, `this` is lost unless you bind it. The combination of a detached `this` and a fixed `[[HomeObject]]` can cause confusing bugs.

## Common Pitfalls

### 1. Using this Before super

```javascript
class Child extends Parent {
  constructor(x) {
    this.x = x; // ReferenceError — must call super first
    super();
  }
}
```

### 2. Arrow Function Methods Are Not on the Prototype

```javascript
class Foo {
  // This is a public instance field with an arrow function
  // Not a prototype method — created anew for every instance
  method = () => { return this; };
}

const a = new Foo();
const b = new Foo();
a.method === b.method; // false — different functions
// Useful for event listeners (preserves `this`), but wastes memory
```

### 3. Class Declarations Are Not Hoisted

Unlike function declarations, you cannot use a class before it is defined in the source code.

### 4. Extending Built-ins (ArraySubclass, etc.)

Extending native built-ins works in ES6+ but can have subtle issues in transpiled code. Babel cannot fully replicate native class behavior for Array, Map, etc. Always test native class behavior in target environments.

```javascript
class MyArray extends Array {
  sum() {
    return this.reduce((a, b) => a + b, 0);
  }
}

const arr = new MyArray(1, 2, 3);
arr.sum(); // 6 — works in native ES6 environments
arr.map(x => x * 2) instanceof MyArray; // true — Symbol.species
```

## Resources

### Official Documentation
- [MDN: Classes](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes)
- [MDN: Private class features](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Private_class_fields)
- [MDN: super](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/super)
- [TC39: Class Fields Proposal](https://github.com/tc39/proposal-class-fields)
- [TC39: Static Class Block](https://github.com/tc39/proposal-class-static-block)

### Articles and Books
- [You Don't Know JS: Objects & Classes](https://github.com/getify/You-Dont-Know-JS/tree/2nd-ed/objects-classes)
- [JavaScript Mixins (2ality)](https://2ality.com/2016/05/mixins.html)
- [ES6 Classes in Depth (MDN Hacks)](https://hacks.mozilla.org/2015/07/es6-in-depth-classes/)
- [Composition vs Inheritance (Eric Elliott)](https://medium.com/javascript-scene/the-two-pillars-of-javascript-ee6f3281e7f3)
