# var, let, and const: Variable Declarations in JavaScript

## Overview

JavaScript provides three ways to declare variables: `var`, `let`, and `const`. While `var` is the legacy declaration from ES5 and earlier, `let` and `const` were introduced in ES6 (2015) to address fundamental issues with `var`'s scoping behavior. Understanding the differences between these keywords is crucial for writing predictable, maintainable JavaScript code.

## var: The Legacy Declaration

### Function Scope

Unlike most programming languages where variables are block-scoped, `var` declarations are function-scoped. This means they are only constrained by function boundaries, not block boundaries like `if`, `for`, or `while` statements.

```javascript
function demonstrateVarScope() {
  if (true) {
    var message = "Hello";
  }
  console.log(message); // "Hello" - accessible outside the block!
}

// Compare with C-style languages where this would be a compilation error
```

### Hoisting Behavior

Variables declared with `var` are "hoisted" to the top of their function scope. This means the declaration is processed before any code execution, but the initialization remains in place.

```javascript
function hoistingExample() {
  console.log(x); // undefined (not ReferenceError!)
  var x = 5;
  console.log(x); // 5
}

// What actually happens (conceptually):
function hoistingExample() {
  var x; // Declaration hoisted
  console.log(x); // undefined
  x = 5; // Assignment stays in place
  console.log(x); // 5
}
```

### Re-declaration Allowed

`var` allows you to re-declare the same variable multiple times in the same scope without error:

```javascript
var count = 1;
var count = 2; // No error - silently overwrites
console.log(count); // 2

// This can lead to subtle bugs when accidentally reusing variable names
function processData() {
  var result = "initial";
  
  // ... 100 lines of code ...
  
  var result = "new value"; // Accidentally redeclared - no warning!
}
```

### Global Object Property

In the global scope, `var` declarations create properties on the global object (`window` in browsers):

```javascript
var globalVar = "I'm global";
console.log(window.globalVar); // "I'm global" (in browser)

// This can pollute the global namespace and cause conflicts
```

### Loop Variable Gotcha

One of the most notorious `var` issues involves closures in loops:

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(function() {
    console.log(i); // What does this print?
  }, 100);
}
// Prints: 3, 3, 3 (not 0, 1, 2!)

// Why? Because there's only ONE 'i' variable shared across all iterations
// By the time the timeouts execute, the loop has finished and i is 3
```

## let: Block-Scoped Declaration

### Block Scope

`let` is block-scoped, meaning it's constrained by the nearest enclosing block (`{}` or statement):

```javascript
function demonstrateLetScope() {
  let x = 1;
  
  if (true) {
    let x = 2; // Different variable (shadowing)
    console.log(x); // 2
  }
  
  console.log(x); // 1 - outer x unchanged
}

// This is the expected behavior in most programming languages
```

### Temporal Dead Zone (TDZ)

Unlike `var`, accessing a `let` variable before its declaration results in a `ReferenceError`. The period between entering the scope and the actual declaration is called the Temporal Dead Zone.

```javascript
function temporalDeadZone() {
  // TDZ starts here
  console.log(x); // ReferenceError: Cannot access 'x' before initialization
  let x = 5; // TDZ ends here
}

// This makes bugs easier to catch during development
```

The TDZ applies even within the same statement:

```javascript
let x = x + 1; // ReferenceError: Cannot access 'x' before initialization

// The right-hand side is evaluated before the variable is initialized
```

### No Re-declaration

`let` prevents accidental re-declaration in the same scope:

```javascript
let count = 1;
let count = 2; // SyntaxError: Identifier 'count' has already been declared

// However, shadowing in nested scopes is allowed:
let value = "outer";
{
  let value = "inner"; // Different scope, OK
  console.log(value); // "inner"
}
console.log(value); // "outer"
```

### Loop Variables Done Right

`let` solves the closure-in-loop problem by creating a new binding for each iteration:

```javascript
for (let i = 0; i < 3; i++) {
  setTimeout(function() {
    console.log(i); // 0, 1, 2 (as expected!)
  }, 100);
}

// Each iteration gets its own 'i' variable
```

### No Global Object Property

Global `let` declarations don't create properties on the global object:

```javascript
let globalLet = "I'm global but isolated";
console.log(window.globalLet); // undefined

// This prevents namespace pollution
```

## const: Immutable Bindings

### Read-Only Binding

`const` creates a read-only binding to a value. The binding itself cannot be reassigned:

```javascript
const PI = 3.14159;
PI = 3; // TypeError: Assignment to constant variable

// Must be initialized at declaration
const x; // SyntaxError: Missing initializer in const declaration
```

### Object Mutability

`const` prevents reassignment, but NOT mutation of objects:

```javascript
const person = { name: "Alice", age: 30 };

// This is OK - mutating the object
person.age = 31;
person.city = "New York";
console.log(person); // { name: "Alice", age: 31, city: "New York" }

// This fails - reassigning the binding
person = { name: "Bob" }; // TypeError: Assignment to constant variable
```

This applies to arrays as well:

```javascript
const numbers = [1, 2, 3];
numbers.push(4); // OK - mutating the array
numbers[0] = 99; // OK - mutating elements
console.log(numbers); // [99, 2, 3, 4]

numbers = [5, 6, 7]; // TypeError: Assignment to constant variable
```

### Preventing Mutation

If you need true immutability, use `Object.freeze()`:

```javascript
const config = Object.freeze({
  apiUrl: "https://api.example.com",
  timeout: 5000
});

config.timeout = 10000; // Fails silently in non-strict mode
console.log(config.timeout); // 5000 (unchanged)

// In strict mode:
"use strict";
config.timeout = 10000; // TypeError: Cannot assign to read only property
```

Note that `Object.freeze()` is shallow:

```javascript
const user = Object.freeze({
  name: "Alice",
  address: { city: "Boston" }
});

user.name = "Bob"; // Fails (frozen)
user.address.city = "New York"; // Succeeds (nested object not frozen)

// For deep freezing, you need a recursive solution
function deepFreeze(obj) {
  Object.freeze(obj);
  Object.getOwnPropertyNames(obj).forEach(prop => {
    if (obj[prop] !== null
      && (typeof obj[prop] === "object" || typeof obj[prop] === "function")
      && !Object.isFrozen(obj[prop])) {
      deepFreeze(obj[prop]);
    }
  });
  return obj;
}
```

### Same Scoping as let

`const` follows all the same scoping rules as `let`:

```javascript
// Block-scoped
{
  const x = 1;
}
console.log(x); // ReferenceError

// Temporal Dead Zone
console.log(MAX); // ReferenceError
const MAX = 100;

// No re-declaration
const count = 1;
const count = 2; // SyntaxError
```

## Comparison Table

| Feature | var | let | const |
|---------|-----|-----|-------|
| Scope | Function | Block | Block |
| Hoisting | Yes (undefined) | Yes (TDZ) | Yes (TDZ) |
| Re-declaration | Allowed | Not allowed | Not allowed |
| Re-assignment | Allowed | Allowed | Not allowed |
| Global object property | Yes | No | No |
| Must initialize | No | No | Yes |
| TDZ | No | Yes | Yes |

## Best Practices

### 1. Default to const

Start with `const` by default. This makes your code more predictable and prevents accidental reassignments:

```javascript
// Good - clear that these values don't change
const MAX_RETRIES = 3;
const API_BASE_URL = "https://api.example.com";
const user = { name: "Alice", role: "admin" };
```

### 2. Use let When Reassignment is Needed

Use `let` only when you know the variable will be reassigned:

```javascript
// Good - accumulator that changes
let sum = 0;
for (const num of numbers) {
  sum += num;
}

// Good - loop counter
let i = 0;
while (i < items.length) {
  processItem(items[i]);
  i++;
}
```

### 3. Never Use var in Modern Code

There's no good reason to use `var` in ES6+ code. It only exists for backwards compatibility:

```javascript
// Bad
var count = 0;

// Good
let count = 0;
```

### 4. One Declaration Per Line

For clarity, declare variables on separate lines:

```javascript
// Bad
let x = 1, y = 2, z = 3;

// Good
let x = 1;
let y = 2;
let z = 3;

// Also good (when related)
const config = {
  timeout: 5000,
  retries: 3
};
```

### 5. Declare Variables at the Top of Their Scope

While block scoping allows declaration anywhere, declaring at the top improves readability:

```javascript
function processData(items) {
  // Declare all variables upfront
  const results = [];
  let errorCount = 0;
  
  for (const item of items) {
    // Use them here
    if (validateItem(item)) {
      results.push(transformItem(item));
    } else {
      errorCount++;
    }
  }
  
  return { results, errorCount };
}
```

## Interview Questions

### Q1: What will this code output?

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}

for (let j = 0; j < 3; j++) {
  setTimeout(() => console.log(j), 0);
}
```

**Answer:** The first loop outputs `3, 3, 3` because `var` creates a single function-scoped variable. The second outputs `0, 1, 2` because `let` creates a new block-scoped variable for each iteration.

### Q2: Is this code valid? If not, why?

```javascript
const arr = [1, 2, 3];
arr.push(4);
console.log(arr);
```

**Answer:** Yes, it's valid. `const` prevents reassignment of the binding, but doesn't make the object immutable. You can mutate the array contents, just not reassign `arr` to a different array.

### Q3: What's the output and why?

```javascript
function test() {
  console.log(a);
  console.log(b);
  var a = 1;
  let b = 2;
}
test();
```

**Answer:** The first `console.log` outputs `undefined` (var hoisting), and the second throws a `ReferenceError` (Temporal Dead Zone for let).

### Q4: What's wrong with this code?

```javascript
const config = {
  port: 3000,
  host: "localhost"
};
Object.freeze(config);

const settings = {
  server: config
};
Object.freeze(settings);

settings.server.port = 8080; // Does this fail?
```

**Answer:** No, it succeeds. `Object.freeze()` is shallow - freezing `settings` doesn't freeze the nested `config` object. You'd need deep freezing to prevent this.

## Common Pitfalls

### 1. Thinking const Makes Objects Immutable

```javascript
// Wrong assumption
const data = { value: 1 };
// "const means I can't change data.value, right?"
data.value = 2; // Actually works fine

// Correct understanding
// const prevents reassignment of 'data'
// It does NOT prevent mutation of the object
```

### 2. Using var in Callbacks

```javascript
// Problem
for (var i = 0; i < 5; i++) {
  document.getElementById(`btn${i}`).addEventListener('click', function() {
    alert(i); // Always alerts 5!
  });
}

// Solution 1: Use let
for (let i = 0; i < 5; i++) {
  document.getElementById(`btn${i}`).addEventListener('click', function() {
    alert(i); // Alerts correct value
  });
}

// Solution 2: IIFE with var (legacy code)
for (var i = 0; i < 5; i++) {
  (function(index) {
    document.getElementById(`btn${index}`).addEventListener('click', function() {
      alert(index);
    });
  })(i);
}
```

### 3. Hoisting Confusion

```javascript
// What does this output?
function confusing() {
  console.log(typeof value);
  
  if (false) {
    var value = "hello";
  }
}
confusing(); // "undefined" - var is hoisted even from unexecuted block!
```

### 4. Block Scope Misunderstanding

```javascript
// Won't work as expected
const functions = [];

for (let i = 0; i < 3; i++) {
  functions.push(() => i);
}

console.log(functions[0]()); // 0 - correct!
console.log(functions[1]()); // 1 - correct!

// But this won't work:
let i = 0;
const functions2 = [];

while (i < 3) {
  functions2.push(() => i);
  i++;
}

console.log(functions2[0]()); // 3 - all reference the same 'i'!
```

## Performance Considerations

The performance difference between `var`, `let`, and `const` is negligible in modern JavaScript engines. Engine optimizations have largely eliminated any performance gaps:

```javascript
// Don't make decisions based on performance myths
const x = 1; // Not "slower" than var
let y = 2;   // Not "slower" than var
var z = 3;   // Not "faster" than let/const
```

The real benefits of `let` and `const` are:
- **Code clarity**: Explicit intent (mutable vs immutable binding)
- **Bug prevention**: Block scoping and TDZ catch errors early
- **Maintainability**: Prevents accidental reassignment and variable shadowing issues

## Resources

### Official Documentation
- [MDN: var](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/var)
- [MDN: let](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let)
- [MDN: const](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/const)
- [ECMAScript Specification: Lexical Environment](https://tc39.es/ecma262/#sec-lexical-environments)

### Articles and Guides
- [ES6 In Depth: let and const](https://hacks.mozilla.org/2015/07/es6-in-depth-let-and-const/)
- [You Don't Know JS: Scope & Closures](https://github.com/getify/You-Dont-Know-JS/tree/2nd-ed/scope-closures)
- [Temporal Dead Zone (TDZ) Demystified](http://jsrocks.org/2015/01/temporal-dead-zone-tdz-demystified/)

### Tools
- [ESLint rules](https://eslint.org/docs/rules/): `no-var`, `prefer-const`, `no-const-assign`
- [Babel](https://babeljs.io/): Transpiler that converts let/const to var for older browsers
