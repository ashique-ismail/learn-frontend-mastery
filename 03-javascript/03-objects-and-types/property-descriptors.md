# Property Descriptors: Object.defineProperty, Getters, and Setters

## Overview

Every property on a JavaScript object is more than just a key-value pair. Under the hood, each property has a **descriptor** — a set of attributes that govern whether the property can be written, enumerated, or configured. Understanding property descriptors is fundamental to building robust APIs, implementing reactive data systems, creating sealed configuration objects, and debugging subtle property access bugs. `Object.defineProperty` and its sibling `Object.defineProperties` give you direct control over these attributes.

## Property Descriptor Types

JavaScript distinguishes between two mutually exclusive descriptor types. A property is either a **data descriptor** or an **accessor descriptor** — never both.

### Data Descriptor

A data descriptor holds a value and controls whether that value can be changed:

| Attribute | Default | Description |
|---|---|---|
| `value` | `undefined` | The actual stored value |
| `writable` | `false` | Whether the value can be changed with `=` |
| `enumerable` | `false` | Whether the property appears in `for...in`, `Object.keys`, etc. |
| `configurable` | `false` | Whether the descriptor itself can be changed or the property deleted |

### Accessor Descriptor

An accessor descriptor replaces the stored value with getter/setter functions:

| Attribute | Default | Description |
|---|---|---|
| `get` | `undefined` | Function called on property read |
| `set` | `undefined` | Function called on property write |
| `enumerable` | `false` | Same as above |
| `configurable` | `false` | Same as above |

Note: `value`/`writable` and `get`/`set` cannot coexist in the same descriptor.

## Object.defineProperty

### Basic Syntax

```javascript
Object.defineProperty(object, propertyName, descriptor);
```

Returns the modified object. Throws a `TypeError` if the operation violates an existing non-configurable constraint.

### Creating Read-Only Properties

```javascript
const config = {};

Object.defineProperty(config, "MAX_RETRIES", {
  value: 3,
  writable: false,
  enumerable: true,
  configurable: false
});

config.MAX_RETRIES = 99; // Silently fails in sloppy mode
// TypeError: Cannot assign to read only property in strict mode

console.log(config.MAX_RETRIES); // 3
```

### Creating Non-Enumerable Properties

Non-enumerable properties are "hidden" from `for...in`, `Object.keys`, `JSON.stringify`, and spread:

```javascript
const user = { name: "Alice", age: 30 };

Object.defineProperty(user, "_internal", {
  value: { createdAt: Date.now() },
  writable: true,
  enumerable: false,
  configurable: true
});

Object.keys(user);      // ["name", "age"] — _internal excluded
JSON.stringify(user);   // '{"name":"Alice","age":30}'
{ ...user };            // { name: "Alice", age: 30 }
"_internal" in user;    // true — `in` still finds it
user._internal;         // { createdAt: ... } — still accessible
```

### Shorthand vs defineProperty Defaults

The defaults differ significantly between literal assignment and `defineProperty`:

```javascript
const a = {};
a.foo = "bar"; // Assignment

Object.getOwnPropertyDescriptor(a, "foo");
// { value: "bar", writable: true, enumerable: true, configurable: true }

const b = {};
Object.defineProperty(b, "foo", { value: "bar" });
// Missing attributes default to false!
// { value: "bar", writable: false, enumerable: false, configurable: false }
```

This asymmetry is a common source of bugs. Always be explicit about all four attributes.

## Object.defineProperties

Define multiple properties at once:

```javascript
const Point = {};

Object.defineProperties(Point, {
  x: {
    value: 0,
    writable: true,
    enumerable: true,
    configurable: false
  },
  y: {
    value: 0,
    writable: true,
    enumerable: true,
    configurable: false
  },
  toString: {
    value() { return `(${this.x}, ${this.y})`; },
    enumerable: false,
    configurable: true
  }
});
```

## Inspecting Property Descriptors

### Object.getOwnPropertyDescriptor

```javascript
const obj = { name: "Alice" };
const desc = Object.getOwnPropertyDescriptor(obj, "name");
// { value: "Alice", writable: true, enumerable: true, configurable: true }
```

### Object.getOwnPropertyDescriptors

Returns all own property descriptors — useful for cloning with full fidelity:

```javascript
const source = {
  get computed() { return 42; },
  data: "hello"
};

const descriptors = Object.getOwnPropertyDescriptors(source);
// {
//   computed: { get: [Function], set: undefined, enumerable: true, configurable: true },
//   data: { value: "hello", writable: true, enumerable: true, configurable: true }
// }

// Full clone including getters/setters — unlike Object.assign
const clone = Object.defineProperties({}, descriptors);
clone.computed; // 42 — getter preserved (not the value)
```

`Object.assign` invokes the getter and copies the returned value. `Object.defineProperties` with the descriptors copies the getter function itself.

## Getters and Setters

### Object Literal Syntax

```javascript
const temperature = {
  _celsius: 0,

  get fahrenheit() {
    return this._celsius * 9/5 + 32;
  },

  set fahrenheit(value) {
    this._celsius = (value - 32) * 5/9;
  },

  get celsius() {
    return this._celsius;
  },

  set celsius(value) {
    if (value < -273.15) {
      throw new RangeError("Temperature below absolute zero");
    }
    this._celsius = value;
  }
};

temperature.celsius = 100;
temperature.fahrenheit; // 212
temperature.fahrenheit = 32;
temperature.celsius;    // 0
```

### defineProperty Accessor Form

```javascript
function createUser(name, dob) {
  const user = { name, _dob: dob };

  Object.defineProperty(user, "age", {
    get() {
      const now = new Date();
      const birth = new Date(this._dob);
      let age = now.getFullYear() - birth.getFullYear();
      const m = now.getMonth() - birth.getMonth();
      if (m < 0 || (m === 0 && now.getDate() < birth.getDate())) age--;
      return age;
    },
    enumerable: true,
    configurable: false
  });

  return user;
}

const alice = createUser("Alice", "1990-06-15");
alice.age; // Computed on each access
```

### Getters and Setters in Classes

```javascript
class Circle {
  #radius;

  constructor(radius) {
    this.#radius = radius;
  }

  get radius() {
    return this.#radius;
  }

  set radius(value) {
    if (value <= 0) throw new RangeError("Radius must be positive");
    this.#radius = value;
  }

  get area() {
    return Math.PI * this.#radius ** 2;
  }

  get circumference() {
    return 2 * Math.PI * this.#radius;
  }
}

const c = new Circle(5);
c.area;          // 78.539...
c.radius = 10;
c.area;          // 314.159...
c.radius = -1;   // RangeError
```

Class getters/setters are placed on `ClassName.prototype`, not on the instance:

```javascript
Object.getOwnPropertyDescriptor(Circle.prototype, "area");
// { get: [Function: get area], set: undefined, enumerable: false, configurable: true }
```

## Configurable and Writable Semantics

### Non-Configurable Properties

Once `configurable: false`, the property cannot be deleted and its descriptor cannot be changed (with one exception: `writable` can go from `true` to `false`):

```javascript
const obj = {};
Object.defineProperty(obj, "fixed", {
  value: 42,
  writable: true,
  configurable: false
});

// Allowed: tighten writable
Object.defineProperty(obj, "fixed", { writable: false }); // OK

// Not allowed: any other change
Object.defineProperty(obj, "fixed", { enumerable: true }); // TypeError
delete obj.fixed; // false — cannot delete
```

### Non-Writable Properties

Non-writable data properties reject assignment silently in sloppy mode, throw in strict mode:

```javascript
"use strict";
const obj = {};
Object.defineProperty(obj, "constant", { value: 99, writable: false });

obj.constant = 100; // TypeError: Cannot assign to read only property 'constant'
```

### Prototype Writable Check

Writing to an inherited non-writable property creates an own property instead of modifying the prototype... unless the inherited one is non-writable:

```javascript
const proto = {};
Object.defineProperty(proto, "x", { value: 1, writable: false });

const child = Object.create(proto);

// Sloppy mode: silent failure (does not create own property or modify proto)
child.x = 99;
child.x; // 1 — still inherited

// Strict mode: TypeError
"use strict";
child.x = 99; // TypeError
```

## Practical Patterns

### Lazy Initialization (Memoization via defineProperty)

Replace a property on first access and cache the result:

```javascript
class HeavyData {
  get processedList() {
    const result = expensiveComputation();
    // Replace the accessor with a plain data property on this specific instance
    Object.defineProperty(this, "processedList", {
      value: result,
      writable: false,
      configurable: false
    });
    return result;
  }
}

const hd = new HeavyData();
hd.processedList; // Runs computation once
hd.processedList; // Returns cached value directly
```

### Reactive / Observable Properties

A simplified Vue 2 / early Vue pattern using setters to trigger updates:

```javascript
function makeReactive(obj, key, callback) {
  let value = obj[key];

  Object.defineProperty(obj, key, {
    get() { return value; },
    set(newVal) {
      if (newVal !== value) {
        const oldVal = value;
        value = newVal;
        callback(key, oldVal, newVal);
      }
    },
    enumerable: true,
    configurable: true
  });
}

const state = { count: 0 };
makeReactive(state, "count", (key, old, next) => {
  console.log(`${key}: ${old} → ${next}`);
});

state.count = 1; // "count: 0 → 1"
state.count = 2; // "count: 1 → 2"
```

### Validation via Setters

```javascript
function createConstrainedValue(min, max, initial) {
  let value = initial;

  return {
    get value() { return value; },
    set value(v) {
      if (typeof v !== "number") throw new TypeError("Must be a number");
      if (v < min || v > max) throw new RangeError(`Must be between ${min} and ${max}`);
      value = v;
    }
  };
}

const percentage = createConstrainedValue(0, 100, 50);
percentage.value = 75;  // OK
percentage.value = 110; // RangeError
```

## Comparison Table

| Method | Scope | Use Case |
|---|---|---|
| `obj.key = value` | Single property, all defaults true | Normal usage |
| `Object.defineProperty` | Single property, full control | APIs, read-only, hidden props |
| `Object.defineProperties` | Multiple properties | Building objects programmatically |
| Literal getter/setter | Single accessor | Computed props on specific object |
| Class getter/setter | Prototype-level accessor | Shared computed props across instances |
| `Object.getOwnPropertyDescriptor` | Read single descriptor | Inspection / debugging |
| `Object.getOwnPropertyDescriptors` | Read all descriptors | Faithful cloning |

## Best Practices

### 1. Be Explicit With All Attributes When Using defineProperty

```javascript
// Bad — relies on false defaults, intention unclear
Object.defineProperty(obj, "key", { value: 1 });

// Good — all attributes explicit
Object.defineProperty(obj, "key", {
  value: 1,
  writable: false,
  enumerable: true,
  configurable: false
});
```

### 2. Use Getters for Derived Values, Not Cached Side Effects

Getters should be pure derivations. If you need caching, use the lazy initialization pattern rather than side effects inside a getter.

### 3. Setter Should Validate, Not Transform

Setters are for validation and notification. Complex transformations belong in explicit methods.

### 4. Prefer Class Syntax for Getters/Setters on Instances

Class syntax is more readable and tooling-friendly than `Object.defineProperty` for prototype-level accessors.

## Interview Questions

### Q1: What is the difference between `writable: false` and `configurable: false`?

**Answer:** `writable: false` prevents the value from being changed via assignment. `configurable: false` prevents the property descriptor from being modified and prevents the property from being deleted. They are orthogonal: a property can be writable but non-configurable (value can change but descriptor cannot), or non-writable but configurable (value cannot be changed, but the descriptor itself can still be reconfigured).

### Q2: Why does `Object.assign` not copy getters faithfully?

**Answer:** `Object.assign` iterates over the source's enumerable own properties and reads each one via `[[Get]]`, which invokes getters and produces their return value. The copy therefore has a plain data property containing the getter's result at the time of copying. To copy getters themselves, use `Object.defineProperties(target, Object.getOwnPropertyDescriptors(source))`.

### Q3: Can you make a property both non-writable and have a setter?

**Answer:** No. A property is either a data descriptor (has `value`/`writable`) or an accessor descriptor (has `get`/`set`). These are mutually exclusive. Defining both in the same call throws a `TypeError`.

### Q4: What happens when you assign to a non-writable property in strict mode vs sloppy mode?

**Answer:** In sloppy mode, the assignment fails silently — no error, no change to the value. In strict mode, a `TypeError` is thrown: "Cannot assign to read only property". The same behavior applies when writing to a property that is non-writable on a prototype — it does not create a shadowing own property.

### Q5: How did Vue 2 use property descriptors for reactivity, and why did Vue 3 switch to Proxy?

**Answer:** Vue 2 used `Object.defineProperty` to wrap each property in a getter/setter at component initialization time. The getter tracked dependencies (which computed properties read this value), and the setter triggered re-renders when the value changed. The limitation was that `defineProperty` only intercepts property access on keys that exist at initialization — dynamically added or deleted properties and array index mutations could not be detected, requiring `Vue.set()` workarounds. Vue 3 replaced this with `Proxy`, which intercepts all property access generically, including additions and deletions.

## Common Pitfalls

### 1. Forgetting That defineProperty Defaults Are All False

```javascript
// You get a non-enumerable, non-configurable, non-writable property
Object.defineProperty(obj, "key", { value: 1 });
// Probably not what you intended
```

### 2. Accidentally Making Properties Non-Deletable

```javascript
const cache = {};
Object.defineProperty(cache, "entries", {
  value: [],
  writable: true,
  configurable: false // Now you can never delete or redefine this
});

delete cache.entries; // false — silent failure
```

### 3. Infinite Recursion in Getters/Setters

```javascript
const obj = {
  get name() {
    return this.name; // Recursive! get name triggers get name again → stack overflow
  }
};

// Correct approach: use a backing variable
const obj2 = {
  _name: "Alice",
  get name() { return this._name; },
  set name(v) { this._name = v; }
};
```

### 4. defineProperty on Non-Configurable Properties

```javascript
const obj = {};
Object.defineProperty(obj, "x", { value: 1, configurable: false });
Object.defineProperty(obj, "x", { value: 2 }); // TypeError: Cannot redefine property
```

## Resources

### Official Documentation
- [MDN: Object.defineProperty](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)
- [MDN: Object.getOwnPropertyDescriptor](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/getOwnPropertyDescriptor)
- [MDN: Property accessors](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Property_accessors)
- [ECMAScript: Property Descriptor](https://tc39.es/ecma262/#sec-property-descriptor-specification-type)

### Articles and Guides
- [You Don't Know JS: Objects (Property Descriptors)](https://github.com/getify/You-Dont-Know-JS/blob/2nd-ed/objects-classes/ch3.md)
- [JavaScript Property Descriptors (2ality)](https://2ality.com/2012/10/javascript-properties.html)
- [How Vue 2 Reactivity Works](https://v2.vuejs.org/v2/guide/reactivity.html)
