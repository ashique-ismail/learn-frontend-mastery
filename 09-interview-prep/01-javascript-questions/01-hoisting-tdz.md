# Hoisting and TDZ - JavaScript Interview Questions

## Table of Contents
- [Core Concepts](#core-concepts)
- [Common Interview Questions](#common-interview-questions)
- [Advanced Questions](#advanced-questions)
- [Practical Examples](#practical-examples)
- [What Interviewers Look For](#what-interviewers-look-for)
- [Red Flags to Avoid](#red-flags-to-avoid)
- [Key Takeaways](#key-takeaways)

## Core Concepts

### What is Hoisting?

Hoisting is JavaScript's behavior of moving variable and function declarations to the top of their scope during the compilation phase, before code execution.

**Important**: Only declarations are hoisted, not initializations.

### Temporal Dead Zone (TDZ)

The TDZ is the period between entering scope and the variable declaration being executed, during which the variable cannot be accessed.

### Three Variable Declaration Types

1. **var**: Function-scoped, hoisted with undefined
2. **let**: Block-scoped, hoisted but in TDZ
3. **const**: Block-scoped, hoisted but in TDZ, must be initialized

## Common Interview Questions

### Question 1: What is Hoisting? Explain with Examples

**Expected Answer:**

Hoisting is JavaScript's default behavior of moving declarations to the top of the current scope before code execution.

```javascript
// What you write
console.log(x); // undefined (not ReferenceError)
var x = 5;
console.log(x); // 5

// How JavaScript interprets it
var x; // Declaration hoisted
console.log(x); // undefined
x = 5; // Assignment stays in place
console.log(x); // 5
```

**Function Hoisting:**
```javascript
// Function declarations are fully hoisted
greet(); // "Hello!" - works before declaration

function greet() {
  console.log("Hello!");
}

// Function expressions are NOT hoisted
// sayHi(); // TypeError: sayHi is not a function

var sayHi = function() {
  console.log("Hi!");
};

sayHi(); // "Hi!" - works after assignment
```

**Why This Matters:**
```javascript
// Can cause unexpected behavior
var name = "Global";

function printName() {
  console.log(name); // undefined, not "Global"
  var name = "Local";
  console.log(name); // "Local"
}

printName();

// JavaScript sees it as:
function printName() {
  var name; // Hoisted to function scope
  console.log(name); // undefined
  name = "Local";
  console.log(name); // "Local"
}
```

**What Interviewers Look For:**
- Understanding that only declarations are hoisted
- Knowledge of var vs let/const hoisting
- Awareness of function hoisting differences
- Can explain unexpected behavior

### Question 2: Explain the Temporal Dead Zone (TDZ)

**Expected Answer:**

The TDZ is the time between entering a block scope and the variable declaration being executed, during which accessing the variable throws a ReferenceError.

```javascript
// TDZ with let
console.log(x); // ReferenceError: Cannot access 'x' before initialization
let x = 5;

// vs var (no TDZ)
console.log(y); // undefined
var y = 5;

// TDZ visualization
{
  // TDZ starts for 'value'
  // Any access to 'value' here throws ReferenceError
  
  console.log(value); // ReferenceError
  
  let value = 10; // TDZ ends, value is initialized
  
  console.log(value); // 10 - OK
}
```

**TDZ with const:**
```javascript
// const must be initialized
// const x; // SyntaxError: Missing initializer

const y = 10; // Must initialize immediately

// y = 20; // TypeError: Assignment to constant variable
```

**TDZ is Temporal, Not Spatial:**
```javascript
function test() {
  // TDZ for 'x' starts here
  
  function inner() {
    console.log(x); // ReferenceError even though declaration is below
  }
  
  inner();
  
  let x = 10; // TDZ ends
}

test();
```

**TDZ Protects Against Errors:**
```javascript
// Before ES6, easy to make mistakes
var x = 5;

if (true) {
  console.log(x); // 5 (uses outer x)
  var x = 10; // But declaration is hoisted!
  // This actually creates undefined above
}

// With let/const, error is caught
let y = 5;

if (true) {
  console.log(y); // ReferenceError - TDZ protection
  let y = 10;
}
```

### Question 3: Compare var, let, and const

**Expected Answer:**

**Scope:**
```javascript
// var - function scoped
function varScope() {
  if (true) {
    var x = 10;
  }
  console.log(x); // 10 - accessible outside block
}

// let/const - block scoped
function letConstScope() {
  if (true) {
    let y = 10;
    const z = 20;
  }
  console.log(y); // ReferenceError
  console.log(z); // ReferenceError
}
```

**Hoisting:**
```javascript
// var - hoisted with undefined
console.log(a); // undefined
var a = 1;

// let/const - hoisted but in TDZ
console.log(b); // ReferenceError
let b = 2;

console.log(c); // ReferenceError
const c = 3;
```

**Re-declaration:**
```javascript
// var - can redeclare
var x = 1;
var x = 2; // OK
console.log(x); // 2

// let - cannot redeclare in same scope
let y = 1;
// let y = 2; // SyntaxError

// const - cannot redeclare
const z = 1;
// const z = 2; // SyntaxError
```

**Re-assignment:**
```javascript
// var - can reassign
var a = 1;
a = 2; // OK

// let - can reassign
let b = 1;
b = 2; // OK

// const - cannot reassign
const c = 1;
// c = 2; // TypeError

// But can mutate objects/arrays
const obj = { value: 1 };
obj.value = 2; // OK - mutating object
// obj = {}; // TypeError - reassigning variable

const arr = [1, 2];
arr.push(3); // OK - mutating array
// arr = []; // TypeError - reassigning variable
```

**Global Object Property:**
```javascript
// var creates global property
var globalVar = 'var';
console.log(window.globalVar); // 'var' (browser)

// let/const don't
let globalLet = 'let';
const globalConst = 'const';
console.log(window.globalLet); // undefined
console.log(window.globalConst); // undefined
```

**Best Practice:**
```javascript
// Default to const
const API_URL = 'https://api.example.com';
const MAX_RETRIES = 3;

// Use let when reassignment needed
let counter = 0;
let isProcessing = false;

// Avoid var in modern code
// var shouldBeConst = 'value'; // Don't do this
```

### Question 4: Function Hoisting Patterns

**Interview Question:** "What's the difference between function declaration and function expression hoisting?"

**Expected Answer:**

**Function Declaration - Fully Hoisted:**
```javascript
// Can call before declaration
greet(); // "Hello!"

function greet() {
  console.log("Hello!");
}

// Hoisted to top of scope
if (true) {
  function test() {
    console.log("Test");
  }
}

test(); // Works (but behavior varies across environments)
```

**Function Expression - Variable Hoisting Only:**
```javascript
// Cannot call before assignment
// greet(); // TypeError: greet is not a function

var greet = function() {
  console.log("Hello!");
};

// What JavaScript sees:
var greet; // Hoisted
// greet(); // TypeError: greet is not a function
greet = function() { console.log("Hello!"); };

greet(); // "Hello!"
```

**Arrow Functions - Same as Function Expressions:**
```javascript
// Cannot call before assignment
// greet(); // ReferenceError with let/const, TypeError with var

const greet = () => {
  console.log("Hello!");
};

greet(); // "Hello!"
```

**Named Function Expressions:**
```javascript
// Variable hoisted, function not
// greet(); // TypeError

var greet = function namedGreet() {
  console.log("Hello!");
  // namedGreet(); // Can recurse using name
};

greet(); // "Hello!"
// namedGreet(); // ReferenceError - name only available inside
```

**Conditional Function Declarations (Avoid):**
```javascript
// Inconsistent behavior across browsers
if (true) {
  function test() {
    console.log("Version 1");
  }
} else {
  function test() {
    console.log("Version 2");
  }
}

// Use expressions instead
let test;
if (true) {
  test = function() {
    console.log("Version 1");
  };
} else {
  test = function() {
    console.log("Version 2");
  };
}
```

### Question 5: Class Hoisting

**Interview Question:** "Are classes hoisted in JavaScript?"

**Expected Answer:**

Classes are hoisted but remain in TDZ until declaration, similar to let/const.

```javascript
// Cannot use before declaration
// const instance = new MyClass(); // ReferenceError

class MyClass {
  constructor(value) {
    this.value = value;
  }
}

const instance = new MyClass(10); // OK

// Class expressions behave similarly
// const instance2 = new MyClass2(); // ReferenceError

const MyClass2 = class {
  constructor(value) {
    this.value = value;
  }
};

const instance2 = new MyClass2(20); // OK
```

**Unlike Functions:**
```javascript
// Function declaration - works
const dog = new Animal('dog');

function Animal(type) {
  this.type = type;
}

// Class declaration - doesn't work
// const cat = new Pet('cat'); // ReferenceError

class Pet {
  constructor(type) {
    this.type = type;
  }
}

const cat = new Pet('cat'); // OK
```

## Advanced Questions

### Question 6: Predict the Output - Complex Hoisting

**Interview Question:** "What will this code output?"

```javascript
var x = 1;

function test() {
  console.log(x);
  var x = 2;
  console.log(x);
}

test();
console.log(x);
```

**Answer:**
```
Output:
undefined
2
1

Explanation:
1. var x in test() is hoisted to function scope
2. First console.log(x) sees hoisted but uninitialized x (undefined)
3. x is assigned 2
4. Second console.log(x) outputs 2
5. Global x remains 1
```

**How JavaScript Interprets It:**
```javascript
var x = 1;

function test() {
  var x; // Hoisted
  console.log(x); // undefined
  x = 2;
  console.log(x); // 2
}

test();
console.log(x); // 1 (global x)
```

**More Complex:**
```javascript
function complex() {
  console.log(a); // undefined
  console.log(b); // ReferenceError
  
  var a = 1;
  let b = 2;
}

// After b declaration error, code stops
complex();
```

### Question 7: Hoisting with Same Names

**Interview Question:** "What happens when function and variable have the same name?"

**Expected Answer:**

Function declarations are hoisted before variable declarations.

```javascript
console.log(typeof foo); // "function"

var foo = "variable";
function foo() {
  return "function";
}

console.log(typeof foo); // "string"

// How JavaScript interprets it:
function foo() {
  return "function";
} // Function hoisted first

var foo; // Variable declaration hoisted (but ignored - already declared)

console.log(typeof foo); // "function"

foo = "variable"; // Assignment happens here

console.log(typeof foo); // "string"
```

**Multiple Function Declarations:**
```javascript
// Last one wins
foo(); // "Third"

function foo() {
  console.log("First");
}

function foo() {
  console.log("Second");
}

function foo() {
  console.log("Third");
}
```

**var and function:**
```javascript
console.log(x); // [Function: x]

var x = 10;

function x() {
  return 20;
}

console.log(x); // 10

// Interpreted as:
function x() {
  return 20;
}

var x; // Ignored - already declared

console.log(x); // [Function: x]

x = 10;

console.log(x); // 10
```

### Question 8: TDZ in Real Scenarios

**Interview Question:** "How does TDZ affect destructuring and default parameters?"

**Expected Answer:**

**Destructuring:**
```javascript
// TDZ applies to destructured variables
// console.log(x, y); // ReferenceError

const { x, y } = { x: 1, y: 2 };

console.log(x, y); // 1, 2

// With default values
function test({ value = getValue() } = {}) {
  console.log(value);
}

function getValue() {
  return 10;
}

test(); // 10
test({ value: 20 }); // 20
```

**Default Parameters:**
```javascript
// Parameters can reference earlier parameters
function test(a = 1, b = a + 1) {
  console.log(a, b);
}

test(); // 1, 2
test(5); // 5, 6

// But not later ones (TDZ)
function invalid(a = b, b = 1) {
  console.log(a, b);
}

// invalid(); // ReferenceError: Cannot access 'b' before initialization

// Parameters have their own scope
function outer(x = y) {
  let y = 'inner';
  console.log(x);
}

let y = 'outer';
outer(); // 'outer' (uses outer scope y)
```

**Complex TDZ:**
```javascript
let x = 1;

{
  // TDZ for x starts
  console.log(x); // ReferenceError (not 1 from outer scope)
  let x = 2; // TDZ ends
}

// Function parameters and TDZ
function test(a = b, b = 2) {
  console.log(a, b);
}

// test(); // ReferenceError: Cannot access 'b' before initialization

// Fixed
function testFixed(b = 2, a = b) {
  console.log(a, b);
}

testFixed(); // 2, 2
```

## Practical Examples

### Example 1: Loop Variable Hoisting

```javascript
// var - creates one variable
for (var i = 0; i < 3; i++) {
  setTimeout(() => {
    console.log(i); // 3, 3, 3
  }, 100);
}

// let - creates new variable each iteration
for (let i = 0; i < 3; i++) {
  setTimeout(() => {
    console.log(i); // 0, 1, 2
  }, 100);
}

// Why? var is hoisted to function scope
var i;
for (i = 0; i < 3; i++) {
  setTimeout(() => {
    console.log(i); // All closures reference same i
  }, 100);
}
```

### Example 2: Module Pattern with Hoisting

```javascript
var Module = (function() {
  // Declarations hoisted within IIFE
  
  var privateVar = "private";
  
  function privateFunction() {
    console.log(privateVar);
  }
  
  // Can reference before declaration due to hoisting
  publicMethod();
  
  function publicMethod() {
    privateFunction();
  }
  
  return {
    publicMethod: publicMethod
  };
})();

Module.publicMethod(); // Works
```

### Example 3: Conditional Variable Declaration

```javascript
// Avoid this pattern
function checkFlag(flag) {
  if (flag) {
    var result = "flag is true";
  } else {
    var result = "flag is false";
  }
  
  return result; // Always works due to hoisting
}

// Better with let
function checkFlagBetter(flag) {
  let result;
  
  if (flag) {
    result = "flag is true";
  } else {
    result = "flag is false";
  }
  
  return result;
}

// Even better
function checkFlagBest(flag) {
  return flag ? "flag is true" : "flag is false";
}
```

## What Interviewers Look For

### 1. Core Understanding
- What hoisting is and how it works
- Difference between declaration and initialization
- TDZ concept and purpose
- var vs let vs const behavior

### 2. Practical Knowledge
- Can predict hoisting behavior
- Understands common pitfalls
- Knows when TDZ errors occur
- Aware of function hoisting differences

### 3. Best Practices
- Prefers const over let over var
- Declares variables at top of scope
- Understands modern patterns
- Avoids hoisting-dependent code

### 4. Problem-Solving
- Can debug hoisting issues
- Identifies TDZ errors
- Refactors legacy var code
- Explains unexpected behavior

### 5. Advanced Knowledge
- Understands scope creation
- Knows about initialization phases
- Familiar with class hoisting
- Aware of parameter scoping

## Red Flags to Avoid

### 1. Fundamental Misunderstandings
- Thinking let/const aren't hoisted
- Believing TDZ is about scope
- Confusing hoisting with scope
- Not knowing var behavior

### 2. Common Mistakes
- Using var in modern code
- Accessing variables before declaration
- Not understanding loop variable binding
- Ignoring TDZ errors

### 3. Poor Practices
- Relying on hoisting
- Declaring variables at random places
- Using function declarations conditionally
- Not initializing const

### 4. Code Quality Issues
- Inconsistent variable declaration
- Mixing var, let, const unnecessarily
- Not declaring variables at all
- Poor variable naming

## Key Takeaways

### Essential Concepts
1. **Hoisting moves declarations**: Not initializations
2. **TDZ protects against errors**: Can't access before initialization
3. **var is function-scoped**: let/const are block-scoped
4. **Functions are fully hoisted**: Function declarations only
5. **Classes have TDZ**: Like let/const

### Best Practices
1. **Use const by default**: Signals immutability
2. **Use let when reassignment needed**: Clear intent
3. **Avoid var**: Use let/const instead
4. **Declare at top of scope**: Even though hoisting works
5. **Initialize variables**: Especially const

### Common Patterns
1. **Loop variables**: Use let, not var
2. **Function declarations**: Fully hoisted, use carefully
3. **Module patterns**: Aware of hoisting scope
4. **Destructuring**: Subject to TDZ
5. **Default parameters**: Can reference earlier params

### Interview Success Tips
1. **Trace execution**: Show how hoisting transforms code
2. **Explain TDZ purpose**: Error prevention, better code
3. **Compare behaviors**: var vs let vs const
4. **Show modern practices**: Prefer const/let
5. **Discuss trade-offs**: When each is appropriate
6. **Predict output**: Walk through hoisting step-by-step

Remember: While hoisting is a JavaScript feature, good code doesn't rely on it. Modern best practices with const/let and proper variable declaration make code more predictable and easier to understand. Understanding hoisting helps debug legacy code and avoid pitfalls, but writing code that's clear without relying on hoisting is the goal.
