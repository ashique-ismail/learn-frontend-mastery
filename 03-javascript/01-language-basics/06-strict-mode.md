# Strict Mode in JavaScript

## The Idea

**In plain English:** Strict mode is a setting you can turn on in JavaScript that makes the language more picky — it refuses to let you make certain sloppy mistakes that would otherwise be silently ignored, and instead tells you about them right away with an error.

**Real-world analogy:** Imagine a spell-checker that you can switch from "suggestions only" mode to "strict mode." In suggestions mode, it lets you save a document full of typos without complaining. In strict mode, it stops you the moment you try to save and points out every problem.

- The spell-checker = JavaScript's runtime engine
- Switching to strict mode = writing `"use strict"` at the top of your file
- Saving a document with typos = running code that silently creates bugs
- The error pop-up blocking you = a ReferenceError or TypeError thrown immediately

---

## Overview

Strict mode is a way to opt into a restricted variant of JavaScript that eliminates some of JavaScript's silent errors by changing them to throw errors, fixes mistakes that make it difficult for JavaScript engines to perform optimizations, and prohibits some syntax likely to be defined in future versions of ECMAScript. Introduced in ES5 (2009), strict mode is now a best practice and is automatically enabled in ES6 modules and classes.

## Enabling Strict Mode

### Global Strict Mode

Apply to entire script file:

```javascript
"use strict";

// All code in this file runs in strict mode
function myFunction() {
  // Also in strict mode
}
```

### Function-Level Strict Mode

Apply to specific function:

```javascript
function myFunction() {
  "use strict";
  // Only this function runs in strict mode
}

function normalFunction() {
  // Regular mode
}
```

### Module and Class Strict Mode

ES6 modules and classes are automatically in strict mode:

```javascript
// In .mjs file or <script type="module">
// Automatically strict mode, no directive needed
export function myFunction() {
  // Strict mode
}

// Classes are always strict
class MyClass {
  constructor() {
    // Strict mode
  }
}
```

## Key Differences and Restrictions

### 1. Variables Must Be Declared

```javascript
"use strict";

// Error: Cannot assign to undeclared variable
x = 10; // ReferenceError: x is not defined

// Must declare
let x = 10; // OK
const y = 20; // OK
var z = 30; // OK

// Non-strict mode (bad practice)
function sloppy() {
  implicitGlobal = 42; // Creates global variable silently
  console.log(window.implicitGlobal); // 42
}
```

### 2. Assignment to Non-Writable Properties Throws Error

```javascript
"use strict";

// Non-writable property
const obj = {};
Object.defineProperty(obj, 'x', {
  value: 42,
  writable: false
});

obj.x = 100; // TypeError: Cannot assign to read only property

// Non-strict: Fails silently
// Strict: Throws TypeError
```

### 3. Deleting Properties

```javascript
"use strict";

// Cannot delete non-configurable properties
const obj = {};
Object.defineProperty(obj, 'x', {
  value: 42,
  configurable: false
});

delete obj.x; // TypeError: Cannot delete property

// Cannot delete variables
let myVar = 10;
delete myVar; // SyntaxError

// Cannot delete functions
function myFunc() {}
delete myFunc; // SyntaxError

// Can delete normal properties
const obj2 = { prop: 1 };
delete obj2.prop; // OK
```

### 4. Duplicate Parameter Names Forbidden

```javascript
"use strict";

// Error: Duplicate parameter names
function sum(a, a, c) { // SyntaxError
  return a + a + c;
}

// Non-strict: Last parameter wins
function sloppySum(a, a, c) {
  return a + a + c; // Uses second 'a'
}
sloppySum(1, 2, 3); // 7 (2 + 2 + 3)
```

### 5. Octal Literals Forbidden

```javascript
"use strict";

// Error: Octal literals not allowed
const num = 010; // SyntaxError

// Use new syntax instead
const octal = 0o10; // 8 in decimal (OK)
const hex = 0x10;   // 16 in decimal (OK)
const binary = 0b10; // 2 in decimal (OK)

// Non-strict: 010 = 8 (confusing!)
```

### 6. with Statement Forbidden

```javascript
"use strict";

// Error: with statement not allowed
with (obj) { // SyntaxError
  console.log(prop);
}

// Use explicit property access instead
console.log(obj.prop);

// Or destructuring
const { prop } = obj;
console.log(prop);
```

### 7. this Value Changes

```javascript
"use strict";

// In global context: undefined (not window/global)
function showThis() {
  console.log(this);
}

showThis(); // undefined (strict)
            // window (non-strict)

// In methods: still the object
const obj = {
  method() {
    console.log(this); // obj
  }
};
obj.method(); // obj

// Explicit binding still works
showThis.call({ name: 'test' }); // { name: 'test' }
```

### 8. Function Declarations in Blocks

```javascript
"use strict";

// Function declarations in blocks are restricted
if (true) {
  function foo() { // Allowed but block-scoped
    return 1;
  }
}

console.log(typeof foo); // "function" in strict mode
                         // May vary in non-strict mode

// Better: Use function expressions or arrow functions
if (true) {
  const foo = () => 1; // Clear and consistent
}
```

### 9. eval Has Its Own Scope

```javascript
"use strict";

// Variables declared in eval don't leak
eval("var x = 10;");
console.log(typeof x); // "undefined"

// Non-strict: x would be in outer scope
// Strict: eval variables are local to eval

// Can still access outer variables
let y = 20;
eval("console.log(y)"); // 20
eval("y = 30;");
console.log(y); // 30
```

### 10. arguments Object Differences

```javascript
"use strict";

function test(a) {
  arguments[0] = 100; // Change arguments
  console.log(a);     // Still 1 in strict mode
                      // Would be 100 in non-strict mode
}
test(1);

// arguments is not aliased to parameters
function strict(a) {
  "use strict";
  a = 99;
  console.log(arguments[0]); // Still 1 (original value)
}
strict(1);

// Cannot assign to arguments
function cantModify() {
  "use strict";
  arguments = [1, 2, 3]; // TypeError
}
```

### 11. Reserved Words

```javascript
"use strict";

// These are reserved and cannot be used as identifiers
let implements; // SyntaxError
let interface;  // SyntaxError
let package;    // SyntaxError
let private;    // SyntaxError
let protected;  // SyntaxError
let public;     // SyntaxError
let static;     // SyntaxError
let yield;      // SyntaxError (already reserved)
let let;        // SyntaxError (already reserved)

// Future reserved words
let enum;       // SyntaxError
```

## Why Use Strict Mode?

### 1. Catches Common Mistakes

```javascript
"use strict";

// Typo in variable name
let myVariable = 10;
myVariabel = 20; // ReferenceError: myVariabel is not defined

// Non-strict: Creates global variable silently!
```

### 2. Prevents Accidental Globals

```javascript
"use strict";

function calculateTotal() {
  total = 0; // ReferenceError
  for (let item of items) {
    total += item.price;
  }
  return total;
}

// Forces you to declare
function calculateTotal() {
  let total = 0; // Explicit and clear
  for (let item of items) {
    total += item.price;
  }
  return total;
}
```

### 3. Better Performance

Strict mode enables engine optimizations:

```javascript
"use strict";

// Engines can optimize better because:
// - No need to check for implicit globals
// - No arguments aliasing
// - Predictable 'this' binding
// - No eval scope leakage

function compute(a, b, c) {
  return a * b + c;
}
```

### 4. Easier Debugging

```javascript
"use strict";

// Immediate errors instead of silent failures
const config = Object.freeze({ timeout: 5000 });
config.timeout = 10000; // TypeError (caught immediately)

// Non-strict: Fails silently, hard to debug
```

### 5. Future-Proof Code

```javascript
"use strict";

// Prevents using future reserved words
// Disallows problematic syntax
// Aligns with ES6+ defaults (modules, classes)
```

## Strict Mode Gotchas

### 1. Mixed Strict and Non-Strict

```javascript
// File 1: strict mode
"use strict";
function strictFunc() {
  // Strict
}

// File 2: non-strict
function normalFunc() {
  // Non-strict
}

// When concatenated: First directive wins!
// Can cause unexpected behavior
```

### 2. this in Functions

```javascript
"use strict";

const obj = {
  value: 42,
  getValue: function() {
    return this.value;
  }
};

const getValue = obj.getValue;
console.log(getValue()); // TypeError: Cannot read property 'value' of undefined

// Solution: Bind
const boundGetValue = obj.getValue.bind(obj);
console.log(boundGetValue()); // 42

// Or use arrow functions (lexical this)
const obj2 = {
  value: 42,
  getValue: () => this.value // 'this' from outer scope
};
```

### 3. Third-Party Libraries

```javascript
// Some old libraries expect non-strict mode
// They may break when you enable strict mode

// Solution: Use function-level strict mode
function myCode() {
  "use strict";
  // Your strict code
}

// Or load library code in non-strict context
```

### 4. Direct vs Indirect eval

```javascript
"use strict";

// Direct eval (has own scope in strict mode)
eval("var x = 1;");
console.log(typeof x); // "undefined"

// Indirect eval (always global scope)
const indirectEval = eval;
indirectEval("var y = 2;");
console.log(typeof y); // "function" (may vary)

// Best practice: Avoid eval entirely
const computedValue = Function("return 1 + 2")(); // 3
```

## Strict Mode in Modern JavaScript

### ES6 Modules Are Strict by Default

```javascript
// In .mjs or <script type="module">
// No need for "use strict"

export function myFunction() {
  // Already in strict mode
  undeclaredVar = 10; // ReferenceError
}

// Cannot opt out of strict mode in modules!
```

### Classes Are Strict by Default

```javascript
class MyClass {
  // No need for "use strict"
  constructor() {
    // Already in strict mode
    console.log(this); // undefined if called without 'new'
  }
  
  method() {
    // Strict mode
  }
}

// Even when called from non-strict context
```

### ESLint Rules

Modern development uses linting:

```javascript
// .eslintrc.js
module.exports = {
  rules: {
    'strict': ['error', 'global'], // Enforce strict mode
    'no-implied-eval': 'error',
    'no-var': 'error',
  }
};
```

## Best Practices

### 1. Always Use Strict Mode

```javascript
// Top of every script file
"use strict";

// Or use ES6 modules (automatic)
export function myFunction() {
  // Strict mode
}
```

### 2. Enable at File Level, Not Function Level

```javascript
// Good: Consistent across file
"use strict";

function func1() {
  // Strict
}

function func2() {
  // Strict
}

// Avoid: Inconsistent
function mixed1() {
  "use strict";
  // Strict
}

function mixed2() {
  // Non-strict - confusing!
}
```

### 3. Use ES6+ Features

Modern JavaScript features work best with strict mode:

```javascript
// Use let/const instead of var
const value = 10;

// Use arrow functions (lexical this)
const obj = {
  method: () => {
    // Lexical this
  }
};

// Use modules
import { something } from './module.js';

// Use classes
class MyClass {
  // Always strict
}
```

### 4. Configure Build Tools

```javascript
// Babel config
{
  "presets": [
    ["@babel/preset-env", {
      // Modern output includes strict mode
    }]
  ]
}

// Webpack
module.exports = {
  // Modules are strict by default
  mode: 'production'
};
```

### 5. Update Legacy Code Gradually

```javascript
// Strategy 1: Wrap in IIFE
(function() {
  "use strict";
  // Modernize code here
})();

// Strategy 2: Convert to modules
// old-file.js -> old-file.mjs
export function legacyFunction() {
  // Now in strict mode
}

// Strategy 3: Function-by-function
function modernFunction() {
  "use strict";
  // Updated function
}
```

## Common Errors and Solutions

### Problem: Accidental Global

```javascript
"use strict";

function calculate() {
  result = 10; // ReferenceError
}

// Solution: Declare variable
function calculate() {
  let result = 10;
  return result;
}
```

### Problem: this is undefined

```javascript
"use strict";

function MyConstructor() {
  this.value = 10; // TypeError if called without 'new'
}

const obj = MyConstructor(); // Error!

// Solution 1: Use class
class MyConstructor {
  constructor() {
    this.value = 10;
  }
}

// Solution 2: Add check
function MyConstructor() {
  if (!(this instanceof MyConstructor)) {
    return new MyConstructor();
  }
  this.value = 10;
}
```

### Problem: Read-Only Property

```javascript
"use strict";

const obj = {};
Object.defineProperty(obj, 'prop', {
  value: 42,
  writable: false
});

obj.prop = 100; // TypeError

// Solution: Check if writable or create new object
const newObj = {
  ...obj,
  prop: 100 // Creates new object with updated prop
};
```

## Interview Questions

### Q1: What happens in strict mode vs non-strict?

```javascript
function test() {
  x = 10;
  return x;
}

test();
console.log(x);
```

**Answer:**

- Non-strict: Creates global variable `x`, prints 10
- Strict: ReferenceError at `x = 10`

### Q2: What's the difference?

```javascript
function getValue() {
  return this.value;
}

const obj = { value: 42, getValue };
const fn = obj.getValue;

obj.getValue(); // ?
fn();           // ?
```

**Answer:**

- `obj.getValue()`: Returns 42 (this is obj)
- Non-strict `fn()`: Returns undefined (this is window/global)
- Strict `fn()`: TypeError (this is undefined)

### Q3: Is this valid strict mode code?

```javascript
"use strict";

function sum(a, b, a) {
  return a + b + a;
}
```

**Answer:** No, SyntaxError. Duplicate parameter names are not allowed in strict mode.

## Resources

### Documentation

- [MDN: Strict Mode](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode)
- [ECMAScript Strict Mode](https://tc39.es/ecma262/#sec-strict-mode-code)

### Articles

- [JavaScript Strict Mode Explained](https://javascript.info/strict-mode)
- [What Does "use strict" Do?](https://stackoverflow.com/questions/1335851/what-does-use-strict-do-in-javascript)

### Tools

- [ESLint strict rule](https://eslint.org/docs/rules/strict)
- [Babel: Transform strict mode](https://babeljs.io/docs/en/babel-plugin-transform-strict-mode)
