# Hoisting

## Table of Contents
- [Overview](#overview)
- [Variable Hoisting](#variable-hoisting)
- [Function Hoisting](#function-hoisting)
- [Temporal Dead Zone](#temporal-dead-zone)
- [How the Interpreter Processes Code](#how-the-interpreter-processes-code)
- [Execution Context and Hoisting](#execution-context-and-hoisting)
- [Common Misconceptions](#common-misconceptions)
- [Performance Implications](#performance-implications)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Overview

Hoisting is JavaScript's behavior of moving declarations to the top of their scope during the compilation phase, before code execution. However, only declarations are hoisted, not initializations. Understanding hoisting is crucial for avoiding bugs and understanding JavaScript's execution model.

## Variable Hoisting

Variables declared with `var`, `let`, and `const` are all hoisted, but behave differently.

### var Hoisting

```javascript
// What you write
console.log(x); // undefined (not ReferenceError)
var x = 5;
console.log(x); // 5

// How JavaScript interprets it (conceptually)
var x; // Declaration hoisted to top
console.log(x); // undefined (declared but not initialized)
x = 5; // Initialization stays in place
console.log(x); // 5
```

### var Hoisting in Functions

```javascript
function example() {
  console.log(message); // undefined
  var message = 'Hello';
  console.log(message); // 'Hello'
}

// Interpreted as:
function example() {
  var message; // Hoisted to top of function scope
  console.log(message); // undefined
  message = 'Hello';
  console.log(message); // 'Hello'
}

example();
```

### Multiple Declarations

```javascript
var x = 1;
console.log(x); // 1

var x = 2; // No error, re-declaration allowed with var
console.log(x); // 2

// Interpreted as:
var x;
x = 1;
console.log(x); // 1
x = 2; // Just reassignment, declaration already hoisted
console.log(x); // 2
```

### let and const Hoisting

```javascript
// let and const ARE hoisted, but behave differently
console.log(x); // ReferenceError: Cannot access 'x' before initialization
let x = 5;

// const has same behavior
console.log(y); // ReferenceError: Cannot access 'y' before initialization
const y = 10;

// Why? Temporal Dead Zone (TDZ) - covered in detail later
```

### Block Scope Hoisting

```javascript
// var: Function-scoped, hoisted to function top
function example1() {
  console.log(x); // undefined (hoisted)
  
  if (true) {
    var x = 5; // Hoisted to function top
  }
  
  console.log(x); // 5 (accessible outside block)
}

// let/const: Block-scoped, hoisted to block top
function example2() {
  // console.log(x); // ReferenceError (TDZ)
  
  if (true) {
    let x = 5; // Hoisted to block top (inside if)
    console.log(x); // 5
  }
  
  // console.log(x); // ReferenceError (x is block-scoped)
}

example1();
example2();
```

## Function Hoisting

### Function Declarations

```javascript
// Function declarations are fully hoisted
greet(); // 'Hello!' - works before declaration

function greet() {
  console.log('Hello!');
}

greet(); // 'Hello!' - works after declaration too

// Why? Entire function (declaration + body) is hoisted
```

### Function Expressions

```javascript
// Function expressions are NOT hoisted
// greet(); // TypeError: greet is not a function

var greet = function() {
  console.log('Hello!');
};

greet(); // 'Hello!' - works after assignment

// Interpreted as:
var greet; // Variable declaration hoisted
// greet(); // greet is undefined, not a function
greet = function() { // Assignment stays in place
  console.log('Hello!');
};
greet(); // Now works
```

### Arrow Functions

```javascript
// Arrow functions behave like function expressions
// greet(); // TypeError: greet is not a function

var greet = () => {
  console.log('Hello!');
};

greet(); // 'Hello!'

// let/const with arrow functions
// sayHi(); // ReferenceError: Cannot access 'sayHi' before initialization

const sayHi = () => {
  console.log('Hi!');
};

sayHi(); // 'Hi!'
```

### Function Declaration Precedence

```javascript
// Function declarations take precedence over variable declarations
var greet = 'Hello';

function greet() {
  console.log('Function');
}

console.log(typeof greet); // 'string'
console.log(greet); // 'Hello'

// Interpreted as:
function greet() { // Function declaration hoisted first
  console.log('Function');
}
var greet; // Variable declaration hoisted (but ignored, already declared)
greet = 'Hello'; // Assignment happens
console.log(typeof greet); // 'string'
console.log(greet); // 'Hello'
```

### Nested Functions

```javascript
function outer() {
  inner(); // Works! inner is hoisted
  
  function inner() {
    console.log('Inner function');
  }
}

outer(); // 'Inner function'

// But function expressions aren't hoisted
function outer2() {
  // inner2(); // TypeError: inner2 is not a function
  
  var inner2 = function() {
    console.log('Inner function');
  };
  
  inner2(); // Works after assignment
}
```

## Temporal Dead Zone

The Temporal Dead Zone (TDZ) is the period between entering scope and the variable's initialization where accessing the variable throws a ReferenceError.

### TDZ with let/const

```javascript
// TDZ starts at beginning of scope
{
  // TDZ for x starts here
  // console.log(x); // ReferenceError: Cannot access 'x' before initialization
  // TDZ continues...
  
  let x = 5; // TDZ ends here, x is initialized
  console.log(x); // 5 - safe to use
}

// const behaves identically
{
  // TDZ for y starts
  // const temp = y; // ReferenceError
  
  const y = 10; // TDZ ends
  console.log(y); // 10
}
```

### TDZ is Temporal, Not Spatial

```javascript
// TDZ is about time, not position in code
{
  const func = () => console.log(x); // No error here
  
  // TDZ for x
  // func(); // ReferenceError if called here
  
  let x = 5; // TDZ ends
  
  func(); // 5 - works here, x is initialized
}
```

### TDZ with Function Parameters

```javascript
// Default parameters access other parameters
function example(a = b, b = 2) { // ReferenceError
  return a + b;
}

// example(); // ReferenceError: Cannot access 'b' before initialization

// Why? Parameters initialized left-to-right, 'b' in TDZ when 'a' defaults

// Fix: Reorder parameters
function fixed(b = 2, a = b) {
  return a + b;
}

console.log(fixed()); // 4 (2 + 2)
```

### TDZ in Conditional Blocks

```javascript
let x = 10;

if (true) {
  // TDZ for block-scoped x starts
  // console.log(x); // ReferenceError (not 10!)
  
  let x = 20; // Different x, block-scoped, TDZ ends
  console.log(x); // 20
}

console.log(x); // 10 (outer x)

// The block-scoped x shadows outer x and has its own TDZ
```

### typeof and TDZ

```javascript
// Unusual TDZ behavior with typeof
console.log(typeof undeclared); // 'undefined' (no error for undeclared)

// console.log(typeof x); // ReferenceError (x is in TDZ)
let x = 5;

// typeof can't save you from TDZ errors
```

## How the Interpreter Processes Code

JavaScript execution happens in two phases: compilation and execution.

### Compilation Phase

```javascript
// Source code
function example() {
  console.log(x);
  var x = 5;
  console.log(x);
}

// Compilation phase:
// 1. Create Execution Context
// 2. Scan code for declarations
// 3. Allocate memory for variables (hoisting)
// 4. Initialize var to undefined
// 5. Create function objects for function declarations

// Execution phase:
// 1. Execute code line by line
// 2. Assign values to variables
// 3. Call functions
```

### Variable Environment Setup

```javascript
function demo() {
  var a = 1;
  let b = 2;
  const c = 3;
  
  function inner() {
    return a + b + c;
  }
}

// During compilation:
// Variable Environment created with:
// - a: undefined (var initialized to undefined)
// - b: <uninitialized> (let in TDZ)
// - c: <uninitialized> (const in TDZ)
// - inner: function object

// During execution:
// - a = 1 (assignment)
// - b = 2 (initialization, exits TDZ)
// - c = 3 (initialization, exits TDZ)
```

### Function Declaration Processing

```javascript
// Function declarations processed before any code executes
greet(); // Works

function greet() {
  console.log('Hello');
}

// Compilation phase:
// 1. Scanner finds function declaration
// 2. Creates function object immediately
// 3. Stores in Variable Environment
// 4. Entire function available before execution phase
```

### Execution Context Creation

```javascript
// Global Execution Context
var globalVar = 'global';

function outer() {
  // Outer Function Execution Context
  var outerVar = 'outer';
  
  function inner() {
    // Inner Function Execution Context
    var innerVar = 'inner';
    console.log(innerVar, outerVar, globalVar);
  }
  
  inner();
}

outer();

// Each execution context has:
// 1. Variable Environment (hoisted variables)
// 2. Scope Chain (access to outer scopes)
// 3. this binding
```

### Creation vs Execution Phase

```javascript
function example() {
  // Creation phase (before code runs):
  // - myVar: undefined
  // - myFunc: function object
  // - myLet: <uninitialized>
  
  console.log(myVar); // undefined (var initialized)
  console.log(myFunc); // function (fully hoisted)
  // console.log(myLet); // ReferenceError (TDZ)
  
  var myVar = 1;
  
  function myFunc() {
    return 'function';
  }
  
  let myLet = 2;
  
  // Execution phase:
  // - myVar = 1 (assignment)
  // - myLet = 2 (initialization)
  
  console.log(myVar); // 1
  console.log(myFunc()); // 'function'
  console.log(myLet); // 2
}

example();
```

## Execution Context and Hoisting

### Global Execution Context

```javascript
// Global scope
console.log(globalVar); // undefined
var globalVar = 'global';

greet(); // 'Hello'
function greet() {
  console.log('Hello');
}

// Global Execution Context:
// Creation phase:
// - Global Object (window/global)
// - this = Global Object
// - Variable Environment:
//   - globalVar: undefined
//   - greet: function

// Execution phase:
// - globalVar = 'global'
// - Execute console.log and function calls
```

### Function Execution Context

```javascript
function example(param) {
  console.log(param); // 5 (parameter)
  console.log(localVar); // undefined (hoisted)
  
  var localVar = 10;
  
  function innerFunc() {
    console.log('inner');
  }
  
  innerFunc(); // 'inner'
}

example(5);

// Function Execution Context for example():
// Creation phase:
// - arguments object: { 0: 5, length: 1 }
// - this binding
// - Variable Environment:
//   - param: 5
//   - localVar: undefined
//   - innerFunc: function

// Execution phase:
// - localVar = 10
// - Execute function body
```

### Call Stack and Hoisting

```javascript
function first() {
  console.log('First');
  second();
  console.log('First Again');
}

function second() {
  console.log('Second');
  third();
  console.log('Second Again');
}

function third() {
  console.log('Third');
}

first();

// Call Stack:
// 1. Global Execution Context (all functions hoisted)
// 2. first() pushed onto stack
// 3. second() pushed onto stack
// 4. third() pushed onto stack
// 5. third() pops off
// 6. second() pops off
// 7. first() pops off

// Output:
// First
// Second
// Third
// Second Again
// First Again
```

### Hoisting in Different Scopes

```javascript
// Global scope
var global = 'global';

function outer() {
  // Function scope
  console.log(global); // 'global'
  console.log(functionScoped); // undefined (hoisted)
  
  var functionScoped = 'function';
  
  if (true) {
    // Block scope
    console.log(functionScoped); // 'function'
    // console.log(blockScoped); // ReferenceError (TDZ)
    
    let blockScoped = 'block';
    console.log(blockScoped); // 'block'
  }
  
  console.log(functionScoped); // 'function'
  // console.log(blockScoped); // ReferenceError (not in scope)
}

outer();

// Hoisting operates at different scope levels:
// - Global scope: var, function declarations
// - Function scope: var, function declarations, parameters
// - Block scope: let, const
```

## Common Misconceptions

### Misconception 1: "Hoisting Physically Moves Code"

**Reality**: Hoisting is a conceptual model. Code doesn't actually move; the JavaScript engine allocates memory for declarations during compilation.

```javascript
// Code doesn't physically move
console.log(x); // undefined
var x = 5;

// Engine creates Variable Environment during compilation:
// { x: undefined }
// Then assigns during execution:
// { x: 5 }
```

### Misconception 2: "let/const Aren't Hoisted"

**Reality**: `let` and `const` ARE hoisted, but remain in the Temporal Dead Zone until initialization.

```javascript
{
  // x is hoisted but in TDZ
  // console.log(x); // ReferenceError (if not hoisted, would say "not defined")
  let x = 5;
}

// TDZ proves hoisting:
let x = 10;
{
  // console.log(x); // ReferenceError: Cannot access 'x' before initialization
  // If inner x wasn't hoisted, would access outer x (10)
  let x = 20; // Inner x hoisted, shadows outer, but in TDZ
}
```

### Misconception 3: "Function Expressions Are Hoisted Like Declarations"

**Reality**: Function expressions follow variable hoisting rules, not function declaration rules.

```javascript
// Function declaration: Fully hoisted
greet1(); // Works
function greet1() {
  console.log('Hello');
}

// Function expression: Variable hoisted, function not
// greet2(); // TypeError: greet2 is not a function
var greet2 = function() {
  console.log('Hello');
};
greet2(); // Works
```

### Misconception 4: "Hoisting Happens During Runtime"

**Reality**: Hoisting happens during the compilation phase, before any code executes.

```javascript
// Compilation phase: All declarations processed
// Execution phase: Code runs line by line

function example() {
  console.log('Start'); // Execution starts here
  var x = 5; // Assignment during execution
  console.log(x); // 5
}

// During compilation:
// - Function 'example' created
// - Variable 'x' allocated (initialized to undefined)

// During execution:
// - console.log executes
// - x = 5 executes
// - console.log executes
```

## Performance Implications

### var vs let/const Performance

```javascript
// var: Hoisted and initialized to undefined
function testVar() {
  console.log(x); // undefined, no error
  var x = 5;
}

// let/const: Hoisted but TDZ checks needed
function testLet() {
  // TDZ check happens at runtime (slight overhead)
  // console.log(x); // ReferenceError
  let x = 5;
}

// Modern engines optimize TDZ checks away
// Performance difference is negligible in practice
```

### Function Declaration vs Expression

```javascript
// Function declarations: Available immediately
function declared() {
  return 'immediate';
}

declared(); // Fast, function object ready

// Function expressions: Created during execution
const expressed = function() {
  return 'created';
};

expressed(); // Slightly slower first call, negligible difference

// In practice, performance difference is minimal
// Choose based on code organization, not performance
```

### Avoiding Hoisting Pitfalls

```javascript
// BAD: Relying on hoisting (unclear code)
function bad() {
  x = 5; // Looks like error, but works due to hoisting
  console.log(x);
  var x;
}

// GOOD: Declare at top (clear intent)
function good() {
  var x;
  x = 5;
  console.log(x);
}

// BEST: Use let/const (block-scoped, clear)
function best() {
  let x = 5;
  console.log(x);
}
```

### Minimizing Scope for Performance

```javascript
// BAD: Function-scoped var accessible throughout
function processArray(arr) {
  var sum = 0;
  for (var i = 0; i < arr.length; i++) {
    sum += arr[i];
  }
  // 'i' still exists here (function-scoped)
  return sum;
}

// GOOD: Block-scoped let, narrower scope
function processArrayBetter(arr) {
  let sum = 0;
  for (let i = 0; i < arr.length; i++) { // 'i' block-scoped to loop
    sum += arr[i];
  }
  // 'i' doesn't exist here (can be GC'd earlier)
  return sum;
}
```

## Interview Questions

### Question 1: What is hoisting in JavaScript?

**Answer**: Hoisting is JavaScript's behavior of processing declarations during the compilation phase before code execution. Variable and function declarations are "hoisted" to the top of their scope. However, only declarations are hoisted, not initializations. `var` declarations are initialized to `undefined`, while `let` and `const` remain in a Temporal Dead Zone until their initialization statement is executed.

### Question 2: What's the difference between var, let, and const hoisting?

**Answer**:
- `var`: Hoisted to function scope, initialized to `undefined`, accessible before declaration (returns `undefined`)
- `let`/`const`: Hoisted to block scope, remain in Temporal Dead Zone (TDZ) until initialization, accessing before declaration throws ReferenceError

All three are hoisted, but their behavior differs. `var` is accessible (as `undefined`) before declaration, while `let`/`const` throw errors if accessed before initialization.

### Question 3: What is the Temporal Dead Zone?

**Answer**: The Temporal Dead Zone (TDZ) is the period between entering a scope where a `let` or `const` variable is declared and the point where it's initialized. Accessing the variable during this period throws a ReferenceError. The TDZ is "temporal" (time-based) not "spatial" (position-based) - it depends on when code executes, not where in the code the variable is accessed.

### Question 4: How are function declarations hoisted differently from function expressions?

**Answer**: 
- Function declarations: Fully hoisted with their body, available before declaration
- Function expressions: Follow variable hoisting rules (variable hoisted, but function created during execution)

Example:
```javascript
greet1(); // Works
function greet1() {} // Declaration

greet2(); // TypeError
var greet2 = function() {}; // Expression
```

### Question 5: Why does this code throw an error?

```javascript
let x = 10;
{
  console.log(x); // ReferenceError
  let x = 20;
}
```

**Answer**: The inner `let x` is hoisted to the top of the block scope and shadows the outer `x`. However, it's in the Temporal Dead Zone until initialization. The `console.log` tries to access the inner `x` while it's still in the TDZ, causing a ReferenceError. If the inner `x` wasn't hoisted, the code would access the outer `x` (10) successfully.

## Key Takeaways

1. **Hoisting is compilation behavior**: Declarations processed before execution, not physical code movement

2. **Only declarations are hoisted**: Initializations remain in place

3. **var initialized to undefined**: Accessible (as `undefined`) before declaration line

4. **let/const have TDZ**: Hoisted but inaccessible until initialization, throw ReferenceError if accessed early

5. **Function declarations fully hoisted**: Available before declaration line

6. **Function expressions not fully hoisted**: Follow variable hoisting rules

7. **TDZ is temporal, not spatial**: Based on execution time, not code position

8. **Block scope vs function scope**: `let`/`const` hoisted to block, `var` to function

9. **Best practice**: Declare variables at top of scope for clarity

10. **Use let/const**: Block scoping prevents many hoisting-related bugs

## Resources

### Official Documentation
- [MDN: Hoisting](https://developer.mozilla.org/en-US/docs/Glossary/Hoisting)
- [MDN: Temporal Dead Zone](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let#temporal_dead_zone_tdz)
- [ECMAScript Specification: Execution Contexts](https://tc39.es/ecma262/#sec-execution-contexts)

### Articles
- "JavaScript Hoisting Explained" by Marius Schulz
- "Understanding Hoisting in JavaScript" by Scotch.io
- "Temporal Dead Zone (TDZ) Demystified" by Dmitri Pavlutin

### Books
- "You Don't Know JS: Scope & Closures" by Kyle Simpson
- "JavaScript: The Definitive Guide" by David Flanagan
- "Eloquent JavaScript" by Marijn Haverbeke

### Videos
- "JavaScript Hoisting Explained" by Web Dev Simplified
- "var, let and const - What, why and how" by Fun Fun Function
- "Temporal Dead Zone" by Traversy Media
