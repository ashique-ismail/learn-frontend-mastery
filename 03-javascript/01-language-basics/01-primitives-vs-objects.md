# Primitives vs Objects: Value Types and Reference Types

## The Idea

**In plain English:** JavaScript stores data in two ways: as a simple, standalone value (called a primitive) or as a container that can hold many values and be changed over time (called an object). The key difference is that when you copy a primitive you get your own independent copy, but when you copy an object you just get another pointer to the exact same container.

**Real-world analogy:** Imagine a school locker system. Each student either carries a sticky note with a number written on it, or they are handed a locker key that opens a shared locker in the hallway. If two friends both write the number 42 on their own sticky notes, changing one note does not affect the other. But if two friends are given keys to the same locker and one friend puts a new item inside, the other friend opens the locker and finds that item too.

- The sticky note with the number = a primitive value (each variable holds its own copy)
- The locker key = a reference (the variable points to a shared object in memory)
- The locker itself = the object stored in heap memory

---

## Overview

JavaScript has two fundamental categories of data types: **primitives** (value types) and **objects** (reference types). Understanding the distinction between these types is critical for predicting how data behaves when assigned, passed to functions, or compared. This distinction affects memory allocation, mutability, equality comparisons, and is a common source of bugs for developers transitioning from other languages.

## Primitive Types

JavaScript has seven primitive types:

1. **String**: `"hello"`, `'world'`, `` `template` ``
2. **Number**: `42`, `3.14`, `NaN`, `Infinity`
3. **BigInt**: `123n`, `BigInt(456)`
4. **Boolean**: `true`, `false`
5. **Undefined**: `undefined`
6. **Null**: `null`
7. **Symbol**: `Symbol('description')`

### Characteristics of Primitives

#### 1. Immutable

Primitives cannot be altered. Operations on primitives always return new values:

```javascript
let str = "hello";
str.toUpperCase(); // Returns "HELLO"
console.log(str);  // Still "hello" - original unchanged

// Even methods that appear to "modify" strings create new values
let text = "JavaScript";
text[0] = "X"; // Fails silently (TypeError in strict mode)
console.log(text); // Still "JavaScript"
```

#### 2. Passed/Assigned by Value

When assigned or passed, primitives are copied:

```javascript
let a = 10;
let b = a; // b gets a COPY of a's value
b = 20;

console.log(a); // 10 (unchanged)
console.log(b); // 20

// Function parameters are also copied
function increment(num) {
  num = num + 1;
  return num;
}

let x = 5;
let result = increment(x);
console.log(x);      // 5 (unchanged)
console.log(result); // 6
```

#### 3. Compared by Value

Equality operators compare the actual values:

```javascript
let num1 = 42;
let num2 = 42;
console.log(num1 === num2); // true

let str1 = "hello";
let str2 = "hello";
console.log(str1 === str2); // true

// Works even with different string creation methods
let str3 = "hello";
let str4 = "hel" + "lo";
console.log(str3 === str4); // true
```

#### 4. Stored on the Stack

Primitives are stored directly on the call stack, making access very fast:

```javascript
// Each primitive is stored directly in its variable's memory slot
let age = 30;      // 30 stored on stack
let name = "Alice"; // "Alice" stored on stack
let active = true; // true stored on stack
```

### Primitive Wrapper Objects

JavaScript automatically wraps primitives in objects when you access properties or methods:

```javascript
let str = "hello";
console.log(str.length);        // 5
console.log(str.toUpperCase()); // "HELLO"

// What actually happens (conceptually):
// let temp = new String(str);
// let result = temp.toUpperCase();
// temp is discarded

// This is why adding properties to primitives doesn't work:
let num = 42;
num.customProperty = "test";
console.log(num.customProperty); // undefined
```

The wrapper objects are:

- `String` for string primitives
- `Number` for number primitives
- `Boolean` for boolean primitives
- `BigInt` for bigint primitives
- `Symbol` for symbol primitives

```javascript
// You can create wrapper objects explicitly (but shouldn't)
let primitiveStr = "hello";
let objectStr = new String("hello");

console.log(typeof primitiveStr); // "string"
console.log(typeof objectStr);    // "object"
console.log(primitiveStr === objectStr); // false!
console.log(primitiveStr == objectStr);  // true (coercion)

// Wrapper objects are problematic:
if (primitiveStr) { /* executes - truthy */ }
if (objectStr) { /* ALWAYS executes - objects are always truthy! */ }

let wrappedFalse = new Boolean(false);
if (wrappedFalse) {
  console.log("Objects are always truthy!"); // This runs!
}
```

## Object Types (Reference Types)

Everything that's not a primitive is an object:

- Plain objects: `{}`
- Arrays: `[]`
- Functions: `function() {}`, `() => {}`
- Dates: `new Date()`
- RegExp: `/pattern/`, `new RegExp()`
- Maps, Sets, WeakMaps, WeakSets
- Typed Arrays: `Uint8Array`, etc.
- And many more...

### Characteristics of Objects

#### 1. Mutable

Objects can be modified in place:

```javascript
const person = { name: "Alice", age: 30 };
person.age = 31;           // Modifies the object
person.city = "New York";  // Adds a property
delete person.age;         // Removes a property

console.log(person); // { name: "Alice", city: "New York" }

// Arrays are objects too
const numbers = [1, 2, 3];
numbers.push(4);      // Modifies in place
numbers[0] = 99;      // Changes element
console.log(numbers); // [99, 2, 3, 4]
```

Note: `const` prevents reassignment, not mutation:

```javascript
const obj = { value: 1 };
obj.value = 2;  // OK - mutating
obj = {};       // TypeError - reassigning
```

#### 2. Passed/Assigned by Reference

Variables hold references (memory addresses) to objects, not the objects themselves:

```javascript
let obj1 = { count: 0 };
let obj2 = obj1; // obj2 references the SAME object

obj2.count = 5;

console.log(obj1.count); // 5 (both variables point to same object)
console.log(obj2.count); // 5

// Function parameters receive references
function addProperty(object) {
  object.newProp = "added";
}

let myObj = { existing: true };
addProperty(myObj);
console.log(myObj); // { existing: true, newProp: "added" }
```

This is a common source of bugs:

```javascript
// Attempting to create a copy
let original = { name: "Alice", scores: [90, 85] };
let copy = original; // NOT a copy - same reference!

copy.name = "Bob";
console.log(original.name); // "Bob" - original was modified!

// Both variables point to the same object in memory
console.log(original === copy); // true
```

#### 3. Compared by Reference

Equality operators compare references, not contents:

```javascript
let obj1 = { value: 42 };
let obj2 = { value: 42 };

console.log(obj1 === obj2); // false - different objects
console.log(obj1 == obj2);  // false - still different references

let obj3 = obj1;
console.log(obj1 === obj3); // true - same reference

// Arrays too
let arr1 = [1, 2, 3];
let arr2 = [1, 2, 3];
console.log(arr1 === arr2); // false - different arrays

// Even empty objects/arrays are different
console.log({} === {});   // false
console.log([] === []);   // false
```

#### 4. Stored on the Heap

Objects are stored in heap memory, with variables holding pointers:

```javascript
// Stack (variable): person -> Heap (object): { name: "Alice", age: 30 }
let person = { name: "Alice", age: 30 };

// Stack: user -> same Heap object as person
let user = person;
```

## Copying Objects

### Shallow Copy

Creates a new object but nested objects remain shared:

```javascript
// Method 1: Spread operator
let original = { a: 1, b: { nested: 2 } };
let copy1 = { ...original };

copy1.a = 99;
console.log(original.a); // 1 (primitive copied)

copy1.b.nested = 99;
console.log(original.b.nested); // 99 (nested object shared!)

// Method 2: Object.assign
let copy2 = Object.assign({}, original);

// Method 3: Array spread for arrays
let arr = [1, 2, [3, 4]];
let arrCopy = [...arr];
arrCopy[0] = 99;
console.log(arr[0]); // 1 (copied)

arrCopy[2][0] = 99;
console.log(arr[2][0]); // 99 (nested array shared!)
```

### Deep Copy

Creates completely independent copies:

```javascript
// Method 1: JSON serialization (limitations apply)
let original = { a: 1, b: { nested: 2 } };
let deepCopy = JSON.parse(JSON.stringify(original));

deepCopy.b.nested = 99;
console.log(original.b.nested); // 2 (independent!)

// Limitations of JSON method:
let problematic = {
  date: new Date(),
  regex: /pattern/,
  method: function() {},
  undef: undefined,
  symbol: Symbol('test'),
  circular: null
};
problematic.circular = problematic; // Circular reference

// JSON.stringify loses information:
let jsonCopy = JSON.parse(JSON.stringify(problematic));
console.log(jsonCopy);
// {
//   date: "2024-05-26T12:00:00.000Z", // Converted to string!
//   regex: {},                         // Becomes empty object!
//   // method: missing (functions removed)
//   // undef: missing (undefined removed)
//   // symbol: missing (symbols removed)
//   // circular: would throw error
// }

// Method 2: structuredClone (modern solution)
let better = {
  date: new Date(),
  regex: /pattern/,
  map: new Map([['key', 'value']]),
  set: new Set([1, 2, 3]),
  arrayBuffer: new Uint8Array([1, 2, 3])
};

let structuredCopy = structuredClone(better);
console.log(structuredCopy.date instanceof Date); // true
console.log(structuredCopy.map instanceof Map);   // true

// Limitations: Still can't clone functions, DOM nodes, symbols
```

Manual deep copy for full control:

```javascript
function deepClone(obj, visited = new WeakMap()) {
  // Handle primitives and null
  if (obj === null || typeof obj !== 'object') {
    return obj;
  }
  
  // Handle circular references
  if (visited.has(obj)) {
    return visited.get(obj);
  }
  
  // Handle Date
  if (obj instanceof Date) {
    return new Date(obj);
  }
  
  // Handle RegExp
  if (obj instanceof RegExp) {
    return new RegExp(obj);
  }
  
  // Handle Array
  if (Array.isArray(obj)) {
    const arrCopy = [];
    visited.set(obj, arrCopy);
    obj.forEach((item, index) => {
      arrCopy[index] = deepClone(item, visited);
    });
    return arrCopy;
  }
  
  // Handle Object
  const objCopy = {};
  visited.set(obj, objCopy);
  Object.keys(obj).forEach(key => {
    objCopy[key] = deepClone(obj[key], visited);
  });
  
  return objCopy;
}
```

## Type Checking

### typeof Operator

Works well for primitives (except null), less useful for objects:

```javascript
// Primitives
console.log(typeof "hello");     // "string"
console.log(typeof 42);          // "number"
console.log(typeof 123n);        // "bigint"
console.log(typeof true);        // "boolean"
console.log(typeof undefined);   // "undefined"
console.log(typeof Symbol('x')); // "symbol"

// null is special (historical bug)
console.log(typeof null); // "object" (this is wrong but unfixable)

// Objects
console.log(typeof {});           // "object"
console.log(typeof []);           // "object" (not "array")
console.log(typeof new Date());   // "object"
console.log(typeof /regex/);      // "object"
console.log(typeof function(){}); // "function" (special case)
```

### Better Object Type Checking

```javascript
// Array check
console.log(Array.isArray([]));        // true
console.log(Array.isArray({}));        // false

// Constructor check
console.log(new Date() instanceof Date);  // true
console.log(/regex/ instanceof RegExp);   // true

// Object.prototype.toString trick (most reliable)
const getType = (value) => 
  Object.prototype.toString.call(value).slice(8, -1);

console.log(getType([]));           // "Array"
console.log(getType({}));           // "Object"
console.log(getType(new Date()));   // "Date"
console.log(getType(/regex/));      // "RegExp"
console.log(getType(null));         // "Null"
console.log(getType(undefined));    // "Undefined"
console.log(getType(42));           // "Number"
console.log(getType("hello"));      // "String"
```

## Comparing Values

### Primitive Comparison

```javascript
// Strings
console.log("hello" === "hello"); // true
console.log("Hello" === "hello"); // false (case-sensitive)

// Numbers
console.log(42 === 42);           // true
console.log(0 === -0);            // true (even though different)
console.log(NaN === NaN);         // false (NaN is not equal to itself!)

// Use Object.is for more precise comparison
console.log(Object.is(0, -0));    // false
console.log(Object.is(NaN, NaN)); // true
```

### Object Comparison

Must compare properties manually:

```javascript
function shallowEqual(obj1, obj2) {
  const keys1 = Object.keys(obj1);
  const keys2 = Object.keys(obj2);
  
  if (keys1.length !== keys2.length) {
    return false;
  }
  
  return keys1.every(key => obj1[key] === obj2[key]);
}

console.log(shallowEqual(
  { a: 1, b: 2 },
  { a: 1, b: 2 }
)); // true

console.log(shallowEqual(
  { a: 1, b: { nested: 2 } },
  { a: 1, b: { nested: 2 } }
)); // false (nested objects are different references)

// Deep equality
function deepEqual(obj1, obj2) {
  if (obj1 === obj2) return true;
  
  if (obj1 === null || obj2 === null) return false;
  if (typeof obj1 !== 'object' || typeof obj2 !== 'object') return false;
  
  const keys1 = Object.keys(obj1);
  const keys2 = Object.keys(obj2);
  
  if (keys1.length !== keys2.length) return false;
  
  return keys1.every(key => deepEqual(obj1[key], obj2[key]));
}

console.log(deepEqual(
  { a: 1, b: { nested: 2 } },
  { a: 1, b: { nested: 2 } }
)); // true
```

## Common Pitfalls

### 1. Accidental Object Mutation

```javascript
// Problem: Mutating shared state
function addDiscount(product) {
  product.price = product.price * 0.9;
  return product;
}

const laptop = { name: "Laptop", price: 1000 };
const discounted = addDiscount(laptop);

console.log(laptop.price);     // 900 (oops! original was changed)
console.log(discounted.price); // 900

// Solution: Create a new object
function addDiscountPure(product) {
  return {
    ...product,
    price: product.price * 0.9
  };
}

const laptop2 = { name: "Laptop", price: 1000 };
const discounted2 = addDiscountPure(laptop2);

console.log(laptop2.price);     // 1000 (original preserved)
console.log(discounted2.price); // 900
```

### 2. Array Method Confusion

```javascript
// Mutating methods (modify original)
const arr1 = [1, 2, 3];
arr1.push(4);        // Returns new length
arr1.pop();          // Returns removed element
arr1.shift();        // Returns removed element
arr1.unshift(0);     // Returns new length
arr1.splice(1, 1);   // Returns removed elements
arr1.reverse();      // Returns reversed array (mutates!)
arr1.sort();         // Returns sorted array (mutates!)

// Non-mutating methods (return new array)
const arr2 = [1, 2, 3];
const mapped = arr2.map(x => x * 2);     // [2, 4, 6]
const filtered = arr2.filter(x => x > 1); // [2, 3]
const sliced = arr2.slice(1, 2);         // [2]
const concatenated = arr2.concat([4, 5]); // [1, 2, 3, 4, 5]

console.log(arr2); // [1, 2, 3] (unchanged)
```

### 3. Default Parameters with Objects

```javascript
// Problem: Shared default object
function createUser(options = {}) {
  options.id = Math.random();
  return options;
}

const user1 = createUser();
const user2 = createUser();

// Both get the SAME object if no argument passed!
// (actually not true - each call creates new empty object)

// But this IS a problem:
const DEFAULT_OPTIONS = { timeout: 5000 };

function makeRequest(options = DEFAULT_OPTIONS) {
  options.timestamp = Date.now();
  // Mutates the shared default!
}

makeRequest();
console.log(DEFAULT_OPTIONS); // { timeout: 5000, timestamp: 1234567890 }

// Solution: Create new object
function makeRequestSafe(options = {}) {
  const config = { ...DEFAULT_OPTIONS, ...options };
  config.timestamp = Date.now();
  return config;
}
```

### 4. Comparing Objects in Conditions

```javascript
// Problem: Object equality
const settings = { theme: 'dark' };

if (settings === { theme: 'dark' }) { // Always false!
  console.log("This never runs");
}

// Solution: Compare properties
if (settings.theme === 'dark') {
  console.log("This works");
}

// For arrays:
const arr = [1, 2, 3];

if (arr === [1, 2, 3]) { // Always false!
  console.log("This never runs");
}

// Solution: Deep comparison or use a library
import isEqual from 'lodash/isEqual';
if (isEqual(arr, [1, 2, 3])) {
  console.log("This works");
}
```

## Performance Considerations

### Memory Usage

```javascript
// Primitives are lightweight
let numbers = new Array(1000000).fill(42);
// Only primitive values stored

// Objects are heavier
let objects = new Array(1000000).fill({ value: 42 });
// 1 million objects in heap memory!

// Better: Reuse object structure when possible
let shared = { value: 42 };
let references = new Array(1000000).fill(shared);
// Only 1 object, 1 million references
```

### Garbage Collection

```javascript
// Objects are garbage collected when no references remain
function createLargeObject() {
  let large = new Array(1000000).fill({ data: "lots of data" });
  return large[0];
}

let single = createLargeObject();
// The entire array and 999,999 objects are now eligible for GC
// Only single reference to one object remains
```

## Best Practices

### 1. Prefer Immutability

```javascript
// Bad: Mutation
function updateUser(user, changes) {
  Object.assign(user, changes);
  return user;
}

// Good: Create new object
function updateUser(user, changes) {
  return { ...user, ...changes };
}
```

### 2. Use Const for References

```javascript
// Good: Signals object won't be reassigned
const config = { timeout: 5000 };
config.retries = 3; // Still mutable, but won't be reassigned

// Bad: Misleading
let config = { timeout: 5000 };
// Looks like it might be reassigned, but isn't
```

### 3. Clone Before Mutating

```javascript
// Bad: Mutates input
function sortUsers(users) {
  return users.sort((a, b) => a.name.localeCompare(b.name));
}

// Good: Clone first
function sortUsers(users) {
  return [...users].sort((a, b) => a.name.localeCompare(b.name));
}
```

### 4. Be Careful with Object Identity in React

```javascript
// Bad: Creates new object on every render
function Component() {
  const style = { color: 'red' }; // New object every render!
  return <div style={style}>Text</div>;
}

// Good: Use useMemo or move outside
const STYLE = { color: 'red' }; // Created once

function Component() {
  return <div style={STYLE}>Text</div>;
}
```

## Resources

### Documentation

- [MDN: JavaScript data types and data structures](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures)
- [MDN: Memory Management](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Memory_Management)
- [ECMAScript: Types](https://tc39.es/ecma262/#sec-ecmascript-data-types-and-values)

### Articles

- [Primitive vs. Reference Values](https://javascript.info/types)
- [Deep vs Shallow Copy](https://javascript.info/object-copy)
- [Understanding Value vs Reference](https://codeburst.io/explaining-value-vs-reference-in-javascript-647a975e12a0)

### Books

- "You Don't Know JS: Types & Grammar" by Kyle Simpson
- "JavaScript: The Good Parts" by Douglas Crockford
