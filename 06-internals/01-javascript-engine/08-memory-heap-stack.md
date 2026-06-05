# Memory: Heap and Stack

## The Idea

**In plain English:** When JavaScript runs your code, it needs two places to store information: the stack (a neat, ordered pile for simple values like numbers and true/false), and the heap (a big open storage room for complex things like objects and lists). Knowing which one your data lives in explains why some code behaves in surprising ways.

**Real-world analogy:** Think of a busy restaurant kitchen. The chef has a small whiteboard right next to them where they jot down simple notes like table numbers and prices — that is the stack. Meanwhile, the back storage room holds large, complex things like full ingredient trays that multiple chefs can reach into and change — that is the heap.

- The whiteboard = the stack (small, fast, holds simple values directly)
- The storage room = the heap (large, slower, holds complex objects)
- The note pointing to a tray's shelf location = a reference (a variable that points to an object in the heap)

---

## Learning Objectives
- Understand how JavaScript manages memory allocation
- Master the difference between stack and heap memory
- Learn how primitives and references are stored
- Comprehend memory allocation patterns and performance implications
- Understand stack frames and heap object allocation

## Introduction

JavaScript engines manage two primary memory regions: the **stack** and the **heap**. Understanding how these work is fundamental to writing efficient JavaScript, avoiding memory leaks, and comprehending reference behavior.

The **call stack** stores primitive values and references in a LIFO (Last In, First Out) structure, while the **heap** stores objects and functions in an unstructured memory pool. This distinction affects performance, garbage collection, and how data is passed between functions.

## Memory Architecture

```
┌─────────────────────────────────────────┐
│         JavaScript Memory               │
├─────────────────────┬───────────────────┤
│      STACK          │       HEAP        │
│  (Fixed-size data)  │  (Dynamic data)   │
│                     │                   │
│  - Primitives       │  - Objects        │
│  - References       │  - Arrays         │
│  - Function frames  │  - Functions      │
│  - Local variables  │  - Closures       │
│                     │                   │
│  Fast allocation    │  Slower allocation│
│  Automatic cleanup  │  GC required      │
│  Limited size       │  Larger size      │
│  LIFO structure     │  Unstructured     │
└─────────────────────┴───────────────────┘
```

## The Call Stack

The **call stack** is a contiguous region of memory that stores:
- Primitive values (numbers, strings, booleans, null, undefined, symbols, bigints)
- References (pointers to heap objects)
- Function call frames (execution contexts)

### Stack Allocation

```javascript
// Stack allocation example
function example() {
  let num = 42;           // Primitive → stored on stack
  let str = 'hello';      // Primitive → stored on stack
  let bool = true;        // Primitive → stored on stack
  let nothing = null;     // Primitive → stored on stack
  let undef = undefined;  // Primitive → stored on stack
}

example();
```

**Stack memory layout:**

```
┌─────────────────────────┐
│  example() frame        │
│  ────────────────       │
│  undef: undefined       │ ← 8 bytes
│  nothing: null          │ ← 8 bytes
│  bool: true             │ ← 8 bytes
│  str: 'hello'           │ ← 8 bytes (reference to heap string)
│  num: 42                │ ← 8 bytes
│  return address         │
│  ────────────────       │
│  Global frame           │
└─────────────────────────┘
```

**Note:** Small strings may be stored inline (stack) or interned, but conceptually they behave as primitives.

### Stack Frame Structure

Each function call creates a **stack frame** (activation record):

```javascript
function first(a) {
  const x = 10;
  second(x);
}

function second(b) {
  const y = 20;
  third(y);
}

function third(c) {
  const z = 30;
  console.log(a + b + c); // ReferenceError: a, b not accessible
}

first(5);
```

**Stack visualization when third() executes:**

```
┌────────────────────────────┐ ← Stack Pointer (SP)
│  third() frame             │
│  ──────────────            │
│  c: 20 (parameter)         │
│  z: 30 (local var)         │
│  return address            │
├────────────────────────────┤
│  second() frame            │
│  ──────────────            │
│  b: 10 (parameter)         │
│  y: 20 (local var)         │
│  return address            │
├────────────────────────────┤
│  first() frame             │
│  ──────────────            │
│  a: 5 (parameter)          │
│  x: 10 (local var)         │
│  return address            │
├────────────────────────────┤
│  Global frame              │
└────────────────────────────┘
```

When third() completes:
1. third() frame popped off (stack pointer decremented)
2. Execution returns to second()
3. second() frame popped off
4. Execution returns to first()
5. first() frame popped off

### Stack Memory Characteristics

**Advantages:**
- **Fast allocation/deallocation** - just increment/decrement stack pointer
- **Automatic cleanup** - memory freed when function returns
- **Cache-friendly** - contiguous memory access
- **Predictable size** - each frame has fixed size

**Limitations:**
- **Fixed size** - stack overflow if too deep
- **LIFO only** - can't arbitrarily free memory
- **Short-lived data** - data dies with function return

```javascript
// Stack overflow example
function recursive(n) {
  console.log(n);
  recursive(n + 1); // No base case!
}

recursive(1); // RangeError: Maximum call stack size exceeded
```

Typical stack sizes:
- Chrome/V8: ~1MB (about 15,000-20,000 frames)
- Firefox: ~2MB
- Node.js: ~1MB (configurable with --stack-size)

## The Heap

The **heap** is an unstructured memory region for dynamic allocation:
- Objects
- Arrays
- Functions
- Closures
- Any non-primitive data

### Heap Allocation

```javascript
// Heap allocation example
function example() {
  let obj = { name: 'Alice' };     // Object → heap
  let arr = [1, 2, 3];             // Array → heap
  let func = () => {};             // Function → heap
  let date = new Date();           // Date object → heap
}

example();
```

**Memory layout:**

```
STACK                           HEAP
┌─────────────────────┐         ┌──────────────────────────┐
│  example() frame    │         │                          │
│  ─────────────      │         │  {name: 'Alice'}         │
│  date: ────────────────────►  │  [1, 2, 3]               │
│  func: ────────────────────►  │  function() {}           │
│  arr:  ────────────────────►  │  Date {2026-05-27...}    │
│  obj:  ────────────────────►  │                          │
└─────────────────────┘         └──────────────────────────┘
   (references)                        (actual objects)
```

### Object Structure in Heap

Objects in V8 heap have hidden metadata:

```
Object in Heap:
┌──────────────────────────┐
│  Hidden Class (Map)      │ ← Points to structure/shape
├──────────────────────────┤
│  Properties (inline)     │ ← Fast access properties
│  name: 'Alice'           │
│  age: 30                 │
├──────────────────────────┤
│  Elements (backing store)│ ← Array-indexed properties
│  [0]: 'first'            │
│  [1]: 'second'           │
├──────────────────────────┤
│  Extra properties        │ ← Overflow properties
│  (slow dictionary mode)  │
└──────────────────────────┘
```

### Heap Memory Characteristics

**Advantages:**
- **Dynamic sizing** - grows as needed (until system limit)
- **Long-lived data** - persists beyond function scope
- **Flexible allocation** - allocate/free in any order
- **Shared access** - multiple references to same object

**Limitations:**
- **Slower allocation** - requires finding free space
- **Garbage collection overhead** - must track and reclaim unused memory
- **Fragmentation** - can lead to inefficient memory use
- **Cache-unfriendly** - objects scattered in memory

## Primitives vs References

### Primitives (Stored by Value)

Primitives are **immutable** and stored directly on the stack:

```javascript
let a = 10;
let b = a;  // COPY of value

b = 20;     // Changes b, not a

console.log(a); // 10
console.log(b); // 20
```

**Memory:**

```
STACK
┌─────────────┐
│  a: 10      │ ← Independent values
│  b: 20      │
└─────────────┘
```

Primitive types:
- Number: `42`, `3.14`, `NaN`, `Infinity`
- String: `'hello'`, `"world"`
- Boolean: `true`, `false`
- Null: `null`
- Undefined: `undefined`
- Symbol: `Symbol('desc')`
- BigInt: `123n`

### References (Stored by Reference)

Objects are **mutable** and stored on the heap. Variables hold references (pointers):

```javascript
let obj1 = { value: 10 };
let obj2 = obj1;  // COPY of reference (both point to same object)

obj2.value = 20;  // Mutates shared object

console.log(obj1.value); // 20 (same object!)
console.log(obj2.value); // 20
```

**Memory:**

```
STACK                    HEAP
┌─────────────┐         ┌──────────────┐
│  obj1: ──────────────► │ {value: 20}  │
│  obj2: ──────────────► │              │
└─────────────┘         └──────────────┘
   (references point to same object)
```

### Comparison Behavior

```javascript
// Primitives: compare values
let a = 10;
let b = 10;
console.log(a === b); // true (same value)

// Objects: compare references
let obj1 = { value: 10 };
let obj2 = { value: 10 };
console.log(obj1 === obj2); // false (different objects)

let obj3 = obj1;
console.log(obj1 === obj3); // true (same reference)
```

### Passing to Functions

```javascript
// Primitives: pass by value
function modifyPrimitive(x) {
  x = 100; // Modifies local copy
}

let num = 10;
modifyPrimitive(num);
console.log(num); // 10 (unchanged)

// Objects: pass by reference
function modifyObject(obj) {
  obj.value = 100; // Mutates original object
}

let myObj = { value: 10 };
modifyObject(myObj);
console.log(myObj.value); // 100 (changed!)

// Reassignment doesn't affect original reference
function reassignObject(obj) {
  obj = { value: 200 }; // Reassigns local reference
}

let myObj2 = { value: 10 };
reassignObject(myObj2);
console.log(myObj2.value); // 10 (unchanged)
```

**Memory visualization:**

```
Passing primitive:
STACK (modifyPrimitive)       STACK (caller)
┌──────────────┐             ┌──────────────┐
│  x: 100      │             │  num: 10     │ ← Independent
└──────────────┘             └──────────────┘

Passing reference:
STACK (modifyObject)         STACK (caller)
┌──────────────┐             ┌──────────────┐
│  obj: ───────────┐         │  myObj: ─────────┐
└──────────────┘  │          └──────────────┘   │
                  │                              │
                  └──────► HEAP                  │
                           ┌──────────────┐     │
                           │ {value: 100} │ ◄───┘
                           └──────────────┘
                      (both references point to same object)
```

## Deep vs Shallow Copy

### Shallow Copy

Copies top-level properties, but nested objects are still shared:

```javascript
const original = {
  name: 'Alice',
  address: {
    city: 'NYC'
  }
};

// Shallow copy methods
const copy1 = { ...original };
const copy2 = Object.assign({}, original);

copy1.name = 'Bob';           // Independent primitive
copy1.address.city = 'LA';    // Shared nested object!

console.log(original.name);         // 'Alice'
console.log(original.address.city); // 'LA' (mutated!)
```

**Memory:**

```
STACK                         HEAP
┌──────────────┐             ┌────────────────────┐
│ original: ────────────────► │ {                  │
│ copy1: ───────────────────► │   name: 'Bob'      │
└──────────────┘              │   address: ───────┐│
                              └────────────────────┘│
                                                    │
                              ┌────────────────────┘
                              ▼
                              ┌────────────────────┐
                              │ { city: 'LA' }     │
                              └────────────────────┘
                                 (shared object)
```

### Deep Copy

Copies all nested objects, creating completely independent structures:

```javascript
const original = {
  name: 'Alice',
  address: {
    city: 'NYC',
    coords: { lat: 40.7, lng: -74.0 }
  }
};

// Deep copy methods
const deepCopy1 = JSON.parse(JSON.stringify(original));
const deepCopy2 = structuredClone(original); // Modern API

deepCopy1.address.city = 'LA';
deepCopy1.address.coords.lat = 34.05;

console.log(original.address.city);      // 'NYC' (unchanged)
console.log(original.address.coords.lat); // 40.7 (unchanged)
```

**Memory:**

```
STACK                         HEAP
┌──────────────┐             ┌────────────────────┐
│ original: ────────────────► │ {name: 'Alice',    │
└──────────────┘              │   address: ───┐    │
                              └────────────────┼────┘
                                               │
┌──────────────┐             ┌────────────────▼────┐
│ deepCopy: ─────────────────► │ {name: 'Alice',    │
└──────────────┘              │   address: ───┐    │
                              └────────────────┼────┘
                                               │
                              ┌────────────────▼────┐
                              │ {city: 'NYC',       │
                              │   coords: ───┐      │
                              └────────────────┼─────┘
                                               │
                       (completely independent objects)
```

### Deep Copy Limitations

**JSON.parse(JSON.stringify()) issues:**
- Loses functions
- Loses undefined properties
- Loses Symbol keys
- Loses Date objects (converts to string)
- Can't handle circular references

```javascript
const obj = {
  func: () => {},           // Lost
  undef: undefined,         // Lost
  date: new Date(),         // Becomes string
  [Symbol('key')]: 'value'  // Lost
};

const copy = JSON.parse(JSON.stringify(obj));
console.log(copy);
// { date: "2026-05-27T12:00:00.000Z" }
```

**structuredClone() is better:**
- Handles Date, RegExp, Map, Set
- Handles circular references
- Preserves undefined
- Still loses functions and Symbols

```javascript
const obj = {
  date: new Date(),
  map: new Map([['key', 'value']]),
  undef: undefined
};

const copy = structuredClone(obj);
console.log(copy.date instanceof Date); // true
console.log(copy.map instanceof Map);   // true
```

## Memory Allocation Patterns

### Pattern 1: Object Literals

```javascript
// Allocates on heap
const user = {
  name: 'Alice',
  age: 30
};
```

**V8 optimization:** Objects with same shape share "hidden class" (Map):

```javascript
// These share the same hidden class
const user1 = { name: 'Alice', age: 30 };
const user2 = { name: 'Bob', age: 25 };

// Different shape = different hidden class (slower)
const user3 = { age: 35, name: 'Charlie' }; // Different property order
```

### Pattern 2: Array Allocation

```javascript
// Allocates backing store on heap
const numbers = [1, 2, 3, 4, 5];
```

**V8 optimization:** Arrays have two storage modes:

1. **Fast (packed elements):** Contiguous memory
2. **Slow (dictionary mode):** Hash table

```javascript
// Fast mode
const fast = [1, 2, 3];

// Switches to slow mode (holes in array)
const slow = [1, 2, 3];
slow[1000] = 4; // Creates sparse array
```

### Pattern 3: Constructor Functions

```javascript
function Person(name, age) {
  this.name = name; // Properties on heap
  this.age = age;
}

Person.prototype.greet = function() { // Shared on heap
  return `Hello, I'm ${this.name}`;
};

const alice = new Person('Alice', 30);
const bob = new Person('Bob', 25);

// alice and bob share the same prototype object
// (greet function allocated once, shared by all instances)
```

### Pattern 4: Closures

Closures maintain references to outer scope variables, keeping them in memory:

```javascript
function createCounter() {
  let count = 0; // Allocated on stack initially
  
  return function increment() {
    count++; // Closure keeps 'count' alive on heap
    return count;
  };
}

const counter = createCounter();
// 'count' variable moved to heap because closure references it
```

**Memory:**

```
HEAP
┌──────────────────────────┐
│  Closure Scope Object    │
│  { count: 0 }            │ ← Persists after createCounter returns
└──────────────────────────┘
           ▲
           │
┌──────────┴───────────────┐
│  increment function      │
│  [[Scopes]]: [           │
│    Closure: ─────────────┘
│    Global                │
│  ]                       │
└──────────────────────────┘
```

## Stack vs Heap Performance

### Stack: Faster

```javascript
function stackHeavy() {
  let a = 1;
  let b = 2;
  let c = 3;
  let d = 4;
  let e = 5;
  // Fast: all primitive allocations on stack
  return a + b + c + d + e;
}
```

**Why faster?**
- Simple pointer increment for allocation
- Automatic deallocation (pop frame)
- Cache-friendly (contiguous memory)

### Heap: Slower

```javascript
function heapHeavy() {
  let obj1 = { value: 1 };
  let obj2 = { value: 2 };
  let obj3 = { value: 3 };
  let obj4 = { value: 4 };
  let obj5 = { value: 5 };
  // Slower: heap allocations + GC overhead
  return obj1.value + obj2.value + obj3.value + obj4.value + obj5.value;
}
```

**Why slower?**
- Must find free space in heap
- Requires garbage collection
- Cache-unfriendly (scattered memory)
- Object property access overhead

### Benchmark Example

```javascript
console.time('stack');
for (let i = 0; i < 1000000; i++) {
  let x = 1;
  let y = 2;
  let z = x + y;
}
console.timeEnd('stack'); // ~2ms

console.time('heap');
for (let i = 0; i < 1000000; i++) {
  let obj = { x: 1, y: 2 };
  let z = obj.x + obj.y;
}
console.timeEnd('heap'); // ~50ms (25x slower!)
```

## Memory Leak Patterns

### Leak 1: Unintended Global Variables

```javascript
function leak() {
  // Forgot 'let' - creates global variable!
  leakedVar = 'Stays in memory forever';
}

leak();
// leakedVar persists globally, never garbage collected
```

**Fix:**
```javascript
'use strict';

function noLeak() {
  leakedVar = 'Error!'; // ReferenceError in strict mode
}
```

### Leak 2: Forgotten Timers

```javascript
function setupTimer() {
  const data = new Array(1000000); // Large array
  
  setInterval(() => {
    console.log(data[0]); // Closure keeps 'data' alive forever
  }, 1000);
  
  // Timer never cleared, 'data' never garbage collected
}

setupTimer();
```

**Fix:**
```javascript
function setupTimer() {
  const data = new Array(1000000);
  
  const timerId = setInterval(() => {
    console.log(data[0]);
  }, 1000);
  
  // Clear when no longer needed
  setTimeout(() => {
    clearInterval(timerId);
  }, 10000);
}
```

### Leak 3: Detached DOM Nodes

```javascript
let detachedElements = [];

function createElements() {
  const container = document.getElementById('container');
  const element = document.createElement('div');
  container.appendChild(element);
  
  // Element removed from DOM but still referenced
  container.removeChild(element);
  detachedElements.push(element); // Keeps in memory!
}
```

**Fix:**
```javascript
// Remove reference when no longer needed
detachedElements = [];
```

### Leak 4: Closure Capturing Large Scope

```javascript
function outer() {
  const largeData = new Array(1000000).fill('data');
  const smallData = 'small';
  
  return function inner() {
    console.log(smallData); // Only uses smallData
    // But entire scope (including largeData) is captured!
  };
}

const fn = outer();
// largeData kept in memory even though unused
```

**Fix:**
```javascript
function outer() {
  const largeData = new Array(1000000).fill('data');
  const smallData = 'small';
  
  // Extract only what's needed
  const extracted = smallData;
  
  return function inner() {
    console.log(extracted);
    // largeData can be garbage collected
  };
}
```

## Common Misconceptions

### ❌ Misconception 1: Strings are always primitives on stack

```javascript
let str = 'hello';
```

**✅ Reality:** Small strings may be stored inline or interned, but large strings are allocated on the heap. Strings are immutable primitives semantically, but implementation varies.

### ❌ Misconception 2: Primitives are never on heap

**✅ Reality:** Boxed primitives (wrapper objects) are on heap:

```javascript
let num = 42;                // Stack
let numObj = new Number(42); // Heap

console.log(typeof num);     // 'number'
console.log(typeof numObj);  // 'object'
```

### ❌ Misconception 3: All objects are slow

**✅ Reality:** V8 optimizes objects with stable shapes using hidden classes. Consistently shaped objects perform well.

```javascript
// Fast: consistent shape
class Point {
  constructor(x, y) {
    this.x = x; // Always has x, y in same order
    this.y = y;
  }
}

// Slow: dynamic shape
function createPoint(x, y, z) {
  const obj = { x, y };
  if (z) obj.z = z; // Inconsistent shape
  return obj;
}
```

### ❌ Misconception 4: Copying objects is always expensive

**✅ Reality:** Shallow copies are fast (just copy references). Deep copies of nested structures are expensive.

### ❌ Misconception 5: Stack memory is unlimited

**✅ Reality:** Stack size is limited (~1MB). Deep recursion causes stack overflow:

```javascript
function infinite() {
  infinite();
}

infinite(); // RangeError: Maximum call stack size exceeded
```

## Performance Implications

### 1. Prefer Primitives When Possible

```javascript
// Slower: object allocation
function withObject(iterations) {
  for (let i = 0; i < iterations; i++) {
    const point = { x: i, y: i * 2 };
    const distance = Math.sqrt(point.x ** 2 + point.y ** 2);
  }
}

// Faster: primitive values
function withPrimitives(iterations) {
  for (let i = 0; i < iterations; i++) {
    const x = i;
    const y = i * 2;
    const distance = Math.sqrt(x ** 2 + y ** 2);
  }
}

// Benchmark
console.time('object');
withObject(1000000);
console.timeEnd('object'); // ~120ms

console.time('primitives');
withPrimitives(1000000);
console.timeEnd('primitives'); // ~30ms (4x faster)
```

### 2. Reuse Objects Instead of Creating New Ones

```javascript
// Bad: creates many temporary objects
function badLoop(iterations) {
  for (let i = 0; i < iterations; i++) {
    const temp = { value: i };
    processObject(temp);
  }
}

// Good: reuse single object
function goodLoop(iterations) {
  const reusable = { value: 0 };
  for (let i = 0; i < iterations; i++) {
    reusable.value = i;
    processObject(reusable);
  }
}
```

### 3. Avoid Unnecessary Deep Copies

```javascript
// Bad: expensive deep copy every time
function updateUser(user, changes) {
  const copy = structuredClone(user);
  Object.assign(copy, changes);
  return copy;
}

// Good: shallow copy sufficient for top-level changes
function updateUser(user, changes) {
  return { ...user, ...changes };
}
```

### 4. Beware of Closures in Loops

```javascript
// Bad: creates N closures, each capturing entire scope
function createHandlers(n) {
  const handlers = [];
  for (let i = 0; i < n; i++) {
    handlers.push(() => {
      console.log(i); // Each closure captures loop scope
    });
  }
  return handlers;
}

// Better: extract needed data only
function createHandlers(n) {
  const handlers = [];
  for (let i = 0; i < n; i++) {
    const value = i; // Capture only needed value
    handlers.push(() => {
      console.log(value);
    });
  }
  return handlers;
}
```

## Interview Questions

**Q1: What is the difference between stack and heap memory?**

**A:**

**Stack:**
- Stores primitives and references
- LIFO (Last In, First Out) structure
- Fast allocation/deallocation (pointer increment/decrement)
- Automatic cleanup when function returns
- Limited size (~1MB)
- Used for function call frames and local variables

**Heap:**
- Stores objects, arrays, functions
- Unstructured memory pool
- Slower allocation (must find free space)
- Requires garbage collection for cleanup
- Much larger size
- Used for dynamic data with flexible lifetime

**Example:**
```javascript
function example() {
  let num = 42;            // Stack
  let obj = { value: 42 }; // obj reference on stack, object on heap
}
```

**Q2: Explain what happens in memory when you assign an object to another variable.**

**A:**

When you assign an object to another variable, you copy the **reference** (pointer), not the object itself. Both variables point to the same object in heap memory.

```javascript
let obj1 = { value: 10 };
let obj2 = obj1; // Copy reference, not object

obj2.value = 20;

console.log(obj1.value); // 20 (same object!)
```

**Memory:**
```
STACK                    HEAP
┌─────────────┐         ┌──────────────┐
│  obj1: ──────────────► │ {value: 20}  │
│  obj2: ──────────────► │              │
└─────────────┘         └──────────────┘
```

**This is different from primitives:**
```javascript
let a = 10;
let b = a; // Copy value

b = 20;

console.log(a); // 10 (independent values)
```

**Memory:**
```
STACK
┌─────────────┐
│  a: 10      │
│  b: 20      │
└─────────────┘
```

**Q3: What is a stack overflow and when does it occur?**

**A:**

A **stack overflow** occurs when the call stack exceeds its maximum size, typically from excessive recursion without a proper base case.

```javascript
function recursive(n) {
  return recursive(n + 1); // No base case!
}

recursive(0); // RangeError: Maximum call stack size exceeded
```

**Why it happens:**
Each function call adds a frame to the stack. Without returning, frames accumulate until stack memory is exhausted.

**Typical limits:**
- Chrome/V8: ~15,000-20,000 frames
- Firefox: ~50,000 frames
- Node.js: ~12,000 frames

**Prevention:**
1. Add proper base case
2. Use iteration instead of recursion
3. Implement tail recursion (limited support)
4. Use trampolining

```javascript
// Fixed: proper base case
function factorial(n) {
  if (n <= 1) return 1; // Base case
  return n * factorial(n - 1);
}

// Better: iterative
function factorialIter(n) {
  let result = 1;
  for (let i = 2; i <= n; i++) {
    result *= i;
  }
  return result;
}
```

**Q4: How do closures affect memory allocation?**

**A:**

Closures keep outer scope variables alive in heap memory, even after the outer function returns.

```javascript
function createCounter() {
  let count = 0; // Initially on stack
  
  return function increment() {
    count++; // Closure keeps 'count' alive
    return count;
  };
}

const counter = createCounter();
// 'count' moved to heap, persists as long as 'counter' exists
```

**Memory:**
```
HEAP
┌──────────────────────────┐
│  Closure Scope Object    │
│  { count: 1 }            │ ← Kept alive by closure
└──────────────────────────┘
           ▲
           │
┌──────────┴───────────────┐
│  increment function      │
│  [[Scopes]]: [Closure]   │
└──────────────────────────┘
```

**Memory implications:**
- Closed-over variables persist until closure is garbage collected
- Can cause memory leaks if closures reference large data unnecessarily
- Multiple closures from same function share the same scope object

**Optimization:**
```javascript
// Bad: closure captures large unnecessary data
function createHandler() {
  const largeArray = new Array(1000000);
  const smallValue = largeArray[0];
  
  return function() {
    console.log(smallValue);
    // Entire largeArray kept in memory!
  };
}

// Good: extract only what's needed
function createHandler() {
  const largeArray = new Array(1000000);
  const extracted = largeArray[0]; // Copy out
  // largeArray can be GC'd
  
  return function() {
    console.log(extracted);
  };
}
```

**Q5: What's the difference between shallow and deep copy?**

**A:**

**Shallow Copy:**
- Copies top-level properties
- Nested objects are copied by reference (shared)

```javascript
const original = {
  name: 'Alice',
  address: { city: 'NYC' }
};

const shallow = { ...original };

shallow.name = 'Bob';          // Independent
shallow.address.city = 'LA';   // Shared!

console.log(original.address.city); // 'LA' (mutated)
```

**Deep Copy:**
- Recursively copies all nested objects
- Completely independent structures

```javascript
const deep = structuredClone(original);

deep.address.city = 'SF';

console.log(original.address.city); // 'NYC' (unchanged)
```

**Methods:**

| Method | Type | Limitations |
|--------|------|-------------|
| `{...obj}` | Shallow | Nested objects shared |
| `Object.assign({}, obj)` | Shallow | Nested objects shared |
| `JSON.parse(JSON.stringify())` | Deep | Loses functions, undefined, Dates |
| `structuredClone()` | Deep | Loses functions, Symbols |

**Q6: Why are primitives faster than objects?**

**A:**

**Reasons:**

1. **Stack vs Heap:**
   - Primitives: stored on stack (fast access)
   - Objects: stored on heap (slower access, pointer dereference)

2. **Allocation Speed:**
   - Primitives: simple stack pointer increment
   - Objects: must find free space in heap

3. **Garbage Collection:**
   - Primitives: automatic cleanup (pop stack frame)
   - Objects: require GC to reclaim memory

4. **Cache Performance:**
   - Primitives: contiguous stack memory (cache-friendly)
   - Objects: scattered heap memory (cache misses)

5. **Property Access:**
   - Primitives: direct value access
   - Objects: hash lookup or hidden class traversal

**Benchmark:**
```javascript
// Primitives
console.time('primitives');
for (let i = 0; i < 1000000; i++) {
  let x = 10;
  let y = 20;
  let sum = x + y;
}
console.timeEnd('primitives'); // ~3ms

// Objects
console.time('objects');
for (let i = 0; i < 1000000; i++) {
  let obj = { x: 10, y: 20 };
  let sum = obj.x + obj.y;
}
console.timeEnd('objects'); // ~40ms (13x slower!)
```

**Q7: How does JavaScript handle memory for large arrays?**

**A:**

Arrays are objects stored on the heap. V8 uses two storage modes:

**1. Fast Mode (Packed Elements):**
- Contiguous memory allocation
- Direct indexed access
- Used when array is dense

```javascript
const fast = [1, 2, 3, 4, 5]; // Packed, contiguous
```

**2. Dictionary Mode (Slow):**
- Hash table storage
- Slower access
- Used when array is sparse or has holes

```javascript
const slow = [];
slow[0] = 1;
slow[1000000] = 2; // Sparse, switches to dictionary mode
```

**Memory optimization tips:**
- Preallocate size: `new Array(length)`
- Avoid holes: don't skip indices
- Avoid mixed types: keep array homogeneous
- Use typed arrays for numeric data: `Int32Array`, `Float64Array`

```javascript
// Slow: mixed types, sparse
const mixed = [];
mixed[0] = 1;
mixed[1] = 'string';
mixed[100] = true;

// Fast: homogeneous, dense
const optimized = new Int32Array(1000);
for (let i = 0; i < 1000; i++) {
  optimized[i] = i;
}
```

**Q8: What are the most common causes of memory leaks in JavaScript?**

**A:**

**1. Unintended Global Variables:**
```javascript
function leak() {
  forgotten = 'global'; // Missing 'let/const'
}
```

**2. Forgotten Timers/Listeners:**
```javascript
setInterval(() => {
  const data = fetchData(); // Accumulates forever
}, 1000);
// Never cleared

element.addEventListener('click', handler);
// Never removed
```

**3. Closures Capturing Large Scope:**
```javascript
function outer() {
  const huge = new Array(1000000);
  return () => console.log('hi'); // Captures entire scope
}
```

**4. Detached DOM Nodes:**
```javascript
const nodes = [];
const node = document.createElement('div');
document.body.appendChild(node);
document.body.removeChild(node);
nodes.push(node); // Keeps detached node in memory
```

**5. Circular References (rare in modern JS):**
```javascript
function createCircular() {
  const obj1 = {};
  const obj2 = {};
  obj1.ref = obj2;
  obj2.ref = obj1; // Modern GC handles this
  return obj1;
}
```

**Prevention:**
- Use strict mode
- Clear timers/listeners when done
- Avoid unnecessary closures
- Nullify references to large objects
- Use WeakMap/WeakSet for metadata

## Key Takeaways

- **Stack** stores primitives and references; **heap** stores objects
- Stack is fast (LIFO, automatic cleanup); heap is slower (requires GC)
- **Primitives** are copied by value; **objects** are copied by reference
- Assignment of objects copies the reference, not the object itself
- **Closures** keep outer scope variables alive in heap memory
- **Shallow copy** shares nested objects; **deep copy** creates independent structures
- Stack overflow occurs when call stack exceeds size limit (deep recursion)
- Memory leaks stem from unintended globals, forgotten timers, and unnecessary closures
- Prefer primitives for performance; reuse objects when possible
- Understanding memory allocation is crucial for performance optimization

## Resources

- [ECMAScript Specification - Memory Model](https://tc39.es/ecma262/#sec-memory-model)
- [V8 Blog - Fast Properties](https://v8.dev/blog/fast-properties)
- [V8 Blog - Elements Kinds](https://v8.dev/blog/elements-kinds)
- [MDN - Memory Management](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Memory_Management)
- [Lin Clark - A Crash Course in Memory Management](https://hacks.mozilla.org/2017/06/a-crash-course-in-memory-management/)
- [Chrome DevTools - Memory Profiler](https://developer.chrome.com/docs/devtools/memory-problems/)
- [JavaScript.info - Garbage Collection](https://javascript.info/garbage-collection)
