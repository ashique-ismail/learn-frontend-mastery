# Control Flow and Labeled Statements

## Overview

Control flow determines the order in which statements are executed in a program. JavaScript provides various control flow statements including conditionals (if/else, switch), loops (for, while, do-while), and jump statements (break, continue, return). Understanding control flow is fundamental to writing any program logic, and mastering advanced patterns like labeled statements can help in complex scenarios.

## Conditional Statements

### if Statement

The most basic conditional executes code if a condition is true:

```javascript
if (condition) {
  // Executes if condition is truthy
}

// Without braces (single statement only - not recommended)
if (condition) doSomething();

// Better: Always use braces
if (condition) {
  doSomething();
}
```

### if-else

```javascript
if (condition) {
  // Executes if condition is truthy
} else {
  // Executes if condition is falsy
}

// Real example
if (user.isAdmin) {
  showAdminPanel();
} else {
  showUserPanel();
}
```

### if-else if-else

```javascript
if (condition1) {
  // First condition
} else if (condition2) {
  // Second condition
} else if (condition3) {
  // Third condition
} else {
  // Default case
}

// Real example
if (score >= 90) {
  grade = 'A';
} else if (score >= 80) {
  grade = 'B';
} else if (score >= 70) {
  grade = 'C';
} else if (score >= 60) {
  grade = 'D';
} else {
  grade = 'F';
}
```

### Ternary Operator (Conditional Expression)

A compact alternative for simple if-else:

```javascript
condition ? valueIfTrue : valueIfFalse;

// Assignment
const status = isActive ? 'active' : 'inactive';

// Return value
return user.isAdmin ? 'admin' : 'user';

// Nested ternaries (use sparingly!)
const result = condition1 ? value1
  : condition2 ? value2
  : condition3 ? value3
  : defaultValue;

// Better: Use if-else for complex conditions
let result;
if (condition1) {
  result = value1;
} else if (condition2) {
  result = value2;
} else if (condition3) {
  result = value3;
} else {
  result = defaultValue;
}
```

### switch Statement

For multiple conditions based on a single value:

```javascript
switch (expression) {
  case value1:
    // Code
    break;
  case value2:
    // Code
    break;
  default:
    // Default code
}

// Real example
switch (day) {
  case 'Monday':
    console.log('Start of work week');
    break;
  case 'Friday':
    console.log('End of work week');
    break;
  case 'Saturday':
  case 'Sunday':
    console.log('Weekend!');
    break;
  default:
    console.log('Midweek day');
}
```

### Switch Fall-through

Intentional fall-through can be useful:

```javascript
switch (month) {
  case 1:
  case 3:
  case 5:
  case 7:
  case 8:
  case 10:
  case 12:
    days = 31;
    break;
  case 4:
  case 6:
  case 9:
  case 11:
    days = 30;
    break;
  case 2:
    days = isLeapYear ? 29 : 28;
    break;
  default:
    throw new Error('Invalid month');
}
```

### Switch with Block Scope

Use braces to create block scope in cases:

```javascript
switch (value) {
  case 1: {
    const result = calculate();
    console.log(result);
    break;
  }
  case 2: {
    const result = calculate2(); // Different scope, OK to reuse name
    console.log(result);
    break;
  }
}
```

## Loops

### while Loop

Executes while condition is true:

```javascript
while (condition) {
  // Loop body
}

// Real example
let count = 0;
while (count < 5) {
  console.log(count);
  count++;
}

// Infinite loop (be careful!)
while (true) {
  if (shouldStop()) break;
  doWork();
}
```

### do-while Loop

Executes at least once, then checks condition:

```javascript
do {
  // Loop body (runs at least once)
} while (condition);

// Real example
let input;
do {
  input = prompt("Enter a number greater than 10:");
} while (Number(input) <= 10);

// Useful when you need first execution
let page = 1;
do {
  const results = fetchPage(page);
  displayResults(results);
  page++;
} while (results.hasMore);
```

### for Loop

Traditional C-style loop:

```javascript
for (initialization; condition; increment) {
  // Loop body
}

// Standard usage
for (let i = 0; i < 5; i++) {
  console.log(i); // 0, 1, 2, 3, 4
}

// Multiple variables
for (let i = 0, j = 10; i < j; i++, j--) {
  console.log(i, j); // 0 10, 1 9, 2 8, 3 7, 4 6
}

// Any part can be omitted
let i = 0;
for (; i < 5; i++) {
  console.log(i);
}

// All parts omitted = infinite loop
for (;;) {
  if (shouldStop()) break;
  doWork();
}
```

### for-in Loop

Iterates over enumerable properties of an object:

```javascript
for (const key in object) {
  // Loop body
}

// Object iteration
const person = { name: "Alice", age: 30, city: "Boston" };
for (const key in person) {
  console.log(`${key}: ${person[key]}`);
}
// name: Alice
// age: 30
// city: Boston

// Also works with arrays (not recommended)
const arr = ['a', 'b', 'c'];
for (const index in arr) {
  console.log(index, arr[index]); // Indices as strings: "0", "1", "2"
}

// Problem: Iterates over all enumerable properties including inherited ones
Array.prototype.customMethod = function() {};
const arr2 = [1, 2, 3];
for (const i in arr2) {
  console.log(i); // 0, 1, 2, customMethod (!)
}

// Solution: Use hasOwnProperty check
for (const key in obj) {
  if (obj.hasOwnProperty(key)) {
    console.log(key, obj[key]);
  }
}

// Or use Object.keys/values/entries
Object.keys(obj).forEach(key => {
  console.log(key, obj[key]);
});
```

### for-of Loop

Iterates over iterable objects (arrays, strings, Maps, Sets):

```javascript
for (const element of iterable) {
  // Loop body
}

// Array iteration (preferred over for-in)
const colors = ['red', 'green', 'blue'];
for (const color of colors) {
  console.log(color); // red, green, blue
}

// String iteration
for (const char of 'hello') {
  console.log(char); // h, e, l, l, o
}

// Set iteration
const numbers = new Set([1, 2, 3, 2, 1]);
for (const num of numbers) {
  console.log(num); // 1, 2, 3 (duplicates removed)
}

// Map iteration
const map = new Map([['a', 1], ['b', 2]]);
for (const [key, value] of map) {
  console.log(key, value); // a 1, b 2
}

// With index (using entries())
for (const [index, value] of arr.entries()) {
  console.log(index, value);
}
```

### for-of vs for-in

```javascript
const arr = ['a', 'b', 'c'];

// for-in: Iterates over keys/indices (strings)
for (const key in arr) {
  console.log(key, typeof key); // "0" string, "1" string, "2" string
}

// for-of: Iterates over values
for (const value of arr) {
  console.log(value); // a, b, c
}

// for-in on objects
const obj = { x: 1, y: 2 };
for (const key in obj) {
  console.log(key); // x, y
}

// for-of on objects requires Object.entries()
for (const [key, value] of Object.entries(obj)) {
  console.log(key, value); // x 1, y 2
}
```

## Jump Statements

### break

Exits the current loop or switch:

```javascript
// Exit loop early
for (let i = 0; i < 10; i++) {
  if (i === 5) break;
  console.log(i); // 0, 1, 2, 3, 4
}

// Real example: Find first match
let found = null;
for (const item of items) {
  if (item.id === targetId) {
    found = item;
    break;
  }
}

// In nested loops (breaks only innermost loop)
for (let i = 0; i < 3; i++) {
  for (let j = 0; j < 3; j++) {
    if (j === 1) break; // Only breaks inner loop
    console.log(i, j);
  }
}
// Output: 0 0, 1 0, 2 0
```

### continue

Skips the rest of current iteration:

```javascript
// Skip certain iterations
for (let i = 0; i < 5; i++) {
  if (i === 2) continue;
  console.log(i); // 0, 1, 3, 4 (skips 2)
}

// Real example: Skip invalid items
for (const item of items) {
  if (!item.isValid) continue;
  
  // Process valid items
  processItem(item);
}

// Filter even numbers
for (let i = 0; i < 10; i++) {
  if (i % 2 === 1) continue; // Skip odd numbers
  console.log(i); // 0, 2, 4, 6, 8
}
```

### return

Exits the function and optionally returns a value:

```javascript
function findUser(id) {
  for (const user of users) {
    if (user.id === id) {
      return user; // Exit function immediately
    }
  }
  return null; // Not found
}

// Early return pattern (guard clauses)
function processData(data) {
  // Validate and return early
  if (!data) return null;
  if (data.length === 0) return [];
  if (!data.isValid) return { error: 'Invalid data' };
  
  // Main logic here
  return transform(data);
}

// Multiple return points
function getStatus(score) {
  if (score >= 90) return 'Excellent';
  if (score >= 70) return 'Good';
  if (score >= 50) return 'Average';
  return 'Needs Improvement';
}
```

## Labeled Statements

Labels allow you to name loops and control break/continue behavior across nested loops:

### Syntax

```javascript
labelName: statement

// Common use: nested loops
outer: for (let i = 0; i < 3; i++) {
  inner: for (let j = 0; j < 3; j++) {
    console.log(i, j);
  }
}
```

### Breaking Outer Loop

```javascript
// Without labels (can't break outer loop from inner)
let found = false;
for (let i = 0; i < 3; i++) {
  for (let j = 0; j < 3; j++) {
    if (i === 1 && j === 1) {
      found = true;
      break; // Only breaks inner loop
    }
    if (found) break;
  }
  if (found) break;
}

// With labels (clean break from nested loop)
outer: for (let i = 0; i < 3; i++) {
  for (let j = 0; j < 3; j++) {
    if (i === 1 && j === 1) {
      break outer; // Breaks outer loop directly
    }
    console.log(i, j);
  }
}
// Output: 0 0, 0 1, 0 2, 1 0
```

### Continue with Labels

```javascript
// Continue outer loop from inner loop
outer: for (let i = 0; i < 3; i++) {
  for (let j = 0; j < 3; j++) {
    if (j === 1) continue outer; // Skip to next outer iteration
    console.log(i, j);
  }
}
// Output: 0 0, 1 0, 2 0
```

### Real-World Example: Matrix Search

```javascript
// Find element in 2D array
function findInMatrix(matrix, target) {
  search: for (let row = 0; row < matrix.length; row++) {
    for (let col = 0; col < matrix[row].length; col++) {
      if (matrix[row][col] === target) {
        console.log(`Found at [${row}, ${col}]`);
        break search; // Exit both loops
      }
    }
  }
}

const matrix = [
  [1, 2, 3],
  [4, 5, 6],
  [7, 8, 9]
];
findInMatrix(matrix, 5); // Found at [1, 1]
```

### Block Labels

Labels can be used with any statement:

```javascript
// Labeled block
myBlock: {
  console.log('Start');
  if (condition) {
    break myBlock; // Exit the block
  }
  console.log('Middle'); // Skipped if condition is true
}
console.log('End'); // Always executes

// Real use case: Complex error handling
validation: {
  if (!data) {
    error = 'No data';
    break validation;
  }
  if (!data.email) {
    error = 'No email';
    break validation;
  }
  if (!isValidEmail(data.email)) {
    error = 'Invalid email';
    break validation;
  }
  error = null;
}
if (error) {
  handleError(error);
}
```

## Modern Control Flow Patterns

### Early Return (Guard Clauses)

```javascript
// Bad: Deep nesting
function processUser(user) {
  if (user) {
    if (user.isActive) {
      if (user.hasPermission) {
        // Do work
        return result;
      }
    }
  }
  return null;
}

// Good: Early returns
function processUser(user) {
  if (!user) return null;
  if (!user.isActive) return null;
  if (!user.hasPermission) return null;
  
  // Do work at top level
  return result;
}
```

### Optional Chaining Instead of Nested Ifs

```javascript
// Bad: Nested checks
let street;
if (user) {
  if (user.address) {
    if (user.address.street) {
      street = user.address.street;
    }
  }
}

// Good: Optional chaining
const street = user?.address?.street;
```

### Array Methods Over Loops

```javascript
// Old: Traditional loop
const results = [];
for (let i = 0; i < items.length; i++) {
  if (items[i].active) {
    results.push(transform(items[i]));
  }
}

// Modern: Array methods
const results = items
  .filter(item => item.active)
  .map(transform);
```

### Switch to Object Lookup

```javascript
// Old: Long switch
function getColor(status) {
  switch (status) {
    case 'active': return 'green';
    case 'pending': return 'yellow';
    case 'inactive': return 'gray';
    default: return 'black';
  }
}

// Modern: Object lookup
const STATUS_COLORS = {
  active: 'green',
  pending: 'yellow',
  inactive: 'gray'
};

function getColor(status) {
  return STATUS_COLORS[status] ?? 'black';
}
```

## Performance Considerations

### Loop Performance

```javascript
// Cache array length
const len = arr.length;
for (let i = 0; i < len; i++) {
  // Process arr[i]
}

// Or use for-of (modern engines optimize this)
for (const item of arr) {
  // Process item
}

// Avoid: Recalculating length every iteration
for (let i = 0; i < arr.length; i++) { /* ... */ }
```

### Break Early

```javascript
// Good: Exit as soon as possible
function hasValue(arr, target) {
  for (const item of arr) {
    if (item === target) return true; // Early exit
  }
  return false;
}

// Or use built-in methods
function hasValue(arr, target) {
  return arr.includes(target); // Optimized by engine
}
```

## Common Pitfalls

### 1. Missing Break in Switch

```javascript
// Bug: Fall-through unintended
switch (value) {
  case 1:
    doSomething();
    // Missing break!
  case 2:
    doSomethingElse(); // Executes for both case 1 and 2!
    break;
}
```

### 2. Variable Scope in Loops

```javascript
// Problem: var is function-scoped
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Prints: 3, 3, 3

// Solution: Use let (block-scoped)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Prints: 0, 1, 2
```

### 3. Modifying Array While Iterating

```javascript
// Bad: Modifying array during iteration
for (let i = 0; i < arr.length; i++) {
  if (condition) {
    arr.splice(i, 1); // Modifies array, skips elements!
  }
}

// Good: Iterate backwards
for (let i = arr.length - 1; i >= 0; i--) {
  if (condition) {
    arr.splice(i, 1);
  }
}

// Better: Create new array
const filtered = arr.filter(item => !condition);
```

### 4. Infinite Loops

```javascript
// Bug: Condition never becomes false
let i = 0;
while (i < 10) {
  console.log(i);
  // Forgot to increment i!
}

// Bug: Off-by-one error
for (let i = 0; i <= arr.length; i++) { // Should be <, not <=
  console.log(arr[i]); // arr[arr.length] is undefined!
}
```

## Best Practices

### 1. Always Use Braces

```javascript
// Bad: Easy to introduce bugs
if (condition)
  doSomething();
  doSomethingElse(); // Not in if block!

// Good: Clear structure
if (condition) {
  doSomething();
  doSomethingElse();
}
```

### 2. Prefer const and let Over var

```javascript
// Bad: var has confusing scope rules
for (var i = 0; i < 10; i++) {
  // ...
}
console.log(i); // 10 (leaked outside loop!)

// Good: let is block-scoped
for (let i = 0; i < 10; i++) {
  // ...
}
console.log(i); // ReferenceError
```

### 3. Use Appropriate Loop Type

```javascript
// Iterating object properties: for-in or Object.keys/values/entries
for (const key in obj) { /* ... */ }
for (const [key, value] of Object.entries(obj)) { /* ... */ }

// Iterating arrays: for-of or array methods
for (const item of array) { /* ... */ }
array.forEach(item => { /* ... */ });

// Index needed: traditional for or forEach with index
for (let i = 0; i < array.length; i++) { /* ... */ }
array.forEach((item, index) => { /* ... */ });
```

### 4. Minimize Nested Loops

```javascript
// Bad: O(n²) complexity
for (const user of users) {
  for (const order of orders) {
    if (order.userId === user.id) {
      // Process
    }
  }
}

// Better: Use Map for O(n) lookup
const ordersByUser = new Map();
for (const order of orders) {
  if (!ordersByUser.has(order.userId)) {
    ordersByUser.set(order.userId, []);
  }
  ordersByUser.get(order.userId).push(order);
}

for (const user of users) {
  const userOrders = ordersByUser.get(user.id) ?? [];
  // Process
}
```

## Resources

### Documentation
- [MDN: Control Flow](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Control_flow_and_error_handling)
- [MDN: Loops and Iteration](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Loops_and_iteration)
- [MDN: Labeled Statement](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/label)

### Articles
- [JavaScript Control Flow](https://javascript.info/ifelse)
- [Understanding Loops](https://javascript.info/while-for)
