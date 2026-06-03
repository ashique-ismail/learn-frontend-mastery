# Proxies and Reflect

## Table of Contents
- [Overview](#overview)
- [Proxy Basics](#proxy-basics)
- [Proxy Traps](#proxy-traps)
- [Reflect API](#reflect-api)
- [Metaprogramming Patterns](#metaprogramming-patterns)
- [Revocable Proxies](#revocable-proxies)
- [Common Misconceptions](#common-misconceptions)
- [Performance Implications](#performance-implications)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Overview

Proxies enable interception and customization of fundamental operations on objects. The Reflect API provides methods that correspond to proxy traps, creating a clean meta-programming interface. Together, they enable powerful patterns like validation, logging, reactive systems, and API virtualization.

## Proxy Basics

### Creating a Proxy

```javascript
// Syntax: new Proxy(target, handler)
const target = { name: 'Alice', age: 30 };

const handler = {
  get(target, property, receiver) {
    console.log(`Getting ${property}`);
    return target[property];
  }
};

const proxy = new Proxy(target, handler);

console.log(proxy.name); // Logs: 'Getting name', Returns: 'Alice'
console.log(proxy.age);  // Logs: 'Getting age', Returns: 30
```

### Simple Validation Example

```javascript
const user = {
  name: 'Bob',
  age: 25
};

const handler = {
  set(target, property, value) {
    if (property === 'age') {
      if (typeof value !== 'number' || value < 0 || value > 150) {
        throw new TypeError('Invalid age');
      }
    }
    target[property] = value;
    return true; // Indicate success
  }
};

const proxy = new Proxy(user, handler);

proxy.age = 30;  // Works
console.log(proxy.age); // 30

// proxy.age = -5;    // TypeError: Invalid age
// proxy.age = 'old'; // TypeError: Invalid age
```

### Default Behavior

```javascript
// Empty handler = transparent proxy (forwards all operations)
const target = { value: 42 };
const proxy = new Proxy(target, {}); // No traps

console.log(proxy.value); // 42 (default behavior)
proxy.value = 100;
console.log(target.value); // 100 (proxy modifies target)

// Proxy and target reference same object
console.log(proxy === target); // false (different objects)
target.newProp = 'new';
console.log(proxy.newProp); // 'new' (sees target changes)
```

## Proxy Traps

Proxy handlers can define "traps" that intercept operations.

### get Trap

```javascript
const handler = {
  get(target, property, receiver) {
    // target: original object
    // property: property being accessed
    // receiver: proxy (or object that inherited from proxy)
    
    if (property in target) {
      console.log(`Accessing ${property}`);
      return target[property];
    }
    
    return `Property ${property} doesn't exist`;
  }
};

const obj = new Proxy({ name: 'Alice' }, handler);

console.log(obj.name);    // Logs: 'Accessing name', Returns: 'Alice'
console.log(obj.missing); // Returns: "Property missing doesn't exist"
```

### set Trap

```javascript
const validator = {
  set(target, property, value, receiver) {
    console.log(`Setting ${property} to ${value}`);
    
    // Validation logic
    if (property === 'age' && typeof value !== 'number') {
      throw new TypeError('Age must be a number');
    }
    
    if (property === 'email' && !value.includes('@')) {
      throw new TypeError('Invalid email');
    }
    
    target[property] = value;
    return true; // Required: indicate success
  }
};

const user = new Proxy({}, validator);

user.name = 'Alice';  // Works
user.age = 30;        // Works
// user.age = 'thirty'; // TypeError
user.email = 'alice@example.com'; // Works
// user.email = 'invalid'; // TypeError
```

### has Trap (in operator)

```javascript
const handler = {
  has(target, property) {
    // Intercept 'in' operator
    if (property.startsWith('_')) {
      return false; // Hide private properties
    }
    return property in target;
  }
};

const obj = new Proxy({
  public: 'visible',
  _private: 'hidden'
}, handler);

console.log('public' in obj);   // true
console.log('_private' in obj); // false (hidden by trap)
console.log(obj._private);      // 'hidden' (can still access directly)
```

### deleteProperty Trap

```javascript
const handler = {
  deleteProperty(target, property) {
    if (property.startsWith('_')) {
      throw new Error(`Cannot delete private property ${property}`);
    }
    delete target[property];
    return true;
  }
};

const obj = new Proxy({
  public: 'can delete',
  _private: 'protected'
}, handler);

delete obj.public;   // Works
console.log(obj.public); // undefined

// delete obj._private; // Error: Cannot delete private property _private
```

### apply Trap (function calls)

```javascript
function sum(a, b) {
  return a + b;
}

const handler = {
  apply(target, thisArg, argumentsList) {
    console.log(`Called with ${argumentsList.join(', ')}`);
    
    // Modify arguments
    const sanitized = argumentsList.map(Number);
    
    return target.apply(thisArg, sanitized);
  }
};

const proxy = new Proxy(sum, handler);

console.log(proxy(1, 2));      // Logs: 'Called with 1, 2', Returns: 3
console.log(proxy('5', '10')); // Logs: 'Called with 5, 10', Returns: 15
```

### construct Trap (new operator)

```javascript
class Person {
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }
}

const handler = {
  construct(target, argumentsList, newTarget) {
    console.log('Creating new instance');
    
    // Validate arguments
    if (argumentsList.length < 2) {
      throw new Error('Person requires name and age');
    }
    
    return new target(...argumentsList);
  }
};

const ProxyPerson = new Proxy(Person, handler);

const person = new ProxyPerson('Alice', 30); // Works
console.log(person); // Person { name: 'Alice', age: 30 }

// const invalid = new ProxyPerson('Bob'); // Error: Person requires name and age
```

### getPrototypeOf and setPrototypeOf Traps

```javascript
const handler = {
  getPrototypeOf(target) {
    console.log('Getting prototype');
    return Object.getPrototypeOf(target);
  },
  
  setPrototypeOf(target, prototype) {
    console.log('Setting prototype');
    // Prevent prototype modification
    return false;
  }
};

const obj = new Proxy({}, handler);

Object.getPrototypeOf(obj); // Logs: 'Getting prototype'

try {
  Object.setPrototypeOf(obj, Array.prototype);
} catch (e) {
  console.log('Cannot set prototype'); // Prevented by trap
}
```

### ownKeys Trap

```javascript
const handler = {
  ownKeys(target) {
    // Filter out private properties
    return Object.keys(target).filter(key => !key.startsWith('_'));
  }
};

const obj = new Proxy({
  public1: 1,
  public2: 2,
  _private: 'hidden'
}, handler);

console.log(Object.keys(obj)); // ['public1', 'public2']
for (let key in obj) {
  console.log(key); // Only public1 and public2
}
```

### getOwnPropertyDescriptor Trap

```javascript
const handler = {
  getOwnPropertyDescriptor(target, property) {
    if (property.startsWith('_')) {
      return undefined; // Hide private properties
    }
    return Object.getOwnPropertyDescriptor(target, property);
  }
};

const obj = new Proxy({
  public: 'visible',
  _private: 'hidden'
}, handler);

console.log(Object.getOwnPropertyDescriptor(obj, 'public'));
// { value: 'visible', writable: true, enumerable: true, configurable: true }

console.log(Object.getOwnPropertyDescriptor(obj, '_private'));
// undefined (hidden)
```

### defineProperty Trap

```javascript
const handler = {
  defineProperty(target, property, descriptor) {
    if (property.startsWith('_')) {
      throw new Error('Cannot define private properties');
    }
    
    Object.defineProperty(target, property, descriptor);
    return true;
  }
};

const obj = new Proxy({}, handler);

Object.defineProperty(obj, 'public', {
  value: 'visible',
  writable: true
}); // Works

// Object.defineProperty(obj, '_private', { value: 'hidden' });
// Error: Cannot define private properties
```

## Reflect API

The Reflect API provides methods that mirror proxy traps, enabling default behavior and forwarding.

### Reflect Methods

```javascript
// Reflect methods correspond to proxy traps:
// - Reflect.get(target, property, receiver)
// - Reflect.set(target, property, value, receiver)
// - Reflect.has(target, property)
// - Reflect.deleteProperty(target, property)
// - Reflect.apply(target, thisArg, argumentsList)
// - Reflect.construct(target, argumentsList, newTarget)
// - Reflect.getPrototypeOf(target)
// - Reflect.setPrototypeOf(target, prototype)
// - Reflect.ownKeys(target)
// - Reflect.getOwnPropertyDescriptor(target, property)
// - Reflect.defineProperty(target, property, descriptor)
// - Reflect.preventExtensions(target)
// - Reflect.isExtensible(target)
```

### Using Reflect in Traps

```javascript
// Best practice: Use Reflect for default behavior
const handler = {
  get(target, property, receiver) {
    console.log(`Getting ${property}`);
    
    // Use Reflect instead of target[property]
    return Reflect.get(target, property, receiver);
  },
  
  set(target, property, value, receiver) {
    console.log(`Setting ${property} to ${value}`);
    
    // Validation
    if (property === 'age' && typeof value !== 'number') {
      return false; // Indicate failure
    }
    
    // Use Reflect for default behavior
    return Reflect.set(target, property, value, receiver);
  }
};

const obj = new Proxy({ name: 'Alice' }, handler);

obj.name; // Logs: 'Getting name'
obj.age = 30; // Logs: 'Setting age to 30'
```

### Why Use Reflect?

```javascript
// Problem: Using target[property] can break receiver
const parent = {
  get value() {
    return this.name; // 'this' context matters
  }
};

const child = {
  name: 'Child',
  __proto__: parent
};

// Without Reflect (incorrect)
const badHandler = {
  get(target, property, receiver) {
    return target[property]; // 'this' in getter will be target, not receiver
  }
};

const badProxy = new Proxy(child, badHandler);
console.log(badProxy.value); // undefined (wrong 'this')

// With Reflect (correct)
const goodHandler = {
  get(target, property, receiver) {
    return Reflect.get(target, property, receiver); // Correct 'this'
  }
};

const goodProxy = new Proxy(child, goodHandler);
console.log(goodProxy.value); // 'Child' (correct 'this')
```

### Reflect vs Object Methods

```javascript
// Reflect methods return boolean success/failure
// Object methods throw errors

// Object.defineProperty throws on failure
try {
  Object.defineProperty({}, 'prop', { value: 1 });
} catch (e) {
  console.log('Failed');
}

// Reflect.defineProperty returns boolean
const success = Reflect.defineProperty({}, 'prop', { value: 1 });
console.log(success); // true

// Better for conditional logic
if (Reflect.defineProperty(obj, 'prop', descriptor)) {
  console.log('Property defined');
} else {
  console.log('Failed to define property');
}
```

## Metaprogramming Patterns

### Observable/Reactive Objects

```javascript
function createObservable(target, callback) {
  return new Proxy(target, {
    set(target, property, value, receiver) {
      const oldValue = target[property];
      const result = Reflect.set(target, property, value, receiver);
      
      if (result && oldValue !== value) {
        callback(property, oldValue, value);
      }
      
      return result;
    }
  });
}

const data = createObservable({ count: 0 }, (prop, oldVal, newVal) => {
  console.log(`${prop} changed from ${oldVal} to ${newVal}`);
});

data.count = 1; // Logs: 'count changed from 0 to 1'
data.count = 2; // Logs: 'count changed from 1 to 2'
```

### Negative Array Indices

```javascript
function createNegativeArray(arr) {
  return new Proxy(arr, {
    get(target, property, receiver) {
      let index = Number(property);
      
      if (Number.isInteger(index) && index < 0) {
        // Convert negative index: -1 → length-1
        index = target.length + index;
        return target[index];
      }
      
      return Reflect.get(target, property, receiver);
    }
  });
}

const arr = createNegativeArray([1, 2, 3, 4, 5]);

console.log(arr[-1]); // 5 (last element)
console.log(arr[-2]); // 4 (second-to-last)
console.log(arr[0]);  // 1 (normal indexing still works)
```

### Default Values

```javascript
function withDefaults(target, defaults) {
  return new Proxy(target, {
    get(target, property, receiver) {
      if (property in target) {
        return Reflect.get(target, property, receiver);
      }
      return defaults[property];
    }
  });
}

const config = withDefaults(
  { api: 'https://api.example.com' },
  { timeout: 5000, retries: 3 }
);

console.log(config.api);     // 'https://api.example.com' (from target)
console.log(config.timeout); // 5000 (from defaults)
console.log(config.retries); // 3 (from defaults)
```

### Type Checking/Validation

```javascript
function createTypedObject(schema) {
  return new Proxy({}, {
    set(target, property, value, receiver) {
      const type = schema[property];
      
      if (!type) {
        throw new Error(`Unknown property: ${property}`);
      }
      
      if (typeof value !== type) {
        throw new TypeError(
          `Property ${property} must be ${type}, got ${typeof value}`
        );
      }
      
      return Reflect.set(target, property, value, receiver);
    }
  });
}

const user = createTypedObject({
  name: 'string',
  age: 'number',
  active: 'boolean'
});

user.name = 'Alice';  // Works
user.age = 30;        // Works
user.active = true;   // Works

// user.age = 'thirty'; // TypeError: Property age must be number
// user.unknown = 1;    // Error: Unknown property: unknown
```

### Hidden Properties

```javascript
function createHiddenProps(target, hiddenKeys) {
  return new Proxy(target, {
    has(target, property) {
      if (hiddenKeys.includes(property)) {
        return false;
      }
      return Reflect.has(target, property);
    },
    
    ownKeys(target) {
      return Reflect.ownKeys(target).filter(
        key => !hiddenKeys.includes(key)
      );
    },
    
    getOwnPropertyDescriptor(target, property) {
      if (hiddenKeys.includes(property)) {
        return undefined;
      }
      return Reflect.getOwnPropertyDescriptor(target, property);
    }
  });
}

const obj = createHiddenProps(
  { public: 1, secret: 2, hidden: 3 },
  ['secret', 'hidden']
);

console.log('public' in obj); // true
console.log('secret' in obj); // false
console.log(Object.keys(obj)); // ['public']
console.log(obj.secret); // 2 (still accessible directly)
```

### API Virtualization

```javascript
// Create API client that generates methods on-the-fly
function createAPIClient(baseURL) {
  return new Proxy({}, {
    get(target, property, receiver) {
      return (...args) => {
        const url = `${baseURL}/${property}`;
        console.log(`API call: ${url}`, args);
        
        // Actual implementation would use fetch()
        return Promise.resolve({ url, args });
      };
    }
  });
}

const api = createAPIClient('https://api.example.com');

api.getUser(123); // Logs: API call: https://api.example.com/getUser [123]
api.createPost({ title: 'Hello' }); // Logs: API call: https://api.example.com/createPost [{ title: 'Hello' }]

// No need to define methods explicitly!
```

## Revocable Proxies

### Creating Revocable Proxies

```javascript
const target = { value: 42 };

const { proxy, revoke } = Proxy.revocable(target, {
  get(target, property, receiver) {
    console.log(`Getting ${property}`);
    return Reflect.get(target, property, receiver);
  }
});

console.log(proxy.value); // Logs: 'Getting value', Returns: 42

// Revoke proxy access
revoke();

try {
  console.log(proxy.value); // TypeError: Cannot perform 'get' on a proxy that has been revoked
} catch (e) {
  console.log('Proxy revoked');
}
```

### Use Case: Temporary Access

```javascript
function grantTemporaryAccess(obj, duration) {
  const { proxy, revoke } = Proxy.revocable(obj, {});
  
  setTimeout(revoke, duration);
  
  return proxy;
}

const sensitiveData = { password: 'secret123' };
const tempAccess = grantTemporaryAccess(sensitiveData, 5000);

console.log(tempAccess.password); // Works for 5 seconds
// After 5 seconds: TypeError when accessing tempAccess
```

## Common Misconceptions

### Misconception 1: "Proxies Copy Objects"

**Reality**: Proxies wrap objects, they don't copy them. Changes through proxy affect original.

```javascript
const target = { value: 1 };
const proxy = new Proxy(target, {});

proxy.value = 2;
console.log(target.value); // 2 (modified through proxy)

target.value = 3;
console.log(proxy.value); // 3 (sees target changes)
```

### Misconception 2: "All Operations Are Trapped"

**Reality**: Only explicitly defined traps intercept operations. Others use default behavior.

```javascript
const proxy = new Proxy({ value: 1 }, {
  get(target, property, receiver) {
    console.log('get trapped');
    return Reflect.get(target, property, receiver);
  }
  // No set trap
});

proxy.value; // Logs: 'get trapped'
proxy.value = 2; // No log, default behavior
```

### Misconception 3: "Proxies Work with All Objects"

**Reality**: Some objects have internal slots that proxies can't intercept (Date, Map, Set, etc.).

```javascript
const date = new Date();
const proxy = new Proxy(date, {});

// proxy.getTime(); // TypeError: 'this' is not a Date object

// Solution: Bind methods or use Reflect
const proxy2 = new Proxy(date, {
  get(target, property, receiver) {
    const value = Reflect.get(target, property, receiver);
    if (typeof value === 'function') {
      return value.bind(target); // Bind to original target
    }
    return value;
  }
});

console.log(proxy2.getTime()); // Works
```

### Misconception 4: "Reflect Is Just for Proxies"

**Reality**: Reflect methods are useful anywhere for cleaner meta-programming.

```javascript
// Instead of try-catch with Object.defineProperty
try {
  Object.defineProperty(obj, 'prop', { value: 1 });
  console.log('Success');
} catch (e) {
  console.log('Failed');
}

// Use Reflect for boolean return
if (Reflect.defineProperty(obj, 'prop', { value: 1 })) {
  console.log('Success');
} else {
  console.log('Failed');
}
```

## Performance Implications

### Proxy Overhead

```javascript
const obj = { value: 1 };
const proxy = new Proxy(obj, {
  get(target, property, receiver) {
    return Reflect.get(target, property, receiver);
  }
});

// Direct access (fastest)
console.time('direct');
for (let i = 0; i < 1000000; i++) {
  obj.value;
}
console.timeEnd('direct'); // ~2ms

// Proxy access (slower, ~10-20x)
console.time('proxy');
for (let i = 0; i < 1000000; i++) {
  proxy.value;
}
console.timeEnd('proxy'); // ~20-40ms

// Proxies have overhead; use when needed, not everywhere
```

### Optimization Tips

```javascript
// BAD: Proxy everything
function makeReactive(obj) {
  for (let key in obj) {
    if (typeof obj[key] === 'object') {
      obj[key] = makeReactive(obj[key]); // Recursive proxies
    }
  }
  return new Proxy(obj, handler);
}

// GOOD: Lazy proxying
function makeReactiveLazy(obj) {
  return new Proxy(obj, {
    get(target, property, receiver) {
      const value = Reflect.get(target, property, receiver);
      
      // Only wrap objects when accessed
      if (typeof value === 'object' && value !== null) {
        return makeReactiveLazy(value);
      }
      
      return value;
    }
  });
}
```

### When to Use Proxies

```javascript
// GOOD use cases:
// - Validation, logging, debugging
// - Reactive systems (Vue 3, MobX)
// - API virtualization
// - Access control

// AVOID for:
// - Hot paths in performance-critical code
// - Simple object operations
// - When direct access suffices
```

## Interview Questions

### Question 1: What is a Proxy and what problem does it solve?

**Answer**: A Proxy is an object that wraps another object and intercepts operations on it through "traps" (handlers). It solves several problems: validation (checking values before setting), logging/debugging (tracking property access), implementing reactive systems (like Vue 3's reactivity), virtualizing APIs, and controlling access to objects. Unlike getters/setters, proxies intercept all operations (get, set, has, delete, etc.) dynamically without modifying the original object.

### Question 2: Name at least 6 proxy traps and their purposes

**Answer**:
1. `get`: Intercepts property access
2. `set`: Intercepts property assignment
3. `has`: Intercepts `in` operator
4. `deleteProperty`: Intercepts `delete` operator
5. `apply`: Intercepts function calls
6. `construct`: Intercepts `new` operator
7. `ownKeys`: Intercepts `Object.keys()`, `for...in`
8. `getOwnPropertyDescriptor`: Intercepts property descriptor access

### Question 3: What is the Reflect API and why use it with Proxies?

**Answer**: Reflect is a built-in object providing methods that mirror proxy traps. Use it in proxy handlers because: 1) It provides correct default behavior (especially important for `receiver` in getters/setters), 2) Methods return boolean success/failure instead of throwing errors, 3) It makes code more explicit and maintainable, and 4) It ensures proper `this` binding when dealing with inheritance chains.

### Question 4: What are revocable proxies and when would you use them?

**Answer**: Revocable proxies are created with `Proxy.revocable()` and return both a proxy and a `revoke` function. Calling `revoke()` permanently disables the proxy - any operation on it throws an error. Use cases include: temporary access grants (expire after time), security (revoke access when done), resource cleanup (prevent memory leaks), and lending objects with ability to revoke later.

### Question 5: Can you proxy all JavaScript objects? What are the limitations?

**Answer**: No, objects with internal slots can't be fully proxied (Date, Map, Set, WeakMap, WeakSet, typed arrays). These objects have internal methods that expect `this` to be the actual object, not a proxy. Solution: bind methods to the original target, or handle specific methods in the trap to ensure correct `this` context.

## Key Takeaways

1. **Proxies wrap objects**: They intercept operations without modifying the original

2. **Traps are handlers**: Each trap corresponds to an internal operation

3. **Reflect provides defaults**: Use Reflect in traps for correct default behavior

4. **Revocable proxies**: Can permanently disable access to proxied objects

5. **Performance overhead**: Proxies are slower than direct access; use judiciously

6. **Powerful metaprogramming**: Enable validation, reactivity, virtualization, access control

7. **Not all objects can be proxied**: Internal slots (Date, Map, Set) require special handling

8. **Boolean returns**: Most traps must return boolean indicating success/failure

9. **Receiver parameter matters**: Ensures correct `this` binding in getters/setters

10. **Reflect vs Object methods**: Reflect returns booleans, Object throws errors

## Resources

### Official Documentation
- [MDN: Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy)
- [MDN: Reflect](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Reflect)
- [ECMAScript Specification: Proxy Objects](https://tc39.es/ecma262/#sec-proxy-objects)

### Articles
- "JavaScript Proxies in Depth" by Pony Foo
- "Meta Programming with Proxies" by Axel Rauschmayer
- "Practical Uses for Proxies" by Keith Cirkel

### Books
- "Exploring ES6" by Axel Rauschmayer
- "You Don't Know JS: ES6 & Beyond" by Kyle Simpson
- "JavaScript: The Definitive Guide" by David Flanagan

### Real-World Usage
- Vue 3 reactivity system
- MobX state management
- Immer immutable updates
- Various validation libraries
