# this Binding Rules

## Table of Contents
- [Overview](#overview)
- [Default Binding](#default-binding)
- [Implicit Binding](#implicit-binding)
- [Explicit Binding](#explicit-binding)
- [new Binding](#new-binding)
- [Arrow Functions](#arrow-functions)
- [Binding Precedence](#binding-precedence)
- [call, apply, bind Internals](#call-apply-bind-internals)
- [Common Misconceptions](#common-misconceptions)
- [Performance Implications](#performance-implications)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Overview

The `this` keyword in JavaScript is one of the most misunderstood features. Unlike most languages where `this` is lexically bound (determined by where code is written), JavaScript's `this` is dynamically bound based on how a function is called. Understanding the four binding rules and their precedence is crucial for mastering JavaScript.

## Default Binding

Default binding applies when a function is called standalone (not as a method, with `new`, or with explicit binding).

### Global Context

```javascript
function showThis() {
  console.log(this);
}

showThis(); // Window (browser) or global (Node.js) in non-strict mode

// In strict mode
'use strict';
function showThisStrict() {
  console.log(this);
}

showThisStrict(); // undefined (default binding doesn't apply in strict mode)
```

### Common Gotcha: Lost Context

```javascript
const obj = {
  value: 42,
  getValue() {
    console.log(this.value);
  }
};

obj.getValue(); // 42 (implicit binding)

// Extract method - loses context!
const getValue = obj.getValue;
getValue(); // undefined (default binding, 'this' is global/undefined)

// Why? Function call site matters, not where function was defined
```

### Nested Functions

```javascript
const obj = {
  value: 42,
  method() {
    console.log(this.value); // 42 (implicit binding)
    
    function nested() {
      console.log(this.value); // undefined (default binding)
    }
    
    nested(); // Called standalone, 'this' is global/undefined
  }
};

obj.method();

// Solution 1: Save reference to 'this'
const obj2 = {
  value: 42,
  method() {
    const self = this; // Capture 'this'
    
    function nested() {
      console.log(self.value); // 42 (closure over 'self')
    }
    
    nested();
  }
};

// Solution 2: Use arrow function (covered later)
const obj3 = {
  value: 42,
  method() {
    const nested = () => {
      console.log(this.value); // 42 (lexical 'this')
    };
    
    nested();
  }
};
```

## Implicit Binding

Implicit binding occurs when a function is called as a method of an object.

### Basic Implicit Binding

```javascript
const person = {
  name: 'Alice',
  greet() {
    console.log(`Hello, I'm ${this.name}`);
  }
};

person.greet(); // 'Hello, I'm Alice'
// 'this' refers to 'person' (object to the left of dot)
```

### Multiple Levels

```javascript
const obj = {
  a: {
    b: {
      c() {
        console.log(this);
      }
    }
  }
};

obj.a.b.c(); // 'this' is obj.a.b (immediate parent object)
// Only the last level matters: obj.a.b.c()
//                                    ^^^^^ call site
```

### Implicit Loss in Callbacks

```javascript
const user = {
  name: 'Bob',
  greet() {
    console.log(`Hello, ${this.name}`);
  }
};

// Direct call - works
user.greet(); // 'Hello, Bob'

// Callback - loses binding!
setTimeout(user.greet, 1000); // 'Hello, undefined'
// Why? setTimeout calls the function directly: user.greet()
// No object context, reverts to default binding

// Solution 1: Wrapper function
setTimeout(() => user.greet(), 1000); // 'Hello, Bob'

// Solution 2: bind (covered later)
setTimeout(user.greet.bind(user), 1000); // 'Hello, Bob'
```

### Array Methods and this

```javascript
const processor = {
  prefix: '>>',
  process(items) {
    return items.map(function(item) {
      return this.prefix + item; // 'this' is undefined!
    });
  }
};

processor.process(['a', 'b']); // ['undefined a', 'undefined b']

// Solution 1: thisArg parameter
const processor2 = {
  prefix: '>>',
  process(items) {
    return items.map(function(item) {
      return this.prefix + item;
    }, this); // Pass 'this' as thisArg
  }
};

processor2.process(['a', 'b']); // ['>>a', '>>b']

// Solution 2: Arrow function
const processor3 = {
  prefix: '>>',
  process(items) {
    return items.map(item => this.prefix + item); // Lexical 'this'
  }
};

processor3.process(['a', 'b']); // ['>>a', '>>b']
```

## Explicit Binding

Explicit binding uses `call`, `apply`, or `bind` to manually set `this`.

### call() Method

```javascript
function greet(greeting, punctuation) {
  console.log(`${greeting}, ${this.name}${punctuation}`);
}

const person = { name: 'Alice' };

greet.call(person, 'Hello', '!'); // 'Hello, Alice!'
// Syntax: func.call(thisArg, arg1, arg2, ...)
```

### apply() Method

```javascript
function greet(greeting, punctuation) {
  console.log(`${greeting}, ${this.name}${punctuation}`);
}

const person = { name: 'Bob' };

greet.apply(person, ['Hello', '!']); // 'Hello, Bob!'
// Syntax: func.apply(thisArg, [argsArray])

// Useful with spread
const args = ['Hello', '!'];
greet.apply(person, args);
```

### bind() Method

```javascript
function greet(greeting) {
  console.log(`${greeting}, ${this.name}`);
}

const person = { name: 'Charlie' };

// bind() returns a new function with 'this' permanently bound
const boundGreet = greet.bind(person);

boundGreet('Hello'); // 'Hello, Charlie'
boundGreet('Hi');    // 'Hi, Charlie'

// Can't override bound 'this'
const other = { name: 'Diana' };
boundGreet.call(other, 'Hey'); // 'Hey, Charlie' (not Diana!)
```

### Partial Application with bind

```javascript
function multiply(a, b) {
  return a * b;
}

// Bind without 'this' context (null), but preset first argument
const double = multiply.bind(null, 2);
const triple = multiply.bind(null, 3);

console.log(double(5)); // 10 (2 * 5)
console.log(triple(5)); // 15 (3 * 5)

// Bind with both 'this' and arguments
const calculator = {
  factor: 10,
  multiply(a, b) {
    return this.factor * a * b;
  }
};

const bound = calculator.multiply.bind(calculator, 2); // Preset 'this' and 'a'
console.log(bound(5)); // 100 (10 * 2 * 5)
```

### Event Handlers with bind

```javascript
class Button {
  constructor(label) {
    this.label = label;
    this.clickCount = 0;
  }
  
  handleClick() {
    this.clickCount++;
    console.log(`${this.label} clicked ${this.clickCount} times`);
  }
  
  render() {
    const button = document.createElement('button');
    button.textContent = this.label;
    
    // Must bind to preserve 'this'
    button.addEventListener('click', this.handleClick.bind(this));
    
    return button;
  }
}

const btn = new Button('Submit');
document.body.appendChild(btn.render());
```

## new Binding

When a function is called with `new`, `this` is bound to the newly created object.

### Basic new Binding

```javascript
function Person(name, age) {
  this.name = name;
  this.age = age;
  console.log(this); // Person { name: '...', age: ... }
}

const alice = new Person('Alice', 30);
console.log(alice.name); // 'Alice'

// What 'new' does:
// 1. Create new empty object: {}
// 2. Set its [[Prototype]]: Object.setPrototypeOf(obj, Person.prototype)
// 3. Bind 'this' to new object
// 4. Execute function body
// 5. Return object (unless function explicitly returns an object)
```

### Manual Implementation of new

```javascript
function myNew(constructor, ...args) {
  // 1. Create new object with constructor's prototype
  const obj = Object.create(constructor.prototype);
  
  // 2. Bind 'this' and execute constructor
  const result = constructor.apply(obj, args);
  
  // 3. Return object or constructor's return value
  return (typeof result === 'object' && result !== null) ? result : obj;
}

function Person(name) {
  this.name = name;
}

const person = myNew(Person, 'Alice');
console.log(person.name); // 'Alice'
console.log(person instanceof Person); // true
```

### Returning Objects from Constructor

```javascript
function Person(name) {
  this.name = name;
  
  // Explicit object return overrides 'this'
  return { name: 'Overridden' };
}

const person = new Person('Alice');
console.log(person.name); // 'Overridden' (returned object used)

// Returning primitive has no effect
function Person2(name) {
  this.name = name;
  return 42; // Ignored
}

const person2 = new Person2('Bob');
console.log(person2.name); // 'Bob' (this returned as usual)
```

### new with Arrow Functions

```javascript
// Arrow functions cannot be used as constructors
const Person = (name) => {
  this.name = name;
};

// new Person('Alice'); // TypeError: Person is not a constructor

// Arrow functions don't have their own 'this' or prototype
console.log(Person.prototype); // undefined
```

## Arrow Functions

Arrow functions don't have their own `this` binding. They lexically capture `this` from their enclosing scope.

### Lexical this

```javascript
const obj = {
  value: 42,
  regularMethod() {
    console.log(this.value); // 42 (implicit binding)
    
    setTimeout(function() {
      console.log(this.value); // undefined (default binding)
    }, 100);
  },
  
  arrowMethod() {
    console.log(this.value); // 42 (implicit binding)
    
    setTimeout(() => {
      console.log(this.value); // 42 (lexical 'this' from arrowMethod)
    }, 100);
  }
};

obj.regularMethod();
obj.arrowMethod();
```

### Arrow Functions Can't Be Rebound

```javascript
const obj = { value: 42 };

const arrow = () => {
  console.log(this.value);
};

// call, apply, bind have no effect on arrow functions
arrow.call(obj);       // undefined (ignores explicit binding)
arrow.apply(obj);      // undefined
const bound = arrow.bind(obj);
bound();               // undefined

// Arrow function's 'this' is determined at creation, not invocation
```

### Arrow Functions in Object Methods

```javascript
// PROBLEM: Arrow function in object literal
const obj = {
  value: 42,
  // 'this' here is lexical (global scope, not obj)
  method: () => {
    console.log(this.value); // undefined (not obj.value)
  }
};

obj.method(); // undefined

// Why? Object literals don't create new scope
// Arrow function captures 'this' from surrounding scope (global)

// SOLUTION: Use regular function
const obj2 = {
  value: 42,
  method() {
    console.log(this.value); // 42 (implicit binding)
  }
};

obj2.method(); // 42
```

### Arrow Functions in Classes

```javascript
class Counter {
  constructor() {
    this.count = 0;
  }
  
  // Regular method
  increment() {
    this.count++;
  }
  
  // Arrow function as class field (proposal)
  incrementArrow = () => {
    this.count++;
  }
}

const counter = new Counter();

// Regular method loses context
const inc = counter.increment;
// inc(); // TypeError: Cannot read property 'count' of undefined

// Arrow function retains context
const incArrow = counter.incrementArrow;
incArrow(); // Works! 'this' is lexically bound to instance
console.log(counter.count); // 1
```

### When to Use Arrow Functions

```javascript
// GOOD: Callbacks, preserving outer 'this'
class DataFetcher {
  constructor() {
    this.data = [];
  }
  
  fetch() {
    fetch('/api/data')
      .then(response => response.json())
      .then(data => {
        this.data = data; // 'this' is DataFetcher instance
      });
  }
}

// GOOD: Array methods
const processor = {
  prefix: '>>',
  process(items) {
    return items.map(item => `${this.prefix}${item}`);
  }
};

// BAD: Object methods that need dynamic 'this'
const calculator = {
  value: 10,
  add: (x) => this.value + x // Wrong! 'this' is not calculator
};

// BAD: Prototype methods
Person.prototype.greet = () => {
  console.log(this.name); // Wrong! 'this' is not instance
};
```

## Binding Precedence

When multiple binding rules apply, they have a specific precedence order.

### Precedence Order

```
1. new binding (highest)
2. Explicit binding (call, apply, bind)
3. Implicit binding (method call)
4. Default binding (lowest)

Arrow functions ignore all rules and use lexical scope
```

### new vs Implicit

```javascript
function Person(name) {
  this.name = name;
}

const obj = {
  createPerson: Person
};

// Implicit binding alone
obj.createPerson('Alice'); // 'this' is obj (name added to obj)

// new overrides implicit
const person = new obj.createPerson('Bob'); // 'this' is new object
console.log(person.name); // 'Bob'
console.log(obj.name); // undefined (new took precedence)
```

### Explicit vs Implicit

```javascript
function greet() {
  console.log(this.name);
}

const obj1 = { name: 'Alice', greet };
const obj2 = { name: 'Bob' };

obj1.greet(); // 'Alice' (implicit)
obj1.greet.call(obj2); // 'Bob' (explicit overrides implicit)
```

### new vs Explicit (bind)

```javascript
function Person(name) {
  this.name = name;
}

const obj = { name: 'Object' };

// bind creates hard binding
const BoundPerson = Person.bind(obj);

// new overrides bind
const person = new BoundPerson('Alice');
console.log(person.name); // 'Alice' (new wins)
console.log(obj.name); // 'Object' (unchanged)

// Why? 'new' creates new object, bind's 'this' is ignored
```

### Complete Precedence Example

```javascript
function show() {
  console.log(this.value);
}

const obj1 = { value: 1, show };
const obj2 = { value: 2 };

// Default
show(); // undefined

// Implicit
obj1.show(); // 1

// Explicit
obj1.show.call(obj2); // 2 (explicit > implicit)

// Bind
const bound = show.bind(obj2);
obj1.bound = bound;
obj1.bound(); // 2 (bind > implicit)

// new
function Value(value) {
  this.value = value;
}
const boundValue = Value.bind(obj1);
const instance = new boundValue(3);
console.log(instance.value); // 3 (new > bind)
```

## call, apply, bind Internals

Understanding how these methods work internally helps understand `this` binding.

### call Implementation

```javascript
// Simplified call() polyfill
Function.prototype.myCall = function(thisArg, ...args) {
  // 1. Handle null/undefined (default binding)
  const context = thisArg ?? globalThis;
  
  // 2. Create unique property on context
  const uniqueKey = Symbol('fn');
  
  // 3. Assign function to context
  context[uniqueKey] = this;
  
  // 4. Call function as method (implicit binding)
  const result = context[uniqueKey](...args);
  
  // 5. Clean up
  delete context[uniqueKey];
  
  // 6. Return result
  return result;
};

function greet(greeting) {
  return `${greeting}, ${this.name}`;
}

const person = { name: 'Alice' };
console.log(greet.myCall(person, 'Hello')); // 'Hello, Alice'
```

### apply Implementation

```javascript
// Simplified apply() polyfill
Function.prototype.myApply = function(thisArg, argsArray) {
  const context = thisArg ?? globalThis;
  const uniqueKey = Symbol('fn');
  
  context[uniqueKey] = this;
  
  // Spread array into arguments
  const result = argsArray 
    ? context[uniqueKey](...argsArray)
    : context[uniqueKey]();
  
  delete context[uniqueKey];
  return result;
};

function sum(a, b, c) {
  return this.multiplier * (a + b + c);
}

const obj = { multiplier: 2 };
console.log(sum.myApply(obj, [1, 2, 3])); // 12 (2 * 6)
```

### bind Implementation

```javascript
// Simplified bind() polyfill
Function.prototype.myBind = function(thisArg, ...boundArgs) {
  const originalFunc = this;
  
  // Return new function that:
  return function(...args) {
    // 1. Combines bound args with call-time args
    const allArgs = [...boundArgs, ...args];
    
    // 2. Check if called with 'new'
    if (new.target) {
      // Called with new: ignore thisArg, use new object
      return new originalFunc(...allArgs);
    }
    
    // 3. Normal call: use bound thisArg
    return originalFunc.apply(thisArg, allArgs);
  };
};

function greet(greeting, punctuation) {
  return `${greeting}, ${this.name}${punctuation}`;
}

const person = { name: 'Bob' };
const boundGreet = greet.myBind(person, 'Hello');
console.log(boundGreet('!')); // 'Hello, Bob!'
```

### Performance Comparison

```javascript
const obj = { value: 42 };

function test() {
  return this.value;
}

// Direct call (fastest)
obj.test = test;
obj.test(); // Implicit binding, very fast

// call/apply (slower, function call overhead)
test.call(obj); // Explicit binding, slower
test.apply(obj); // Similar to call

// bind (creates new function, overhead on creation)
const bound = test.bind(obj); // One-time cost
bound(); // Subsequent calls relatively fast

// Arrow function (lexical, compiled away)
const arrow = () => obj.value; // Fast, no 'this' lookup
arrow();
```

## Common Misconceptions

### Misconception 1: "this Refers to the Function Itself"

**Reality**: `this` doesn't refer to the function; it refers to the context in which the function is called.

```javascript
function count() {
  this.counter++; // 'this' is NOT the function
}

count.counter = 0;
count(); // 'this' is global/undefined, not count function
console.log(count.counter); // 0 (unchanged)

// To reference function itself, use name or arguments.callee (deprecated)
function count2() {
  count2.counter++;
}
count2.counter = 0;
count2();
console.log(count2.counter); // 1
```

### Misconception 2: "this Is Determined at Write-Time"

**Reality**: `this` (except in arrow functions) is determined at call-time, not write-time.

```javascript
const greet = function() {
  console.log(this.name);
};

const obj1 = { name: 'Alice', greet };
const obj2 = { name: 'Bob', greet };

obj1.greet(); // 'Alice' (call-time binding)
obj2.greet(); // 'Bob' (different call-time binding)

// Same function, different 'this' based on how it's called
```

### Misconception 3: "Arrow Functions Can Be Used Anywhere"

**Reality**: Arrow functions are not suitable for methods that need dynamic `this`.

```javascript
const obj = {
  value: 42,
  method: () => {
    console.log(this.value); // undefined, 'this' is lexical (global)
  }
};

obj.method(); // undefined (not obj.value)
```

### Misconception 4: "bind Can Be Overridden"

**Reality**: Once a function is bound, its `this` binding is permanent.

```javascript
function show() {
  console.log(this.value);
}

const obj1 = { value: 1 };
const obj2 = { value: 2 };

const bound = show.bind(obj1);
bound(); // 1

// Cannot rebind
bound.call(obj2); // 1 (not 2, bind cannot be overridden)
bound.apply(obj2); // 1
bound.bind(obj2)(); // 1
```

## Performance Implications

### Binding Method Overhead

```javascript
// Fastest: Direct method call
const obj = {
  value: 42,
  method() {
    return this.value;
  }
};
obj.method(); // Fastest

// Slower: call/apply (function call overhead)
function getValue() {
  return this.value;
}
getValue.call(obj); // Slower

// bind overhead on creation, then fast
const bound = getValue.bind(obj); // One-time cost
bound(); // Subsequent calls fast
```

### Event Handler Binding

```javascript
class Component {
  constructor() {
    this.count = 0;
    
    // GOOD: Bind once in constructor
    this.handleClick = this.handleClick.bind(this);
  }
  
  handleClick() {
    this.count++;
  }
  
  render() {
    // Reuses same bound function
    return <button onClick={this.handleClick} />;
  }
}

// BAD: Binding in render (creates new function each render)
class BadComponent {
  handleClick() {
    this.count++;
  }
  
  render() {
    // New bound function every render!
    return <button onClick={this.handleClick.bind(this)} />;
  }
}
```

### Arrow Function Memory

```javascript
class Counter {
  constructor() {
    this.count = 0;
  }
  
  // Regular method: Shared on prototype
  increment() {
    this.count++;
  }
  
  // Arrow function: Per-instance
  incrementArrow = () => {
    this.count++;
  }
}

// Memory comparison
const c1 = new Counter();
const c2 = new Counter();

// increment is shared (same function)
console.log(c1.increment === c2.increment); // true

// incrementArrow is per-instance (different functions)
console.log(c1.incrementArrow === c2.incrementArrow); // false

// Trade-off: Arrow functions convenient but use more memory
```

## Interview Questions

### Question 1: What are the four rules for this binding?

**Answer**: 
1. **Default binding**: Standalone function call, `this` is global/undefined (strict mode)
2. **Implicit binding**: Method call, `this` is the object (left of dot)
3. **Explicit binding**: `call`/`apply`/`bind`, `this` is specified argument
4. **new binding**: Constructor call, `this` is newly created object

Arrow functions don't follow these rules; they use lexical `this`.

### Question 2: What's the difference between call, apply, and bind?

**Answer**:
- `call(thisArg, arg1, arg2, ...)`: Invokes function immediately with specified `this` and individual arguments
- `apply(thisArg, [argsArray])`: Same as `call` but takes arguments as array
- `bind(thisArg, arg1, ...)`: Returns new function with permanently bound `this` (doesn't invoke immediately)

`call` and `apply` are similar (differ only in argument format), while `bind` creates a new function.

### Question 3: How do arrow functions handle this?

**Answer**: Arrow functions don't have their own `this` binding. They lexically capture `this` from the enclosing scope at creation time. This makes them immune to `call`, `apply`, and `bind`. They're ideal for callbacks and avoiding `this` binding issues, but shouldn't be used for object methods that need dynamic `this`.

### Question 4: What's the precedence order of binding rules?

**Answer**: 
1. `new` binding (highest)
2. Explicit binding (`call`/`apply`/`bind`)
3. Implicit binding (method call)
4. Default binding (lowest)

Arrow functions bypass all rules and use lexical scope.

### Question 5: Why do arrow functions in object literals not bind to the object?

**Answer**: Object literals don't create a new scope. When you define an arrow function in an object literal, it captures `this` from the surrounding scope (usually global), not from the object. Arrow functions determine `this` lexically at creation time, and the object hasn't been created yet when the arrow function is defined.

## Key Takeaways

1. **this is call-site determined**: How a function is called determines `this` (except arrow functions)

2. **Four binding rules**: Default, implicit, explicit, new (in precedence order)

3. **Arrow functions are lexical**: They capture `this` from enclosing scope, ignoring binding rules

4. **Lost context is common**: Callbacks often lose `this`; use arrow functions or bind

5. **bind creates permanent binding**: Once bound, `this` cannot be changed

6. **call vs apply**: Only difference is argument format (individual vs array)

7. **new binding is powerful**: Creates new object and binds `this` to it

8. **Object methods need regular functions**: Arrow functions in object literals don't bind to object

9. **Strict mode affects default binding**: `this` is `undefined` instead of global

10. **Performance trade-offs**: Direct calls fastest, bind has creation cost, arrow functions use more memory

## Resources

### Official Documentation
- [MDN: this](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this)
- [MDN: Function.prototype.call()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/call)
- [MDN: Arrow functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions)

### Articles
- "Gentle Explanation of 'this' in JavaScript" by Dmitri Pavlutin
- "Understanding JavaScript's 'this' keyword" by Tyler McGinnis
- "You Don't Know JS: this & Object Prototypes" by Kyle Simpson

### Books
- "You Don't Know JS: this & Object Prototypes" by Kyle Simpson
- "JavaScript: The Definitive Guide" by David Flanagan
- "Eloquent JavaScript" by Marijn Haverbeke

### Videos
- "JavaScript this Keyword" by Web Dev Simplified
- "Understanding 'this' in JavaScript" by Fun Fun Function
- "Arrow Functions and this" by Traversy Media
