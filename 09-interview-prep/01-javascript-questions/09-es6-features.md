# ES6+ Features - JavaScript Interview Questions

## Table of Contents
- [Core Concepts](#core-concepts)
- [Common Interview Questions](#common-interview-questions)
- [Advanced Questions](#advanced-questions)
- [What Interviewers Look For](#what-interviewers-look-for)
- [Red Flags to Avoid](#red-flags-to-avoid)
- [Key Takeaways](#key-takeaways)

## Common Interview Questions

### Question 1: Destructuring - Objects and Arrays

**Expected Answer:**

Destructuring extracts values from arrays or properties from objects into distinct variables.

```javascript
// Object destructuring
const user = { name: 'John', age: 30, email: 'john@example.com' };

const { name, age } = user;
console.log(name, age); // 'John', 30

// Renaming
const { name: userName, age: userAge } = user;

// Default values
const { city = 'NYC' } = user;

// Nested destructuring
const person = {
  name: 'Jane',
  address: { city: 'NYC', zip: '10001' }
};

const { address: { city, zip } } = person;

// Array destructuring
const numbers = [1, 2, 3, 4, 5];
const [first, second, ...rest] = numbers;
console.log(first, second, rest); // 1, 2, [3, 4, 5]

// Skipping elements
const [, , third] = numbers;

// Swapping variables
let a = 1, b = 2;
[a, b] = [b, a];

// Function parameters
function greet({ name, age }) {
  console.log(`Hello ${name}, you are ${age}`);
}

greet({ name: 'John', age: 30 });
```

### Question 2: Spread and Rest Operators

**Expected Answer:**

```javascript
// Spread - expands iterable into individual elements
const arr1 = [1, 2, 3];
const arr2 = [4, 5, 6];
const combined = [...arr1, ...arr2];

// Object spread
const obj1 = { a: 1, b: 2 };
const obj2 = { c: 3, d: 4 };
const merged = { ...obj1, ...obj2 };

// Rest - collects multiple elements into array
function sum(...numbers) {
  return numbers.reduce((acc, n) => acc + n, 0);
}

sum(1, 2, 3, 4); // 10

// Array destructuring with rest
const [first, ...remaining] = [1, 2, 3, 4];

// Object destructuring with rest
const { name, ...otherProps } = user;
```

### Question 3: Template Literals and Tagged Templates

**Expected Answer:**

```javascript
// Basic template literals
const name = 'John';
const age = 30;
const message = `Hello, I'm ${name} and I'm ${age} years old`;

// Multi-line strings
const html = `
  <div>
    <h1>${title}</h1>
    <p>${content}</p>
  </div>
`;

// Tagged templates
function highlight(strings, ...values) {
  return strings.reduce((result, str, i) => {
    return result + str + (values[i] ? `<mark>${values[i]}</mark>` : '');
  }, '');
}

const user = 'John';
const action = 'logged in';
const message = highlight`User ${user} has ${action}`;
// User <mark>John</mark> has <mark>logged in</mark>
```

### Question 4: Arrow Functions

**Expected Answer:**

```javascript
// Basic syntax
const add = (a, b) => a + b;

// Single parameter (no parentheses)
const double = x => x * 2;

// No parameters
const random = () => Math.random();

// Multiple statements (need braces and return)
const calculate = (a, b) => {
  const sum = a + b;
  return sum * 2;
};

// Returning object (wrap in parentheses)
const createUser = (name, age) => ({ name, age });

// Key difference: lexical this
const obj = {
  name: 'Object',
  regularFunc() {
    setTimeout(function() {
      console.log(this.name); // undefined
    }, 100);
  },
  arrowFunc() {
    setTimeout(() => {
      console.log(this.name); // 'Object'
    }, 100);
  }
};
```

### Question 5: Promises and Async/Await

**Expected Answer:**

```javascript
// Promise creation
const fetchData = () => {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve({ data: 'result' });
    }, 1000);
  });
};

// Promise chaining
fetchData()
  .then(result => result.data)
  .then(data => console.log(data))
  .catch(error => console.error(error));

// Async/await
async function getData() {
  try {
    const result = await fetchData();
    console.log(result.data);
  } catch (error) {
    console.error(error);
  }
}

// Parallel execution
async function getMultiple() {
  const [users, posts] = await Promise.all([
    fetchUsers(),
    fetchPosts()
  ]);
}
```

### Question 6: Classes

**Expected Answer:**

```javascript
class Person {
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }
  
  greet() {
    console.log(`Hello, I'm ${this.name}`);
  }
  
  static create(name, age) {
    return new Person(name, age);
  }
}

// Inheritance
class Developer extends Person {
  constructor(name, age, language) {
    super(name, age);
    this.language = language;
  }
  
  code() {
    console.log(`Coding in ${this.language}`);
  }
}

// Private fields (ES2022)
class Counter {
  #count = 0;
  
  increment() {
    this.#count++;
  }
  
  getCount() {
    return this.#count;
  }
}
```

### Question 7: Modules (Import/Export)

**Expected Answer:**

```javascript
// Named exports
export const add = (a, b) => a + b;
export const subtract = (a, b) => a - b;

// Default export
export default class Calculator {
  add(a, b) {
    return a + b;
  }
}

// Importing
import Calculator from './calculator.js';
import { add, subtract } from './math.js';
import * as math from './math.js';

// Re-exporting
export { add, subtract } from './math.js';
export * from './utils.js';
```

### Question 8: Default Parameters

**Expected Answer:**

```javascript
// Basic default
function greet(name = 'Guest') {
  console.log(`Hello, ${name}`);
}

// Expression as default
function createId(prefix = 'id', suffix = Date.now()) {
  return `${prefix}-${suffix}`;
}

// Using previous parameters
function buildUrl(protocol = 'https', domain, path = '/') {
  return `${protocol}://${domain}${path}`;
}

// Destructuring with defaults
function configure({ host = 'localhost', port = 3000 } = {}) {
  console.log(`${host}:${port}`);
}
```

### Question 9: Enhanced Object Literals

**Expected Answer:**

```javascript
const name = 'John';
const age = 30;

// Shorthand properties
const user = { name, age };

// Shorthand methods
const obj = {
  greet() {
    console.log('Hello');
  }
};

// Computed property names
const prop = 'dynamicKey';
const obj = {
  [prop]: 'value',
  [`${prop}2`]: 'value2'
};

// Method properties can use super
const parent = { greet() { return 'Hello'; } };
const child = {
  __proto__: parent,
  greet() {
    return super.greet() + ' World';
  }
};
```

### Question 10: Symbols

**Expected Answer:**

```javascript
// Creating symbols
const sym1 = Symbol('description');
const sym2 = Symbol('description');
console.log(sym1 === sym2); // false (unique)

// Well-known symbols
const obj = {
  [Symbol.iterator]() {
    let i = 0;
    return {
      next: () => ({
        value: i++,
        done: i > 5
      })
    };
  }
};

// Private-like properties
const _private = Symbol('private');
class MyClass {
  constructor() {
    this[_private] = 'secret';
  }
}

// Global symbol registry
const globalSym = Symbol.for('shared');
const same = Symbol.for('shared');
console.log(globalSym === same); // true
```

## Advanced Questions

### Question 11: Generators and Iterators

**Expected Answer:**

```javascript
// Generator function
function* numberGenerator() {
  yield 1;
  yield 2;
  yield 3;
}

const gen = numberGenerator();
console.log(gen.next()); // { value: 1, done: false }
console.log(gen.next()); // { value: 2, done: false }

// Infinite generator
function* infiniteSequence() {
  let i = 0;
  while (true) {
    yield i++;
  }
}

// Generator delegation
function* gen1() {
  yield 1;
  yield 2;
}

function* gen2() {
  yield* gen1();
  yield 3;
}

// Custom iterator
const range = {
  from: 1,
  to: 5,
  *[Symbol.iterator]() {
    for (let i = this.from; i <= this.to; i++) {
      yield i;
    }
  }
};

for (let num of range) {
  console.log(num);
}
```

### Question 12: Proxy and Reflect

**Expected Answer:**

```javascript
// Proxy for object interception
const target = { name: 'John', age: 30 };

const handler = {
  get(target, prop) {
    console.log(`Getting ${prop}`);
    return prop in target ? target[prop] : 'Not found';
  },
  set(target, prop, value) {
    console.log(`Setting ${prop} to ${value}`);
    if (prop === 'age' && typeof value !== 'number') {
      throw new TypeError('Age must be a number');
    }
    target[prop] = value;
    return true;
  }
};

const proxy = new Proxy(target, handler);

// Validation proxy
function createValidator(schema) {
  return new Proxy({}, {
    set(target, prop, value) {
      const validator = schema[prop];
      if (validator && !validator(value)) {
        throw new Error(`Invalid value for ${prop}`);
      }
      target[prop] = value;
      return true;
    }
  });
}

const userSchema = {
  age: val => typeof val === 'number' && val > 0,
  name: val => typeof val === 'string' && val.length > 0
};

const user = createValidator(userSchema);
```

### Question 13: WeakMap and WeakSet

**Expected Answer:**

```javascript
// WeakMap - keys must be objects, garbage collected
const weakMap = new WeakMap();
let obj = { name: 'John' };

weakMap.set(obj, 'metadata');
console.log(weakMap.get(obj)); // 'metadata'

obj = null; // Object can be garbage collected

// Private data pattern
const privateData = new WeakMap();

class Person {
  constructor(name) {
    privateData.set(this, { name });
  }
  
  getName() {
    return privateData.get(this).name;
  }
}

// WeakSet - for objects only
const visitedNodes = new WeakSet();

function traverse(node) {
  if (visitedNodes.has(node)) return;
  visitedNodes.add(node);
  // Process node
}
```

### Question 14: Optional Chaining and Nullish Coalescing

**Expected Answer:**

```javascript
// Optional chaining (?.)
const user = {
  name: 'John',
  address: {
    street: '123 Main St'
  }
};

// Safe property access
const zip = user.address?.zip; // undefined (no error)
const city = user.address?.city?.name; // undefined

// Optional method call
const result = obj.method?.(); // Only calls if method exists

// Optional array access
const item = arr?.[0];

// Nullish coalescing (??)
const value = null ?? 'default'; // 'default'
const value = 0 ?? 'default'; // 0 (not 'default')
const value = '' ?? 'default'; // '' (not 'default')

// vs OR operator
const value = 0 || 'default'; // 'default' (0 is falsy)
const value = 0 ?? 'default'; // 0 (only null/undefined)

// Combined
const name = user?.profile?.name ?? 'Anonymous';
```

### Question 15: BigInt

**Expected Answer:**

```javascript
// Creating BigInt
const big1 = 9007199254740991n;
const big2 = BigInt('9007199254740991');

// Operations
const sum = 100n + 200n;
const product = 50n * 2n;
const division = 10n / 3n; // 3n (truncates)

// Cannot mix with regular numbers
// const mixed = 10n + 5; // TypeError
const mixed = 10n + BigInt(5); // OK

// Comparison
console.log(10n > 5); // true
console.log(10n === 10); // false (different types)
console.log(10n == 10); // true (coercion)

// Use cases: precise integer calculations
const largeNumber = 123456789012345678901234567890n;
```

## What Interviewers Look For

### 1. Core Understanding
- Knowledge of major ES6+ features
- When to use each feature
- Differences from ES5 equivalents
- Browser/environment support awareness

### 2. Practical Application
- Using features appropriately
- Understanding trade-offs
- Modern code patterns
- Async/await over promises when simpler

### 3. Advanced Features
- Generators and iterators
- Proxies and Reflect
- WeakMap/WeakSet use cases
- Symbols for metaprogramming

### 4. Best Practices
- Const over let over var
- Arrow functions for callbacks
- Destructuring for clarity
- Template literals for strings
- Async/await for async code

### 5. Migration Knowledge
- Transpilation (Babel)
- Polyfills for older browsers
- Progressive enhancement
- Feature detection

## Red Flags to Avoid

### 1. Not Using Modern Features
- Still writing ES5 code
- Not using let/const
- Callback hell instead of async/await
- String concatenation instead of templates

### 2. Misusing Features
- Arrow functions for object methods
- Using classes when objects suffice
- Overusing destructuring (readability)
- Spread operator performance issues

### 3. Compatibility Ignorance
- Not knowing browser support
- Not mentioning transpilation
- Assuming all features everywhere
- Not using polyfills when needed

### 4. Missing Context
- Can't explain why feature is useful
- No real-world examples
- Don't know limitations
- Haven't used in production

## Key Takeaways

### Essential Features
1. **let/const**: Block scope, immutability
2. **Arrow functions**: Concise syntax, lexical this
3. **Destructuring**: Extract values elegantly
4. **Spread/rest**: Array/object manipulation
5. **Template literals**: String interpolation
6. **Promises/async-await**: Async operations
7. **Classes**: OOP syntax
8. **Modules**: Code organization

### Best Practices
1. **Default to const**: Use let only when reassigning
2. **Arrow functions for callbacks**: Shorter, lexical this
3. **Destructure for clarity**: Extract what you need
4. **Async/await over chains**: More readable
5. **Template literals for strings**: Cleaner interpolation

### Interview Success Tips
1. **Show modern knowledge**: Use ES6+ in answers
2. **Explain benefits**: Why feature improves code
3. **Mention support**: Transpilation, polyfills
4. **Practical examples**: Real-world use cases
5. **Compare with old way**: Show evolution
6. **Discuss trade-offs**: When not to use

Remember: ES6+ features make JavaScript more powerful and expressive. Use them appropriately to write cleaner, more maintainable code while being mindful of browser support and transpilation needs.
