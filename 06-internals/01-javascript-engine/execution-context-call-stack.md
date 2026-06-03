# Execution Context and Call Stack

## Learning Objectives
- Understand the anatomy of execution contexts in JavaScript
- Master how the call stack manages function execution
- Learn the phases of execution context creation
- Comprehend scope chain construction and variable resolution
- Understand how JavaScript code executes line by line

## Introduction

Every time JavaScript code runs, it operates within an **execution context**. Understanding execution contexts is fundamental to mastering JavaScript because they determine variable accessibility, the value of `this`, and how code executes sequentially through the **call stack**.

The JavaScript engine creates different types of execution contexts, manages them in a stack data structure, and follows specific phases during context creation and execution. This mechanism is what enables closures, scope, hoisting, and asynchronous behavior.

## Types of Execution Contexts

### 1. Global Execution Context (GEC)

The default context created when JavaScript code first runs. There's only one global execution context per program.

```javascript
// This code runs in the Global Execution Context
var globalVar = 'I am global';
const globalConst = 42;

function greet() {
  console.log('Hello from global scope');
}

console.log(window === this); // true in browsers (non-strict mode)
```

**Characteristics:**
- Creates the global object (`window` in browsers, `global` in Node.js)
- Sets `this` to the global object
- Creates variable and function declarations in global scope
- Only one exists per JavaScript runtime

### 2. Function Execution Context (FEC)

Created whenever a function is invoked. Each function call creates a new execution context.

```javascript
function outer() {
  const outerVar = 'outer';
  
  function inner() {
    const innerVar = 'inner';
    console.log(outerVar); // Accesses outer scope
  }
  
  inner(); // Creates new FEC for inner()
}

outer(); // Creates new FEC for outer()
```

**Characteristics:**
- Created on function invocation
- Has access to arguments object (non-arrow functions)
- Creates its own `this` binding
- Can access variables from outer scopes via scope chain

### 3. Eval Execution Context

Created when code runs inside `eval()` function (rarely used, generally discouraged).

```javascript
// Avoid using eval in production code
eval('var x = 10; console.log(x);'); // Creates eval execution context
```

## Execution Context Structure

Each execution context has three key components:

```
Execution Context {
  VariableEnvironment: {
    EnvironmentRecord: {...},
    outer: <reference to outer environment>
  },
  LexicalEnvironment: {
    EnvironmentRecord: {...},
    outer: <reference to outer environment>
  },
  ThisBinding: <value>
}
```

### 1. Variable Environment

Stores variable and function declarations created by `var` statements and function declarations.

```javascript
function example() {
  var x = 10;  // Stored in Variable Environment
  var y = 20;
  
  function nested() { // Stored in Variable Environment
    return x + y;
  }
}
```

### 2. Lexical Environment

Initially a copy of Variable Environment, but stores `let`, `const`, and function parameter bindings.

```javascript
function example(param) {  // param stored in Lexical Environment
  let x = 10;              // x stored in Lexical Environment
  const y = 20;            // y stored in Lexical Environment
  var z = 30;              // z stored in Variable Environment
}
```

### 3. This Binding

Determines the value of `this` within the execution context.

```javascript
const obj = {
  name: 'Object',
  regularFunc: function() {
    console.log(this.name); // 'this' bound to obj
  },
  arrowFunc: () => {
    console.log(this.name); // 'this' lexically inherited
  }
};

obj.regularFunc(); // "Object"
obj.arrowFunc();   // undefined (or global name)
```

## Phases of Execution Context Creation

Execution contexts are created in two distinct phases:

### Phase 1: Creation Phase (Memory Allocation)

During this phase, the JavaScript engine:

1. **Creates the Variable Environment and Lexical Environment**
2. **Scans for function declarations** - stores entire function in memory
3. **Scans for variable declarations** - allocates memory, sets to `undefined`
4. **Determines `this` binding**
5. **Creates the scope chain**

```javascript
console.log(x);        // undefined (not ReferenceError)
console.log(greet);    // [Function: greet]
console.log(y);        // ReferenceError: Cannot access 'y' before initialization

var x = 10;
let y = 20;

function greet() {
  return 'Hello';
}
```

**Memory State After Creation Phase:**

```
Global Execution Context (Creation Phase) {
  VariableEnvironment: {
    x: undefined,
    greet: <function object>
  },
  LexicalEnvironment: {
    y: <uninitialized> (TDZ)
  },
  ThisBinding: window/global
}
```

### Phase 2: Execution Phase

JavaScript engine executes code line by line:

1. **Assigns values to variables**
2. **Executes function calls**
3. **Creates new execution contexts for function invocations**

```javascript
// Creation Phase complete, now Execution Phase begins

var x = 10;            // x = undefined → x = 10
let y = 20;            // y leaves TDZ → y = 20

function greet() {     // Already stored in memory
  return 'Hello';
}

console.log(x);        // 10
console.log(y);        // 20
console.log(greet());  // "Hello"
```

**Memory State After Execution Phase:**

```
Global Execution Context (Execution Phase) {
  VariableEnvironment: {
    x: 10,
    greet: <function object>
  },
  LexicalEnvironment: {
    y: 20
  },
  ThisBinding: window/global
}
```

## The Call Stack

The **call stack** (execution stack) is a LIFO (Last In, First Out) data structure that manages execution contexts.

### Call Stack Operations

```javascript
function first() {
  console.log('Inside first');
  second();
  console.log('First complete');
}

function second() {
  console.log('Inside second');
  third();
  console.log('Second complete');
}

function third() {
  console.log('Inside third');
}

first();
```

**Call Stack Visualization:**

```
Step 1: Global Execution Context created
┌──────────────────────────┐
│   Global EC              │
└──────────────────────────┘

Step 2: first() called
┌──────────────────────────┐
│   first() EC             │
├──────────────────────────┤
│   Global EC              │
└──────────────────────────┘

Step 3: second() called from first()
┌──────────────────────────┐
│   second() EC            │
├──────────────────────────┤
│   first() EC             │
├──────────────────────────┤
│   Global EC              │
└──────────────────────────┘

Step 4: third() called from second()
┌──────────────────────────┐
│   third() EC             │
├──────────────────────────┤
│   second() EC            │
├──────────────────────────┤
│   first() EC             │
├──────────────────────────┤
│   Global EC              │
└──────────────────────────┘

Step 5: third() completes, popped off
┌──────────────────────────┐
│   second() EC            │
├──────────────────────────┤
│   first() EC             │
├──────────────────────────┤
│   Global EC              │
└──────────────────────────┘

Step 6: second() completes, popped off
┌──────────────────────────┐
│   first() EC             │
├──────────────────────────┤
│   Global EC              │
└──────────────────────────┘

Step 7: first() completes, popped off
┌──────────────────────────┐
│   Global EC              │
└──────────────────────────┘
```

**Output:**
```
Inside first
Inside second
Inside third
Second complete
First complete
```

### Stack Overflow

When the call stack exceeds its maximum size (varies by browser/environment):

```javascript
function recursive() {
  recursive(); // No base case!
}

recursive(); // RangeError: Maximum call stack size exceeded
```

Typical stack size limits:
- Chrome/V8: ~15,000-20,000 frames
- Firefox: ~50,000 frames
- Node.js: ~12,000 frames (configurable with --stack-size)

```javascript
// Better: Proper recursion with base case
function countdown(n) {
  if (n <= 0) {
    console.log('Done!');
    return;
  }
  console.log(n);
  countdown(n - 1);
}

countdown(5);
```

## Scope Chain

The scope chain determines variable access across nested execution contexts.

### Scope Chain Construction

```javascript
const global = 'global scope';

function outer() {
  const outerVar = 'outer scope';
  
  function inner() {
    const innerVar = 'inner scope';
    
    console.log(innerVar);  // Own scope
    console.log(outerVar);  // Outer scope via scope chain
    console.log(global);    // Global scope via scope chain
  }
  
  inner();
}

outer();
```

**Scope Chain Visualization:**

```
inner() Execution Context
│
├─ LexicalEnvironment: { innerVar: 'inner scope' }
│  outer reference ──┐
                     │
outer() Execution Context
│                    │
├─ LexicalEnvironment: { outerVar: 'outer scope' } ←┘
│  outer reference ──┐
                     │
Global Execution Context
│                    │
├─ LexicalEnvironment: { global: 'global scope' } ←┘
   outer reference: null
```

### Variable Resolution Process

When JavaScript encounters a variable reference:

1. **Check current execution context's Lexical Environment**
2. **If not found, check outer reference**
3. **Continue up the scope chain**
4. **If reached global scope and not found → ReferenceError**

```javascript
const a = 'global a';

function level1() {
  const b = 'level1 b';
  
  function level2() {
    const c = 'level2 c';
    const a = 'level2 a'; // Shadows global 'a'
    
    console.log(a); // 'level2 a' - found in own scope
    console.log(b); // 'level1 b' - found in outer scope
    console.log(c); // 'level2 c' - found in own scope
    console.log(d); // ReferenceError - not in scope chain
  }
  
  level2();
}

level1();
```

## Practical Examples

### Example 1: Closure Creation via Scope Chain

```javascript
function createCounter() {
  let count = 0; // Stored in createCounter's Lexical Environment
  
  return function increment() {
    count++;
    console.log(count);
  };
}

const counter = createCounter();

counter(); // 1
counter(); // 2
counter(); // 3

// increment() maintains reference to createCounter's Lexical Environment
// even after createCounter has completed execution
```

**Execution Context Flow:**

```
1. createCounter() called
   ┌─────────────────────────────┐
   │ createCounter EC            │
   │ LexicalEnv: { count: 0 }    │
   │ Returns: increment function │
   └─────────────────────────────┘

2. createCounter() completes and pops off stack
   BUT count variable remains in memory because
   increment function closure maintains reference

3. counter() called (invokes increment)
   ┌─────────────────────────────┐
   │ increment EC                │
   │ Scope chain → createCounter │
   │ LexicalEnv (outer)          │
   │ { count: 1 } ←─────────────┘
   └─────────────────────────────┘
```

### Example 2: `this` Binding in Different Contexts

```javascript
const user = {
  name: 'Alice',
  
  regularMethod: function() {
    console.log(this.name);
    
    function innerRegular() {
      console.log(this.name); // 'this' context lost!
    }
    innerRegular();
    
    const innerArrow = () => {
      console.log(this.name); // 'this' lexically inherited
    };
    innerArrow();
  },
  
  arrowMethod: () => {
    console.log(this.name); // 'this' from outer scope
  }
};

user.regularMethod();
// Alice
// undefined (or error in strict mode)
// Alice

user.arrowMethod();
// undefined (or global name)
```

**This Binding Analysis:**

```
user.regularMethod() Execution Context:
- ThisBinding: user object
- innerRegular(): new FEC, ThisBinding: global object
- innerArrow(): lexically inherits 'this' from regularMethod (user object)

user.arrowMethod() Execution Context:
- Arrow function, no own 'this'
- ThisBinding: lexically inherited from global scope
```

### Example 3: Variable Shadowing

```javascript
let x = 'global';

function outer() {
  let x = 'outer';
  
  function inner() {
    let x = 'inner';
    console.log(x); // 'inner'
  }
  
  inner();
  console.log(x); // 'outer'
}

outer();
console.log(x); // 'global'
```

Each execution context has its own `x` variable:

```
Global EC: { x: 'global' }
  ↓
outer() EC: { x: 'outer' }
  ↓
inner() EC: { x: 'inner' } ← Shadows outer scopes
```

### Example 4: Temporal Dead Zone (TDZ)

```javascript
function demonstrateTDZ() {
  console.log(varVariable);   // undefined (hoisted)
  // console.log(letVariable); // ReferenceError: TDZ
  
  var varVariable = 'var';
  let letVariable = 'let';
  
  console.log(varVariable);   // 'var'
  console.log(letVariable);   // 'let'
}

demonstrateTDZ();
```

**Creation Phase:**

```
demonstrateTDZ EC (Creation Phase) {
  VariableEnvironment: {
    varVariable: undefined  ← Initialized
  },
  LexicalEnvironment: {
    letVariable: <uninitialized> ← TDZ
  }
}
```

**Execution Phase:**

```
Line 2: varVariable = undefined (already initialized)
Line 3: letVariable still in TDZ → ReferenceError

Line 6: varVariable = 'var'
Line 7: letVariable exits TDZ, assigned 'let'
```

### Example 5: Complex Nested Execution

```javascript
const globalVar = 'GLOBAL';

function first(a) {
  const firstVar = 'FIRST';
  
  function second(b) {
    const secondVar = 'SECOND';
    
    function third(c) {
      console.log(globalVar); // Scope chain: third → second → first → global
      console.log(firstVar);  // Scope chain: third → second → first
      console.log(secondVar); // Scope chain: third → second
      console.log(a, b, c);   // Parameters from different contexts
    }
    
    third(3);
  }
  
  second(2);
}

first(1);
```

**Call Stack at Deepest Point:**

```
┌──────────────────────────────────┐
│ third(3) EC                      │
│ LexicalEnv: { c: 3 }             │
│ Scope chain →                    │
├──────────────────────────────────┤
│ second(2) EC                     │
│ LexicalEnv: { b: 2, secondVar }  │
│ Scope chain →                    │
├──────────────────────────────────┤
│ first(1) EC                      │
│ LexicalEnv: { a: 1, firstVar }   │
│ Scope chain →                    │
├──────────────────────────────────┤
│ Global EC                        │
│ LexicalEnv: { globalVar }        │
└──────────────────────────────────┘
```

## Common Misconceptions

### ❌ Misconception 1: Variables are hoisted with their values

```javascript
console.log(x); // undefined, not 10
var x = 10;
```

**✅ Reality:** Only the declaration is hoisted, not the initialization. During the creation phase, `x` is set to `undefined`.

### ❌ Misconception 2: The call stack is infinite

```javascript
function infinite() {
  infinite();
}
// infinite(); // Will crash with stack overflow
```

**✅ Reality:** The call stack has a finite size. Excessive recursion causes stack overflow errors.

### ❌ Misconception 3: `let` and `const` are not hoisted

```javascript
console.log(x); // ReferenceError
let x = 10;
```

**✅ Reality:** `let` and `const` ARE hoisted, but remain in the Temporal Dead Zone (uninitialized) until their declaration is executed.

### ❌ Misconception 4: Inner functions always have access to outer variables

```javascript
function outer() {
  var x = 10;
}

outer();

function unrelated() {
  console.log(x); // ReferenceError
}
```

**✅ Reality:** Scope chain is determined by lexical position (where code is written), not execution order. `unrelated()` cannot access `outer()`'s variables.

### ❌ Misconception 5: Global variables are always accessible

```javascript
var globalVar = 'global';

function test() {
  var globalVar = 'local'; // Shadows global
  console.log(globalVar);  // 'local'
}

test();
```

**✅ Reality:** Variable shadowing can make outer scope variables inaccessible. The local variable takes precedence.

## Performance Implications

### 1. Scope Chain Lookup Cost

Deeper scope chains mean slower variable resolution:

```javascript
// Slower: Deep scope chain
function level1() {
  const a = 1;
  function level2() {
    function level3() {
      function level4() {
        console.log(a); // Must traverse 3 outer references
      }
      level4();
    }
    level3();
  }
  level2();
}

// Faster: Shallow scope
function optimized() {
  const a = 1;
  console.log(a); // Direct access
}
```

**Optimization:** Cache frequently accessed outer scope variables in local scope:

```javascript
function outer() {
  const expensiveValue = computeExpensive();
  
  return function inner() {
    const cached = expensiveValue; // Cache in local scope
    // Use cached instead of expensiveValue repeatedly
    return cached * 2;
  };
}
```

### 2. Call Stack Depth

Deep call stacks consume more memory:

```javascript
// Recursive: Deep call stack
function recursiveSum(n, acc = 0) {
  if (n <= 0) return acc;
  return recursiveSum(n - 1, acc + n); // New execution context per call
}

// Iterative: Single execution context
function iterativeSum(n) {
  let sum = 0;
  for (let i = 1; i <= n; i++) {
    sum += i;
  }
  return sum;
}

console.time('recursive');
recursiveSum(10000); // May cause stack overflow
console.timeEnd('recursive');

console.time('iterative');
iterativeSum(10000); // Constant stack space
console.timeEnd('iterative');
```

### 3. Closure Memory Overhead

Closures maintain references to outer scope variables:

```javascript
// Memory leak potential
function createClosure() {
  const largeArray = new Array(1000000).fill('data');
  
  return function() {
    console.log(largeArray[0]); // Entire array kept in memory
  };
}

const closure = createClosure();
// largeArray remains in memory as long as closure exists

// Optimized: Only keep what you need
function createOptimizedClosure() {
  const largeArray = new Array(1000000).fill('data');
  const firstElement = largeArray[0]; // Extract only needed data
  
  return function() {
    console.log(firstElement); // largeArray can be garbage collected
  };
}
```

### 4. Avoid eval() and with()

Both create dynamic scopes that prevent optimizations:

```javascript
// Bad: Prevents optimization
function slow(code) {
  eval(code); // Creates new scope dynamically
}

// Good: Predictable scope chain
function fast() {
  const x = 10;
  return x * 2;
}
```

## Interview Questions

**Q1: Explain what happens step-by-step when this code executes:**

```javascript
var x = 10;

function foo() {
  console.log(x);
  var x = 20;
  console.log(x);
}

foo();
```

**A:** 

**Step 1: Global Execution Context Creation Phase**
- `x` declared and initialized to `undefined` in Variable Environment
- `foo` function object created and stored in Variable Environment

**Step 2: Global Execution Context Execution Phase**
- `x` assigned value `10`
- `foo()` invoked

**Step 3: foo() Execution Context Creation Phase**
- New execution context created and pushed to call stack
- `x` declared in foo's Variable Environment, initialized to `undefined` (shadows global x)

**Step 4: foo() Execution Context Execution Phase**
- Line 1: `console.log(x)` outputs `undefined` (local x, not global)
- Line 2: Local `x` assigned value `20`
- Line 3: `console.log(x)` outputs `20`
- foo() completes, execution context popped from stack

**Output:**
```
undefined
20
```

**Q2: What is the difference between the Variable Environment and Lexical Environment?**

**A:** 

Both are environment records within an execution context, but they serve different purposes:

**Variable Environment:**
- Stores bindings created by `var` statements and function declarations
- Created during the creation phase and never changes
- Used for variable hoisting

**Lexical Environment:**
- Initially a copy of the Variable Environment
- Stores bindings for `let`, `const`, and function parameters
- Can change during execution (e.g., with `catch` blocks)
- Used for scope resolution and `this` binding

```javascript
function example(param) {        // param → Lexical Environment
  var varVariable = 'var';       // → Variable Environment
  let letVariable = 'let';       // → Lexical Environment
  const constVariable = 'const'; // → Lexical Environment
  
  function nested() {}           // → Variable Environment
}
```

**Q3: How does the JavaScript engine resolve variable access across nested scopes?**

**A:** 

The engine uses the **scope chain** for variable resolution:

1. **Check current Lexical Environment** for the variable
2. **If not found**, follow the `outer` reference to parent Lexical Environment
3. **Repeat** until variable is found or global scope is reached
4. **If not found in global scope**, throw `ReferenceError`

```javascript
const global = 'GLOBAL';

function outer() {
  const outerVar = 'OUTER';
  
  function inner() {
    const innerVar = 'INNER';
    console.log(innerVar);  // Step 1: Found in inner's LE
    console.log(outerVar);  // Step 1: Not found → Step 2: Found in outer's LE
    console.log(global);    // Steps 1-2: Not found → Step 3: Found in global LE
    console.log(missing);   // Steps 1-3: Not found → ReferenceError
  }
  
  inner();
}
```

**Q4: Why does this code produce different outputs?**

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// Output: 3, 3, 3

for (let j = 0; j < 3; j++) {
  setTimeout(() => console.log(j), 0);
}
// Output: 0, 1, 2
```

**A:** 

**`var` version:**
- `var` is function-scoped (or global if no function)
- Only ONE `i` variable exists in the outer scope
- All three timeouts close over the SAME `i` variable
- By the time timeouts execute, the loop has completed and `i = 3`
- All timeouts log the final value: `3`

**`let` version:**
- `let` is block-scoped
- Each loop iteration creates a NEW block scope
- Each timeout closes over a DIFFERENT `j` variable
- Three separate `j` variables with values 0, 1, 2
- Each timeout logs its own `j` value

**Scope visualization:**

```
var version:
Global Scope { i: 3 } ← All timeouts reference this

let version:
Block Scope 0 { j: 0 } ← Timeout 1 references this
Block Scope 1 { j: 1 } ← Timeout 2 references this
Block Scope 2 { j: 2 } ← Timeout 3 references this
```

**Q5: What happens when you exceed the call stack limit?**

**A:** 

When the call stack exceeds its maximum depth, the JavaScript engine throws a **RangeError: Maximum call stack size exceeded**.

**Common causes:**
1. **Infinite recursion** (no base case)
2. **Deep recursion** (base case reached too slowly)
3. **Mutual recursion** without proper termination

```javascript
// Infinite recursion
function infinite() {
  infinite(); // RangeError
}

// Solution: Add base case
function safe(n) {
  if (n <= 0) return 0;
  return n + safe(n - 1);
}
```

**Stack limits vary by environment:**
- Chrome/V8: ~15,000 frames
- Firefox: ~50,000 frames
- Node.js: ~12,000 frames (configurable)

**Solutions:**
- Use iteration instead of recursion
- Implement tail call optimization (limited support)
- Use trampolining for recursive algorithms
- Batch processing for large datasets

**Q6: Explain the Temporal Dead Zone (TDZ) and why it exists.**

**A:** 

The **Temporal Dead Zone** is the period between entering a scope and the actual declaration of a `let` or `const` variable, during which the variable cannot be accessed.

**Why TDZ exists:**
1. **Catch errors early**: Accessing variables before declaration is likely a mistake
2. **Const semantics**: Ensures `const` variables are initialized at declaration
3. **Predictable behavior**: Makes scope boundaries clear

```javascript
function demonstrateTDZ() {
  // TDZ starts
  console.log(x); // ReferenceError: Cannot access 'x' before initialization
  console.log(y); // undefined (var is hoisted and initialized)
  
  let x = 10;     // TDZ ends for x
  var y = 20;
}
```

**Creation vs Execution phases:**

```
Creation Phase:
- x: <uninitialized> (TDZ)
- y: undefined (initialized)

Execution Phase:
- Attempt to access x → ReferenceError (still in TDZ)
- Access y → undefined (already initialized)
- x = 10 → TDZ ends
```

**Q7: How do closures work in terms of execution contexts and scope chains?**

**A:** 

A **closure** is created when an inner function maintains a reference to its outer function's Lexical Environment, even after the outer function has completed execution.

```javascript
function createCounter() {
  let count = 0;
  
  return function increment() {
    count++;
    return count;
  };
}

const counter = createCounter();
console.log(counter()); // 1
console.log(counter()); // 2
```

**Step-by-step:**

1. **`createCounter()` called:**
   - Execution context created
   - Lexical Environment: `{ count: 0 }`
   - Returns `increment` function

2. **`createCounter()` completes:**
   - Execution context popped from stack
   - BUT: Lexical Environment remains in memory
   - Why? `increment` function maintains reference to it

3. **`counter()` called (invokes `increment`):**
   - New execution context created for `increment`
   - Scope chain: `increment` → `createCounter` Lexical Environment
   - Accesses and modifies `count`
   - Returns updated value

**Memory structure:**

```
counter function {
  [[Scopes]]: [
    0: Closure (createCounter) {
      count: 2  ← Persisted Lexical Environment
    },
    1: Global
  ]
}
```

**Q8: What is the difference between the call stack and the execution context?**

**A:** 

**Call Stack:**
- Data structure (stack) that manages execution contexts
- LIFO (Last In, First Out) order
- Physical representation of program execution flow
- Single stack per JavaScript thread

**Execution Context:**
- Abstract container for code execution
- Contains Variable Environment, Lexical Environment, and ThisBinding
- Created for global code, functions, and eval
- Pushed to and popped from the call stack

**Analogy:**
- Call stack = Book shelf
- Execution contexts = Individual books
- Books are added to the top and removed from the top

```javascript
function first() {
  second();
}

function second() {
  third();
}

function third() {
  console.log('At deepest point');
}

first();
```

**Relationship:**

```
Call Stack (structure)     Execution Contexts (contents)
┌──────────────────┐       
│   third() EC     │ ←───── { LE: {...}, VE: {...}, this: ... }
├──────────────────┤       
│   second() EC    │ ←───── { LE: {...}, VE: {...}, this: ... }
├──────────────────┤       
│   first() EC     │ ←───── { LE: {...}, VE: {...}, this: ... }
├──────────────────┤       
│   Global EC      │ ←───── { LE: {...}, VE: {...}, this: window }
└──────────────────┘
```

## Key Takeaways

- **Execution contexts** are abstract containers that hold information about code execution (variables, functions, `this`, scope chain)
- Three types: **Global Execution Context**, **Function Execution Context**, **Eval Execution Context**
- Each context has **Variable Environment** (var, functions), **Lexical Environment** (let, const, parameters), and **ThisBinding**
- Contexts are created in two phases: **Creation Phase** (memory allocation, hoisting) and **Execution Phase** (code execution, value assignment)
- The **call stack** manages execution contexts using LIFO ordering
- **Scope chain** enables variable access across nested scopes via outer environment references
- **Closures** work by maintaining references to outer Lexical Environments even after outer functions complete
- **Temporal Dead Zone** prevents `let`/`const` access before declaration, unlike `var` which is initialized to `undefined`
- Deep scope chains and excessive recursion have performance costs
- Understanding execution contexts is fundamental to mastering closures, hoisting, `this`, and scope

## Resources

- [ECMAScript Specification - Execution Contexts](https://tc39.es/ecma262/#sec-execution-contexts)
- [ECMAScript Specification - Lexical Environments](https://tc39.es/ecma262/#sec-lexical-environments)
- [MDN - Closures](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures)
- [JavaScript.info - Execution Context](https://javascript.info/closure)
- [V8 Blog - Understanding V8's Bytecode](https://v8.dev/blog/ignition-interpreter)
- [You Don't Know JS - Scope & Closures](https://github.com/getify/You-Dont-Know-JS/tree/2nd-ed/scope-closures)
- [ECMAScript Specification - Environment Records](https://tc39.es/ecma262/#sec-environment-records)
