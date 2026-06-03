# Functional Programming - JavaScript Interview Questions

## Table of Contents
- [Core Concepts](#core-concepts)
- [Common Interview Questions](#common-interview-questions)
- [Advanced Questions](#advanced-questions)
- [Practical Examples](#practical-examples)
- [What Interviewers Look For](#what-interviewers-look-for)
- [Red Flags to Avoid](#red-flags-to-avoid)
- [Key Takeaways](#key-takeaways)

## Core Concepts

### Functional Programming Principles

1. **Pure Functions**: Same input always produces same output, no side effects
2. **Immutability**: Data cannot be changed after creation
3. **First-Class Functions**: Functions as values
4. **Higher-Order Functions**: Functions that take/return functions
5. **Function Composition**: Combining functions to build complexity
6. **Declarative**: What to do, not how to do it

## Common Interview Questions

### Question 1: What are Pure Functions and Why Are They Important?

**Expected Answer:**

A pure function is a function that:
1. Always returns the same output for the same input
2. Has no side effects (doesn't modify external state)

```javascript
// Pure function
function add(a, b) {
  return a + b;
}

console.log(add(2, 3)); // 5 (always)
console.log(add(2, 3)); // 5 (always)

// Impure function - depends on external state
let counter = 0;

function increment() {
  counter++; // Side effect - modifies external variable
  return counter;
}

console.log(increment()); // 1
console.log(increment()); // 2 (different output)

// Impure function - modifies input
function addToArray(arr, item) {
  arr.push(item); // Side effect - mutates input
  return arr;
}

// Pure alternative
function addToArrayPure(arr, item) {
  return [...arr, item]; // Returns new array
}

const original = [1, 2, 3];
const updated = addToArrayPure(original, 4);

console.log(original); // [1, 2, 3] (unchanged)
console.log(updated);  // [1, 2, 3, 4]
```

**Benefits of Pure Functions:**
- Testable (predictable output)
- Cacheable (memoization)
- Portable (no dependencies)
- Parallelizable (no race conditions)
- Easy to reason about

**What Interviewers Look For:**
- Clear definition of purity
- Understanding of side effects
- Can identify impure functions
- Knows how to make functions pure

### Question 2: Explain Higher-Order Functions with Examples

**Expected Answer:**

Higher-order functions either take functions as arguments or return functions (or both).

```javascript
// Takes function as argument
function map(array, fn) {
  const result = [];
  for (let item of array) {
    result.push(fn(item));
  }
  return result;
}

const numbers = [1, 2, 3, 4];
const doubled = map(numbers, x => x * 2);
console.log(doubled); // [2, 4, 6, 8]

// Returns a function
function multiplier(factor) {
  return function(number) {
    return number * factor;
  };
}

const double = multiplier(2);
const triple = multiplier(3);

console.log(double(5)); // 10
console.log(triple(5)); // 15

// Both takes and returns functions
function pipe(...fns) {
  return function(value) {
    return fns.reduce((acc, fn) => fn(acc), value);
  };
}

const addOne = x => x + 1;
const double = x => x * 2;
const square = x => x * x;

const compute = pipe(addOne, double, square);
console.log(compute(2)); // ((2 + 1) * 2)^2 = 36
```

**Common Higher-Order Functions:**
```javascript
// Array methods
const numbers = [1, 2, 3, 4, 5];

// map - transform each element
const doubled = numbers.map(x => x * 2);

// filter - select elements
const evens = numbers.filter(x => x % 2 === 0);

// reduce - accumulate value
const sum = numbers.reduce((acc, x) => acc + x, 0);

// forEach - side effects (less functional)
numbers.forEach(x => console.log(x));

// some/every - testing
const hasEven = numbers.some(x => x % 2 === 0);
const allPositive = numbers.every(x => x > 0);
```

### Question 3: What is Function Composition?

**Expected Answer:**

Function composition is combining simple functions to build more complex ones.

```javascript
// Basic composition
const compose = (f, g) => x => f(g(x));

const add1 = x => x + 1;
const multiply2 = x => x * 2;

const add1ThenMultiply2 = compose(multiply2, add1);
console.log(add1ThenMultiply2(3)); // (3 + 1) * 2 = 8

// Multiple function composition
const composeAll = (...fns) => x => 
  fns.reduceRight((acc, fn) => fn(acc), x);

const subtract1 = x => x - 1;
const square = x => x * x;

const computation = composeAll(square, multiply2, add1);
console.log(computation(3)); // square(multiply2(add1(3))) = 64

// Pipe (left to right)
const pipe = (...fns) => x => 
  fns.reduce((acc, fn) => fn(acc), x);

const computation2 = pipe(add1, multiply2, square);
console.log(computation2(3)); // ((3 + 1) * 2)^2 = 64

// Real-world example
const trim = str => str.trim();
const toLowerCase = str => str.toLowerCase();
const removeSpaces = str => str.replace(/\s+/g, '-');

const slugify = pipe(trim, toLowerCase, removeSpaces);
console.log(slugify('  Hello World  ')); // "hello-world"
```

**Point-Free Style:**
```javascript
// Not point-free (explicitly mentions data parameter)
const getNames = users => users.map(user => user.name);

// Point-free (no explicit data parameter)
const getName = user => user.name;
const getNames = users => users.map(getName);

// Or even more point-free
const getNames = map(getName);
```

### Question 4: Explain Immutability and Its Benefits

**Expected Answer:**

Immutability means data cannot be changed after creation. Instead of modifying, you create new values.

```javascript
// Mutable (bad)
const user = { name: 'John', age: 30 };
user.age = 31; // Mutation
user.email = 'john@example.com'; // Mutation

// Immutable (good)
const user = { name: 'John', age: 30 };
const updatedUser = { ...user, age: 31 }; // New object
const userWithEmail = { ...updatedUser, email: 'john@example.com' };

// Array mutations
const numbers = [1, 2, 3];

// Mutable
numbers.push(4); // Mutation
numbers[0] = 0; // Mutation

// Immutable
const withFour = [...numbers, 4]; // New array
const withZero = [0, ...numbers.slice(1)]; // New array

// Immutable update helpers
const updateObject = (obj, key, value) => ({
  ...obj,
  [key]: value
});

const updateArray = (arr, index, value) => [
  ...arr.slice(0, index),
  value,
  ...arr.slice(index + 1)
];

const removeFromArray = (arr, index) => [
  ...arr.slice(0, index),
  ...arr.slice(index + 1)
];

// Nested immutable updates
const state = {
  user: {
    name: 'John',
    address: {
      city: 'NYC'
    }
  }
};

// Update nested property immutably
const newState = {
  ...state,
  user: {
    ...state.user,
    address: {
      ...state.user.address,
      city: 'LA'
    }
  }
};

// Using Immer library (easier)
import produce from 'immer';

const newState = produce(state, draft => {
  draft.user.address.city = 'LA';
});
```

**Benefits:**
- Predictable state changes
- Time-travel debugging
- Easy change detection
- No unexpected mutations
- Thread-safe (in multi-threaded environments)

### Question 5: What is Currying?

**Expected Answer:**

Currying transforms a function with multiple arguments into a sequence of functions each taking a single argument.

```javascript
// Regular function
function add(a, b, c) {
  return a + b + c;
}

console.log(add(1, 2, 3)); // 6

// Curried version
function addCurried(a) {
  return function(b) {
    return function(c) {
      return a + b + c;
    };
  };
}

console.log(addCurried(1)(2)(3)); // 6

// ES6 arrow functions
const addCurried = a => b => c => a + b + c;

// Generic curry function
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    } else {
      return function(...nextArgs) {
        return curried.apply(this, args.concat(nextArgs));
      };
    }
  };
}

const add = (a, b, c) => a + b + c;
const curriedAdd = curry(add);

console.log(curriedAdd(1)(2)(3)); // 6
console.log(curriedAdd(1, 2)(3)); // 6
console.log(curriedAdd(1)(2, 3)); // 6

// Practical use - partial application
const multiply = (a, b) => a * b;
const curriedMultiply = curry(multiply);

const double = curriedMultiply(2);
const triple = curriedMultiply(3);

console.log(double(5)); // 10
console.log(triple(5)); // 15

// Real-world example - API client
const makeRequest = curry((method, url, data) => {
  return fetch(url, { method, body: JSON.stringify(data) });
});

const get = makeRequest('GET');
const post = makeRequest('POST');

const getUsers = get('/api/users');
const createUser = post('/api/users');

getUsers(); // GET /api/users
createUser({ name: 'John' }); // POST /api/users
```

## Advanced Questions

### Question 6: Implement map, filter, and reduce from Scratch

**Interview Question:** "Can you implement these array methods functionally?"

**Solution:**
```javascript
// map
const map = (array, fn) => {
  const result = [];
  for (let i = 0; i < array.length; i++) {
    result.push(fn(array[i], i, array));
  }
  return result;
};

// Recursive version
const mapRecursive = ([head, ...tail], fn) =>
  head === undefined
    ? []
    : [fn(head), ...mapRecursive(tail, fn)];

// filter
const filter = (array, predicate) => {
  const result = [];
  for (let i = 0; i < array.length; i++) {
    if (predicate(array[i], i, array)) {
      result.push(array[i]);
    }
  }
  return result;
};

// Recursive version
const filterRecursive = ([head, ...tail], predicate) =>
  head === undefined
    ? []
    : predicate(head)
    ? [head, ...filterRecursive(tail, predicate)]
    : filterRecursive(tail, predicate);

// reduce
const reduce = (array, fn, initial) => {
  let accumulator = initial;
  for (let i = 0; i < array.length; i++) {
    accumulator = fn(accumulator, array[i], i, array);
  }
  return accumulator;
};

// Recursive version
const reduceRecursive = ([head, ...tail], fn, accumulator) =>
  head === undefined
    ? accumulator
    : reduceRecursive(tail, fn, fn(accumulator, head));

// Tests
const numbers = [1, 2, 3, 4, 5];

console.log(map(numbers, x => x * 2)); // [2, 4, 6, 8, 10]
console.log(filter(numbers, x => x % 2 === 0)); // [2, 4]
console.log(reduce(numbers, (acc, x) => acc + x, 0)); // 15
```

### Question 7: Implement Memoization

**Interview Question:** "Create a memoization function for performance optimization."

**Solution:**
```javascript
// Basic memoization
function memoize(fn) {
  const cache = {};
  
  return function(...args) {
    const key = JSON.stringify(args);
    
    if (cache[key] !== undefined) {
      console.log('From cache');
      return cache[key];
    }
    
    const result = fn.apply(this, args);
    cache[key] = result;
    return result;
  };
}

// Test with expensive operation
const fibonacci = memoize(function(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
});

console.log(fibonacci(10)); // Computed
console.log(fibonacci(10)); // From cache

// Advanced memoization with WeakMap
function memoizeAdvanced(fn) {
  const cache = new Map();
  
  return function(...args) {
    const key = args[0];
    
    if (cache.has(key)) {
      return cache.get(key);
    }
    
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

// With expiration
function memoizeWithExpiry(fn, ttl = 5000) {
  const cache = new Map();
  
  return function(...args) {
    const key = JSON.stringify(args);
    const cached = cache.get(key);
    
    if (cached && Date.now() - cached.timestamp < ttl) {
      return cached.value;
    }
    
    const result = fn.apply(this, args);
    cache.set(key, { value: result, timestamp: Date.now() });
    return result;
  };
}
```

### Question 8: Function Composition with Async

**Interview Question:** "How do you compose async functions?"

**Solution:**
```javascript
// Async pipe
const pipeAsync = (...fns) => async value => {
  let result = value;
  for (const fn of fns) {
    result = await fn(result);
  }
  return result;
};

// Usage
const fetchUser = async id => {
  const response = await fetch(`/api/users/${id}`);
  return response.json();
};

const enrichUser = async user => {
  const posts = await fetch(`/api/posts?userId=${user.id}`);
  return { ...user, posts: await posts.json() };
};

const formatUser = user => ({
  name: user.name,
  postCount: user.posts.length
});

const getUserSummary = pipeAsync(
  fetchUser,
  enrichUser,
  formatUser
);

getUserSummary(1).then(console.log);

// Async compose
const composeAsync = (...fns) => async value => {
  let result = value;
  for (let i = fns.length - 1; i >= 0; i--) {
    result = await fns[i](result);
  }
  return result;
};

// Parallel async operations
const parallel = (...fns) => async value => {
  const results = await Promise.all(fns.map(fn => fn(value)));
  return results;
};

const getUserData = parallel(
  fetchUser,
  fetchUserPosts,
  fetchUserComments
);
```

## Practical Examples

### Example 1: Data Transformation Pipeline

```javascript
const users = [
  { id: 1, name: 'John', age: 30, active: true },
  { id: 2, name: 'Jane', age: 25, active: false },
  { id: 3, name: 'Bob', age: 35, active: true }
];

// Imperative (how)
const result1 = [];
for (let user of users) {
  if (user.active && user.age > 28) {
    result1.push(user.name.toUpperCase());
  }
}

// Functional (what)
const result2 = users
  .filter(user => user.active)
  .filter(user => user.age > 28)
  .map(user => user.name)
  .map(name => name.toUpperCase());

// Or with pipe
const isActive = user => user.active;
const isOlderThan = age => user => user.age > age;
const getName = user => user.name;
const toUpperCase = str => str.toUpperCase();

const processUsers = pipe(
  filter(isActive),
  filter(isOlderThan(28)),
  map(getName),
  map(toUpperCase)
);

const result3 = processUsers(users);
```

### Example 2: Functional State Management

```javascript
// Reducer pattern (like Redux)
const initialState = {
  count: 0,
  todos: []
};

const actions = {
  INCREMENT: 'INCREMENT',
  ADD_TODO: 'ADD_TODO',
  TOGGLE_TODO: 'TOGGLE_TODO'
};

// Pure reducer function
function reducer(state, action) {
  switch (action.type) {
    case actions.INCREMENT:
      return { ...state, count: state.count + 1 };
    
    case actions.ADD_TODO:
      return {
        ...state,
        todos: [...state.todos, { id: Date.now(), text: action.payload, done: false }]
      };
    
    case actions.TOGGLE_TODO:
      return {
        ...state,
        todos: state.todos.map(todo =>
          todo.id === action.payload
            ? { ...todo, done: !todo.done }
            : todo
        )
      };
    
    default:
      return state;
  }
}

// Usage
let state = initialState;
state = reducer(state, { type: actions.INCREMENT });
state = reducer(state, { type: actions.ADD_TODO, payload: 'Learn FP' });
console.log(state);
```

### Example 3: Transducer Pattern

```javascript
// Transducers - composable, efficient transformations
const map = fn => reducer => (acc, value) =>
  reducer(acc, fn(value));

const filter = predicate => reducer => (acc, value) =>
  predicate(value) ? reducer(acc, value) : acc;

const transduce = (xform, reducer, initial, collection) => {
  const transformedReducer = xform(reducer);
  return collection.reduce(transformedReducer, initial);
};

// Compose transformations
const double = x => x * 2;
const isEven = x => x % 2 === 0;
const append = (acc, value) => [...acc, value];

const xform = compose(
  map(double),
  filter(isEven)
);

const numbers = [1, 2, 3, 4, 5];
const result = transduce(xform, append, [], numbers);
console.log(result); // [4, 8] (doubled evens)
```

## What Interviewers Look For

### 1. Core Understanding
- Pure functions definition and benefits
- Immutability principles
- Higher-order functions usage
- Function composition concepts

### 2. Practical Skills
- Can refactor imperative to functional
- Knows array methods (map/filter/reduce)
- Understands when FP is appropriate
- Can write pure, testable code

### 3. Advanced Knowledge
- Currying and partial application
- Memoization for optimization
- Function composition patterns
- Transducers and advanced patterns

### 4. Trade-offs Understanding
- Performance considerations
- Readability vs purity
- When to use FP vs OOP
- Debugging functional code

### 5. Real-World Application
- State management (Redux pattern)
- Data transformation pipelines
- API client design
- Event handling

## Red Flags to Avoid

### 1. Misunderstandings
- Thinking FP means no loops ever
- Confusing immutability with constants
- Not understanding side effects
- Believing FP is always better

### 2. Common Mistakes
- Mutating arrays/objects accidentally
- Overusing reduce for everything
- Creating impure functions unknowingly
- Not handling edge cases in pure functions

### 3. Performance Issues
- Excessive copying of large structures
- Not using memoization when beneficial
- Creating too many intermediate arrays
- Not considering memory usage

### 4. Code Quality
- Unreadable nested compositions
- Over-engineering simple problems
- Mixing paradigms inconsistently
- Not testing pure functions

## Key Takeaways

### Essential Concepts
1. **Pure functions**: Same input → same output, no side effects
2. **Immutability**: Create new data instead of mutating
3. **Higher-order functions**: Functions as first-class citizens
4. **Composition**: Build complex from simple
5. **Declarative**: Express what, not how

### Best Practices
1. **Default to pure**: Make functions pure when possible
2. **Avoid mutations**: Use spread, map, filter instead of push, splice
3. **Compose small functions**: Build complexity gradually
4. **Separate data and behavior**: Functions operate on data
5. **Test pure functions**: Easy to test, no mocks needed

### Common Patterns
1. **map/filter/reduce**: Core transformation toolkit
2. **Pipe/compose**: Function composition
3. **Curry/partial**: Function configuration
4. **Memoization**: Performance optimization
5. **Reducer pattern**: State management

### Interview Success Tips
1. **Start with definitions**: Pure functions, immutability
2. **Show refactoring**: Imperative → functional
3. **Discuss trade-offs**: When FP helps, when it doesn't
4. **Demonstrate composition**: Build complex from simple
5. **Mention real-world use**: Redux, React hooks, RxJS
6. **Write tests**: Pure functions are easily testable

Remember: Functional programming is a tool, not a religion. Use FP principles to write more predictable, testable code, but balance with readability and performance. JavaScript is multi-paradigm—leverage functional concepts where they provide clear benefits.
