# Function Types: Declarations, Expressions, and Arrow Functions

## The Idea

**In plain English:** A function is a reusable set of instructions you give to the computer — you write it once, then call it by name whenever you need it to run. JavaScript gives you three different ways to write those instructions, and each way has slightly different rules about when it can be used and what it "knows" about the code around it.

**Real-world analogy:** Think of a barista at a coffee shop who follows recipe cards. There are three ways to set up those recipe cards: pinned to the board before the shift starts, written on a sticky note during the shift, or typed into a shortcut button on the espresso machine.

- The recipe card pinned before the shift = a function declaration (available from the very start, even before you reach that point in the code)
- The sticky note written mid-shift = a function expression (only usable after the moment it was written down)
- The shortcut button on the machine = an arrow function (compact, quick to set up, and automatically "knows" whose machine it belongs to)

---

## Overview

JavaScript provides multiple ways to define functions, each with distinct characteristics regarding hoisting, syntax, the `this` binding, and use cases. Understanding these differences is crucial for writing effective JavaScript code and avoiding common pitfalls. The three primary function types are function declarations, function expressions, and arrow functions (introduced in ES6).

## Function Declarations

### Syntax

```javascript
function functionName(parameters) {
  // Function body
  return value;
}

// Example
function greet(name) {
  return `Hello, ${name}!`;
}

console.log(greet("Alice")); // "Hello, Alice!"
```

### Characteristics

#### 1. Hoisting

Function declarations are hoisted to the top of their scope:

```javascript
// Can call before declaration
console.log(add(2, 3)); // 5

function add(a, b) {
  return a + b;
}

// What JavaScript sees (conceptually):
// function add(a, b) { return a + b; }
// console.log(add(2, 3));
```

Hoisting includes the entire function:

```javascript
sayHello(); // "Hello!" - works!

function sayHello() {
  console.log("Hello!");
}

// Compare with variables
console.log(myVar); // undefined (only declaration hoisted)
var myVar = 5;
```

#### 2. Function Scope and Block Scope

In strict mode and modern JavaScript, function declarations in blocks are block-scoped:

```javascript
"use strict";

if (true) {
  function blockScoped() {
    return "inside";
  }
  console.log(blockScoped()); // "inside"
}

console.log(typeof blockScoped); // "undefined" (block-scoped)

// In non-strict mode (legacy behavior may vary)
```

#### 3. Named Functions

Function declarations create named functions, which aids debugging:

```javascript
function calculateTotal(items) {
  return items.reduce((sum, item) => sum + item.price, 0);
}

// Stack traces show "calculateTotal"
// Better debugging experience
```

### Use Cases

```javascript
// Top-level utility functions
function validateEmail(email) {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}

// Public API functions
function processData(data) {
  return transform(filter(validate(data)));
}

// Recursive functions (name is useful)
function factorial(n) {
  if (n <= 1) return 1;
  return n * factorial(n - 1);
}
```

## Function Expressions

### Syntax

```javascript
const functionName = function(parameters) {
  // Function body
  return value;
};

// Example
const greet = function(name) {
  return `Hello, ${name}!`;
};

console.log(greet("Bob")); // "Hello, Bob!"
```

### Characteristics

#### 1. No Hoisting

Function expressions are not hoisted:

```javascript
// Error: Cannot access before initialization
console.log(add(2, 3)); // ReferenceError

const add = function(a, b) {
  return a + b;
};

// With var: undefined, not a function
console.log(multiply(2, 3)); // TypeError: multiply is not a function
var multiply = function(a, b) {
  return a * b;
};
```

#### 2. Anonymous vs Named

Function expressions can be anonymous or named:

```javascript
// Anonymous function expression
const greet = function(name) {
  return `Hello, ${name}!`;
};

// Named function expression
const greet2 = function greetFunction(name) {
  return `Hello, ${name}!`;
};

// The name is only available inside the function
const factorial = function fact(n) {
  if (n <= 1) return 1;
  return n * fact(n - 1); // Can use 'fact' internally
};

console.log(factorial(5)); // 120
console.log(typeof fact);  // "undefined" - name not exposed
```

Benefits of named function expressions:

```javascript
// Better stack traces
const calculate = function calculateTotal(items) {
  // Error stack shows "calculateTotal" instead of "anonymous"
  return items.map(item => item.price * item.quantity);
};

// Self-reference without relying on variable name
const countdown = function count(n) {
  if (n <= 0) return;
  console.log(n);
  setTimeout(() => count(n - 1), 1000);
};
```

#### 3. First-Class Citizens

Functions can be assigned, passed, and returned:

```javascript
// Assign to variable
const operation = function(a, b) {
  return a + b;
};

// Pass as argument
function execute(func, x, y) {
  return func(x, y);
}

// Return from function
function createMultiplier(factor) {
  return function(number) {
    return number * factor;
  };
}

const double = createMultiplier(2);
console.log(double(5)); // 10
```

### Use Cases

```javascript
// Conditional function assignment
const logger = isProduction
  ? function() { /* no-op */ }
  : function(msg) { console.log(msg); };

// Callbacks
setTimeout(function() {
  console.log("Delayed message");
}, 1000);

// Array methods
const doubled = numbers.map(function(n) {
  return n * 2;
});

// Immediately Invoked Function Expression (IIFE)
(function() {
  const private = "Not accessible outside";
  console.log("IIFE executed");
})();

// Method definitions
const obj = {
  method: function() {
    return "method called";
  }
};
```

## Arrow Functions

### Syntax

```javascript
// Basic syntax
const functionName = (parameters) => {
  // Function body
  return value;
};

// Single parameter (parentheses optional)
const greet = name => {
  return `Hello, ${name}!`;
};

// Implicit return (single expression)
const greet2 = name => `Hello, ${name}!`;

// No parameters
const sayHello = () => "Hello!";

// Multiple parameters
const add = (a, b) => a + b;

// Returning object literal (wrap in parentheses)
const createUser = (name, age) => ({ name, age });
```

### Characteristics

#### 1. Lexical this Binding

Arrow functions don't have their own `this` - they inherit from parent scope:

```javascript
// Traditional function (own 'this')
const obj = {
  value: 42,
  traditional: function() {
    setTimeout(function() {
      console.log(this.value); // undefined ('this' is global/undefined)
    }, 100);
  }
};

// Arrow function (lexical 'this')
const obj2 = {
  value: 42,
  arrow: function() {
    setTimeout(() => {
      console.log(this.value); // 42 ('this' from obj2.arrow)
    }, 100);
  }
};

// Real-world example: Event handlers in classes
class Counter {
  constructor() {
    this.count = 0;
  }
  
  // Arrow function as class field (lexical this)
  increment = () => {
    this.count++;
    console.log(this.count);
  }
  
  // Traditional method (dynamic this)
  traditionalIncrement() {
    this.count++;
  }
}

const counter = new Counter();
button.addEventListener('click', counter.increment); // Works!
button.addEventListener('click', counter.traditionalIncrement); // 'this' is button!
```

#### 2. No arguments Object

Arrow functions don't have `arguments`:

```javascript
// Traditional function
function traditional() {
  console.log(arguments); // [1, 2, 3]
}
traditional(1, 2, 3);

// Arrow function
const arrow = () => {
  console.log(arguments); // ReferenceError or outer scope's arguments
};

// Use rest parameters instead
const arrow2 = (...args) => {
  console.log(args); // [1, 2, 3]
};
arrow2(1, 2, 3);
```

#### 3. Cannot Be Used as Constructors

Arrow functions can't be called with `new`:

```javascript
const ArrowFunc = () => {};
new ArrowFunc(); // TypeError: ArrowFunc is not a constructor

// Traditional functions can be constructors
function TraditionalFunc() {
  this.value = 42;
}
new TraditionalFunc(); // Works
```

#### 4. No prototype Property

```javascript
const arrow = () => {};
console.log(arrow.prototype); // undefined

function traditional() {}
console.log(traditional.prototype); // {} (object)
```

#### 5. Cannot Change this with call/apply/bind

```javascript
const obj = { value: 42 };

const arrow = () => {
  console.log(this.value);
};

const traditional = function() {
  console.log(this.value);
};

// Arrow function ignores 'this' binding
arrow.call(obj); // undefined (or global this)

// Traditional function respects binding
traditional.call(obj); // 42
```

### Implicit Return

```javascript
// Explicit return (requires braces and return keyword)
const add = (a, b) => {
  return a + b;
};

// Implicit return (single expression, no braces)
const add2 = (a, b) => a + b;

// Multiline implicit return (wrap in parentheses)
const createUser = (name, age) => (
  {
    name,
    age,
    createdAt: new Date()
  }
);

// Object literal requires parentheses
const getObject = () => ({ key: 'value' }); // Object
const getBlock = () => { key: 'value' };    // Undefined (block, not object!)
```

### Use Cases

```javascript
// Array methods (concise callbacks)
const doubled = numbers.map(n => n * 2);
const evens = numbers.filter(n => n % 2 === 0);
const sum = numbers.reduce((acc, n) => acc + n, 0);

// Event handlers needing 'this' context
class Component {
  constructor() {
    this.state = { count: 0 };
  }
  
  handleClick = () => {
    this.setState({ count: this.state.count + 1 });
  }
}

// Short callbacks
setTimeout(() => console.log('Hello'), 1000);
promise.then(data => processData(data));

// Functional programming
const compose = (...fns) => x => fns.reduceRight((acc, fn) => fn(acc), x);
const pipe = (...fns) => x => fns.reduce((acc, fn) => fn(acc), x);
```

## Comparison Table

| Feature | Declaration | Expression | Arrow |
|---------|-------------|------------|-------|
| Hoisting | Yes (entire function) | No (variable hoisted) | No |
| `this` | Dynamic | Dynamic | Lexical |
| `arguments` | Yes | Yes | No (use rest) |
| Constructor | Yes | Yes | No |
| Method syntax | No | Yes | Yes |
| Implicit return | No | No | Yes |
| Name | Always named | Can be anonymous | Always anonymous |
| Best for | Top-level functions | Conditional, callbacks | Short callbacks, lexical this |

## When to Use Each

### Use Function Declarations When:

```javascript
// 1. Top-level utility functions
function validateEmail(email) {
  return /\S+@\S+\.\S+/.test(email);
}

// 2. Functions that need hoisting
bootstrap();

function bootstrap() {
  initialize();
  loadData();
}

// 3. Recursive functions (name helps)
function traverse(node) {
  if (!node) return;
  process(node);
  traverse(node.left);
  traverse(node.right);
}
```

### Use Function Expressions When:

```javascript
// 1. Conditional function assignment
const logger = isDevelopment
  ? function(msg) { console.log(msg); }
  : function() {};

// 2. IIFEs
(function() {
  // Private scope
  const secret = "hidden";
})();

// 3. Creating closures
function createCounter() {
  let count = 0;
  return function() {
    return ++count;
  };
}

// 4. Before arrow functions existed (legacy code)
const handler = function(event) {
  // ...
};
```

### Use Arrow Functions When:

```javascript
// 1. Array methods
const filtered = items.filter(item => item.active);
const mapped = items.map(item => item.value);

// 2. Short callbacks
setTimeout(() => console.log('done'), 1000);
promise.then(data => process(data));

// 3. Need lexical 'this'
class Component {
  handleEvent = () => {
    this.setState({ updated: true });
  }
}

// 4. Functional composition
const add5 = x => x + 5;
const multiply2 = x => x * 2;
const result = multiply2(add5(10)); // 30

// 5. One-liner utilities
const double = x => x * 2;
const isEven = x => x % 2 === 0;
const getKey = obj => obj.key;
```

## Common Patterns

### Method Definitions

```javascript
// ES5: Function expressions
const obj = {
  method: function() {
    return "traditional";
  }
};

// ES6: Method shorthand (preferred for methods)
const obj2 = {
  method() {
    return "shorthand";
  }
};

// Arrow functions (DON'T use for methods needing 'this')
const obj3 = {
  value: 42,
  // Bad: 'this' doesn't refer to obj3
  method: () => {
    return this.value; // undefined or outer scope's 'this'
  },
  // Good: Use method shorthand
  method2() {
    return this.value; // 42
  }
};
```

### Constructor Functions

```javascript
// Function declaration/expression (ES5 style)
function Person(name, age) {
  this.name = name;
  this.age = age;
}

Person.prototype.greet = function() {
  return `Hello, I'm ${this.name}`;
};

// Class syntax (ES6, preferred)
class Person {
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }
  
  greet() {
    return `Hello, I'm ${this.name}`;
  }
}

// Arrow functions CAN'T be constructors
const Person = (name) => {
  this.name = name; // Error if called with 'new'
};
```

### Callback Functions

```javascript
// Verbose traditional
button.addEventListener('click', function(event) {
  console.log('Clicked:', event.target);
});

// Concise arrow
button.addEventListener('click', event => {
  console.log('Clicked:', event.target);
});

// Very concise (implicit return for testing)
const isValid = items.some(item => item.status === 'active');
```

## Best Practices

### 1. Prefer const for Function Expressions and Arrows

```javascript
// Good: Prevents reassignment
const add = (a, b) => a + b;

// Bad: Allows accidental reassignment
let add = (a, b) => a + b;
add = "oops"; // No error, but breaks code
```

### 2. Use Arrow Functions for Short Callbacks

```javascript
// Good: Concise and clear
const doubled = nums.map(n => n * 2);

// Overkill: Too verbose
const doubled = nums.map(function(n) {
  return n * 2;
});
```

### 3. Use Function Declarations for Top-Level Functions

```javascript
// Good: Hoisted, clear intent
function processData(data) {
  // ...
}

// Less ideal: Not hoisted, extra syntax
const processData = (data) => {
  // ...
};
```

### 4. Don't Use Arrow Functions as Methods

```javascript
// Bad: 'this' is lexical (outer scope)
const obj = {
  value: 42,
  getValue: () => this.value // Wrong 'this'!
};

// Good: Use method shorthand
const obj = {
  value: 42,
  getValue() {
    return this.value; // Correct 'this'
  }
};
```

### 5. Name Your Functions (Even Expressions)

```javascript
// Good: Named for debugging
const handler = function handleClick(event) {
  // Stack traces show "handleClick"
};

// Less ideal: Anonymous in stack traces
const handler = function(event) {
  // Shows as "anonymous"
};
```

## Common Pitfalls

### 1. Arrow Functions as Methods

```javascript
const user = {
  name: "Alice",
  greet: () => {
    console.log(`Hello, ${this.name}`); // 'this' is not 'user'!
  }
};

user.greet(); // "Hello, undefined"
```

### 2. Forgotten Return in Arrow Functions

```javascript
// Forgot parentheses around object
const createUser = id => { id: id }; // Returns undefined! (block, not object)

// Correct: Wrap object in parentheses
const createUser = id => ({ id: id });

// Or use explicit return
const createUser = id => {
  return { id: id };
};
```

### 3. Using arguments in Arrow Functions

```javascript
const sum = () => {
  return arguments.reduce((a, b) => a + b, 0); // ReferenceError!
};

// Correct: Use rest parameters
const sum = (...numbers) => {
  return numbers.reduce((a, b) => a + b, 0);
};
```

## Resources

### Documentation
- [MDN: Functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Functions)
- [MDN: Arrow Functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions)
- [MDN: Function Declaration](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function)

### Articles
- [JavaScript Functions Explained](https://javascript.info/function-basics)
- [Arrow Functions in Depth](https://javascript.info/arrow-functions)
