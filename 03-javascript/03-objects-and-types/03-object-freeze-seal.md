# Object.freeze, Object.seal, and Object.preventExtensions

## The Idea

**In plain English:** JavaScript lets you lock down objects so that their contents cannot be changed, partially or fully. `freeze` makes an object completely read-only, `seal` locks its shape (no adding or removing properties) but still lets you update values, and `preventExtensions` only stops new properties from being added.

**Real-world analogy:** Think of a museum display case with exhibits inside. A fully locked glass case (freeze) means you cannot add new items, remove existing ones, or move anything around. A sealed display tray (seal) means you cannot add or remove items, but you can swap one exhibit for another of the same kind. An open shelf with a "no new items" sign (preventExtensions) means no new things can be placed there, but you can rearrange, remove, or replace what is already on it.

- The display case = the JavaScript object
- The items on display = the properties and their values
- The level of locking = `freeze`, `seal`, or `preventExtensions`

---

## Overview

JavaScript provides three built-in methods for restricting object mutability at the descriptor level. They form a spectrum from mildest to most restrictive: `preventExtensions` blocks new properties from being added, `seal` additionally makes all existing properties non-configurable, and `freeze` further makes all data properties non-writable. These operations are permanent and irreversible — once applied, they cannot be undone. Understanding their precise semantics is critical for building immutable configuration objects, protecting API contracts, and reasoning about data integrity in large applications.

## Object.preventExtensions

### Behavior

`Object.preventExtensions(obj)` prevents new own properties from being added to the object. Existing properties remain untouched — still writable, configurable, and deletable.

```javascript
const obj = { x: 1, y: 2 };
Object.preventExtensions(obj);

obj.z = 3;           // Silently fails in sloppy mode
obj.x = 99;          // OK — existing property still writable
delete obj.y;        // OK — existing property still deletable

console.log(obj);    // { x: 99 }
console.log(Object.isExtensible(obj)); // false
```

In strict mode, adding a new property throws:

```javascript
"use strict";
const obj = Object.preventExtensions({ a: 1 });
obj.b = 2; // TypeError: Cannot add property b, object is not extensible
```

### Effect on Prototype

`preventExtensions` only affects own properties. The prototype is unaffected:

```javascript
const proto = {};
const child = Object.create(proto);
Object.preventExtensions(child);

child.newProp = 1; // TypeError — cannot add to child
proto.newProp = 1; // OK — proto is still extensible

child.newProp; // 1 — found via prototype chain
```

### Use Case

Useful when you want to close off an object's shape while still allowing existing values to change. Common in module patterns where you want to prevent consumers from accidentally attaching properties.

## Object.seal

### Behavior

`Object.seal(obj)` calls `preventExtensions` and then marks every existing own property as `configurable: false`. Properties can still be read and written (if they were writable), but they cannot be deleted, reconfigured, or have their type changed (data ↔ accessor).

```javascript
const config = {
  host: "localhost",
  port: 3000,
  debug: false
};

Object.seal(config);

config.port = 8080;          // OK — still writable
config.debug = true;         // OK — still writable
config.newField = "x";       // Fails — not extensible
delete config.host;          // Fails — non-configurable
Object.defineProperty(config, "port", { enumerable: false }); // TypeError

console.log(Object.isSealed(config)); // true
```

### What Seal Does to Descriptors

```javascript
const obj = { a: 1 };
Object.getOwnPropertyDescriptor(obj, "a");
// { value: 1, writable: true, enumerable: true, configurable: true }

Object.seal(obj);
Object.getOwnPropertyDescriptor(obj, "a");
// { value: 1, writable: true, enumerable: true, configurable: false }
// Only configurable changed
```

### Checking Seal Status

```javascript
const s = Object.seal({ x: 1 });
Object.isSealed(s);           // true
Object.isExtensible(s);       // false — implied by seal
Object.isFrozen(s);           // false — still writable
```

A non-extensible object with all non-configurable, non-writable properties is also sealed and frozen.

### Use Case

Seal is appropriate when you want to lock an object's shape (no new or deleted properties) but still allow values to be updated. Useful for state objects in systems where the schema should be stable but the data is mutable.

## Object.freeze

### Behavior

`Object.freeze(obj)` seals the object and additionally marks every data property as `writable: false`. The result is a fully locked object: no additions, no deletions, no reconfiguration, no value changes.

```javascript
const settings = Object.freeze({
  maxConnections: 10,
  timeout: 5000,
  baseUrl: "https://api.example.com"
});

settings.timeout = 9999;            // Silently fails
settings.newProp = "x";             // Silently fails
delete settings.baseUrl;            // Silently fails

console.log(settings.timeout);      // 5000 — unchanged
console.log(Object.isFrozen(settings)); // true
```

In strict mode, all failed mutations throw `TypeError`.

### What Freeze Does to Descriptors

```javascript
const obj = { count: 0, label: "test" };
Object.freeze(obj);

Object.getOwnPropertyDescriptor(obj, "count");
// { value: 0, writable: false, enumerable: true, configurable: false }
// Both writable and configurable are now false
```

### Accessor Properties and Freeze

Freeze makes data properties non-writable. Accessor properties (getters/setters) are unaffected by the `writable` flag (they don't have one). However, their `configurable` becomes `false`, so the accessor itself cannot be changed:

```javascript
const obj = {
  _x: 0,
  get x() { return this._x; },
  set x(v) { this._x = v; }
};

Object.freeze(obj);
obj.x = 99;       // Invokes the setter — works! But _x is now frozen too
// Wait: _x is a data property, so it's frozen:
// _x: { value: 0, writable: false ... }
// So the setter will silently fail or throw when trying to write _x
```

This is a subtle gotcha: freeze does not prevent setters from being invoked, but if the setter modifies another property on the same object and that property is also frozen, the write will fail.

### Shallow Nature of freeze

This is the most important limitation. `Object.freeze` is one level deep:

```javascript
const state = Object.freeze({
  user: { name: "Alice", role: "admin" }, // Not frozen!
  settings: { theme: "dark" }              // Not frozen!
});

state.user = {};              // Fails — own property 'user' is frozen
state.user.name = "Bob";     // Succeeds — nested object is not frozen
state.settings.theme = "light"; // Succeeds

console.log(state.user.name); // "Bob"
```

### Deep Freeze

A recursive freeze that traverses all nested objects:

```javascript
function deepFreeze(obj) {
  if (obj === null || typeof obj !== "object" || Object.isFrozen(obj)) {
    return obj;
  }

  Object.freeze(obj);

  Object.getOwnPropertyNames(obj).forEach(name => {
    const value = obj[name];
    if (value && typeof value === "object") {
      deepFreeze(value);
    }
  });

  return obj;
}

const config = deepFreeze({
  server: { host: "localhost", port: 3000 },
  db: { name: "myapp", poolSize: 5 }
});

config.server.port = 9999; // Silently fails (or TypeError in strict mode)
console.log(config.server.port); // 3000
```

Note: deep freeze mutates the objects in place, so it should only be applied to objects you own.

### Freeze and Arrays

Arrays are objects, so freeze works the same way:

```javascript
const arr = Object.freeze([1, 2, 3]);

arr.push(4);    // TypeError: Cannot add property 3, object is not extensible
arr[0] = 99;    // TypeError: Cannot assign to read only property '0'
arr.length = 0; // TypeError: Cannot assign to read only property 'length'

// The array is immutable — use spread to create modified copies
const newArr = [...arr, 4]; // [1, 2, 3, 4]
```

## Object.isFrozen, isSealed, isExtensible

These methods test the effective state of an object:

```javascript
const obj = { a: 1 };

Object.isExtensible(obj); // true
Object.isSealed(obj);     // false
Object.isFrozen(obj);     // false

Object.preventExtensions(obj);
Object.isExtensible(obj); // false
Object.isSealed(obj);     // false — properties still configurable

Object.seal(obj);
Object.isSealed(obj);     // true
Object.isFrozen(obj);     // false — still writable

Object.freeze(obj);
Object.isFrozen(obj);     // true
Object.isSealed(obj);     // true — frozen implies sealed
Object.isExtensible(obj); // false — sealed implies non-extensible
```

### Edge Case: Empty Objects

An empty non-extensible object is trivially both sealed and frozen:

```javascript
const empty = Object.preventExtensions({});
Object.isSealed(empty);  // true — no properties to be configurable
Object.isFrozen(empty);  // true — no properties to be writable
```

## Comparison Table

| Method | New Properties | Delete Properties | Reconfigure Descriptors | Change Values |
|---|---|---|---|---|
| (Normal object) | Allowed | Allowed | Allowed | Allowed |
| `preventExtensions` | Blocked | Allowed | Allowed | Allowed |
| `seal` | Blocked | Blocked | Blocked | Allowed |
| `freeze` | Blocked | Blocked | Blocked | Blocked |

| Method | `isExtensible` | `isSealed` | `isFrozen` |
|---|---|---|---|
| `preventExtensions` | false | false | false |
| `seal` | false | true | false |
| `freeze` | false | true | true |

## Performance Implications

### V8 Optimization

Frozen objects can be more efficiently optimized by JavaScript engines. V8 can represent a frozen object's properties as immutable constants, enabling inlining and removing the need for change tracking. For performance-critical code that reads many fixed constants, freeze can be a micro-optimization.

```javascript
// Engine can optimize reads from frozen objects more aggressively
const CONSTANTS = Object.freeze({
  PI: 3.14159265358979,
  E: 2.71828182845905,
  PHI: 1.61803398874989
});
```

### Copy-on-Write Pattern

In immutable state patterns, frozen objects enable safe sharing without defensive copies:

```javascript
function updateUser(user, changes) {
  // user is frozen — we must return a new object
  return Object.freeze({ ...user, ...changes });
}

const user = Object.freeze({ id: 1, name: "Alice", role: "user" });
const updated = updateUser(user, { role: "admin" });
// user is unchanged, updated is a new frozen object
```

## Best Practices

### 1. Use freeze for Constants and Configuration

```javascript
// Module-level constants
export const HTTP_METHODS = Object.freeze({
  GET: "GET",
  POST: "POST",
  PUT: "PUT",
  DELETE: "DELETE",
  PATCH: "PATCH"
});

// Configuration that should never change at runtime
export const APP_CONFIG = Object.freeze({
  apiVersion: "v2",
  maxPageSize: 100
});
```

### 2. Use seal When Shape Is Fixed but Data Changes

```javascript
// A state object whose schema is known at creation
function createState(initial) {
  return Object.seal({ ...initial });
}

const state = createState({ count: 0, loading: false, error: null });
state.count++;       // OK — known property
state.extra = "x";   // Fails — unknown property caught immediately
```

### 3. Remember to Deep Freeze Nested Objects

Surface-level `freeze` gives false confidence. Use `deepFreeze` for truly immutable objects, or use an immutability library like Immer for complex state.

### 4. Always Use Strict Mode With Frozen Objects

Without strict mode, violations fail silently, which is harder to debug than explicit errors.

## Interview Questions

### Q1: What is the difference between Object.freeze and const?

**Answer:** `const` prevents the variable binding from being reassigned — you cannot point the variable at a different object. It says nothing about the object itself. `Object.freeze` prevents the object's properties from being modified. They address different things: `const obj = {}` prevents `obj = somethingElse` but allows `obj.x = 1`. `Object.freeze(obj)` allows reassigning the variable `obj` but prevents any mutation of the object's properties.

### Q2: Is Object.freeze deep or shallow?

**Answer:** Shallow. `Object.freeze` only freezes the direct properties of the object. Nested objects referenced by those properties are not frozen unless you explicitly freeze them too. For deep immutability, you need a recursive deep freeze function or an immutability library.

### Q3: Can you unfreeze an object?

**Answer:** No. Freeze, seal, and preventExtensions are all irreversible. Once an object is frozen, it stays frozen for its entire lifetime. The `configurable: false` flag prevents the descriptor from being changed, which means you cannot restore `writable: true` or delete the property constraints.

### Q4: What happens when you call freeze on a class instance?

**Answer:** The instance's own properties are frozen, but the prototype is not. Methods defined on the prototype remain accessible and callable. If a prototype method tries to modify an own property on the frozen instance, the write fails. Static properties on the class are also unaffected.

### Q5: How does Object.seal relate to the configurable descriptor attribute?

**Answer:** `Object.seal` sets `configurable: false` on every own property of the object. This is exactly what makes deletion impossible (a property with `configurable: false` cannot be deleted) and what makes descriptor changes impossible. It also calls `preventExtensions`, blocking new properties.

## Common Pitfalls

### 1. False Security From Shallow Freeze

```javascript
const config = Object.freeze({
  db: { host: "localhost", password: "secret" } // NOT frozen
});

config.db.password = "hacked"; // Succeeds!
// Always deep freeze or use Object.freeze on nested objects explicitly
```

### 2. Mutating Frozen Objects Silently in Sloppy Mode

```javascript
// No strict mode — dangerous
const obj = Object.freeze({ x: 1 });
obj.x = 2; // Silent failure
console.log(obj.x); // 1 — looks like a bug in the mutation code
```

### 3. Expecting freeze to Affect Array Methods

```javascript
const frozen = Object.freeze([1, 2, 3]);
frozen.sort(); // TypeError: Cannot assign to read only property '0'
// All in-place array mutations fail: push, pop, splice, reverse, sort, fill
// Use non-mutating alternatives: [...frozen].sort()
```

### 4. Confusing preventExtensions With seal

```javascript
const obj = Object.preventExtensions({ a: 1 });
delete obj.a; // Succeeds! preventExtensions does not prevent deletion
// If you want to prevent deletion, use seal
```

## Resources

### Official Documentation
- [MDN: Object.freeze](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/freeze)
- [MDN: Object.seal](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/seal)
- [MDN: Object.preventExtensions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/preventExtensions)
- [ECMAScript: ValidateAndApplyPropertyDescriptor](https://tc39.es/ecma262/#sec-validateandapplypropertydescriptor)

### Articles and Guides
- [Immutability in JavaScript (2ality)](https://2ality.com/2018/01/object-frozen.html)
- [You Don't Know JS: Objects (Immutability)](https://github.com/getify/You-Dont-Know-JS/blob/2nd-ed/objects-classes/ch3.md)
- [Immer: Immutable State Management](https://immerjs.github.io/immer/)
