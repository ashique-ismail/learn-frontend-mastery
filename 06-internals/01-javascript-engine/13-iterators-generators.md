# Iterators and Generators

## The Idea

**In plain English:** Iterators and generators are tools that let you go through a list of items one at a time, pausing between each one instead of loading everything at once. A generator is a special kind of function that can hand you one value, wait, and then pick up right where it left off when you ask for the next one.

**Real-world analogy:** Think of a vending machine that dispenses snacks one at a time. You press the button, it drops one snack and stops. You press it again, it drops another. It does not pour every snack out at once.

- The vending machine = the generator function (holds all possible values but releases them on demand)
- Pressing the button = calling `.next()` (asking for the next value)
- The snack that drops out = the `yield`ed value returned to you
- The machine running out of stock = `done: true` (no more values to give)

---

## Table of Contents
- [Overview](#overview)
- [Iterator Protocol](#iterator-protocol)
- [Iterable Protocol](#iterable-protocol)
- [Generator Functions](#generator-functions)
- [Async Iterators](#async-iterators)
- [Async Generators](#async-generators)
- [Advanced Patterns](#advanced-patterns)
- [Common Misconceptions](#common-misconceptions)
- [Performance Implications](#performance-implications)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Overview

Iterators and generators provide a standardized way to iterate over sequences of values. Iterators define how to access items sequentially, while generators provide a concise syntax for creating iterators. Understanding these concepts is essential for working with modern JavaScript features like for-of loops, spread operators, and async iteration.

## Iterator Protocol

The iterator protocol defines a standard way to produce a sequence of values.

### Basic Iterator

```javascript
// Iterator protocol: Object with next() method
// next() returns { value, done }

const iterator = {
  current: 0,
  last: 5,
  
  next() {
    if (this.current <= this.last) {
      return { value: this.current++, done: false };
    }
    return { done: true };
  }
};

console.log(iterator.next()); // { value: 0, done: false }
console.log(iterator.next()); // { value: 1, done: false }
console.log(iterator.next()); // { value: 2, done: false }
// ... continues until done: true
```

### Manual Iteration

```javascript
function createRangeIterator(start, end) {
  let current = start;
  
  return {
    next() {
      if (current <= end) {
        return { value: current++, done: false };
      }
      return { value: undefined, done: true };
    }
  };
}

const iter = createRangeIterator(1, 3);

let result = iter.next();
while (!result.done) {
  console.log(result.value); // 1, 2, 3
  result = iter.next();
}
```

### Return and Throw Methods

```javascript
// Iterators can optionally have return() and throw() methods
const iterator = {
  values: [1, 2, 3, 4, 5],
  index: 0,
  
  next() {
    if (this.index < this.values.length) {
      return { value: this.values[this.index++], done: false };
    }
    return { done: true };
  },
  
  return(value) {
    // Called when iteration is stopped early
    console.log('Iterator closed early');
    this.index = this.values.length; // Reset state
    return { value, done: true };
  },
  
  throw(error) {
    // Called to throw an error into iterator
    console.log('Error thrown');
    throw error;
  }
};

// return() called when breaking early
for (const val of { [Symbol.iterator]: () => iterator }) {
  console.log(val);
  if (val === 2) break; // Triggers return()
}
// Logs: 1, 2, 'Iterator closed early'
```

## Iterable Protocol

The iterable protocol defines how objects become iterable (usable with for-of, spread, etc.).

### Making Objects Iterable

```javascript
// Iterable protocol: Object with [Symbol.iterator] method
// [Symbol.iterator]() returns iterator

const range = {
  start: 1,
  end: 5,
  
  [Symbol.iterator]() {
    let current = this.start;
    const last = this.end;
    
    return {
      next() {
        if (current <= last) {
          return { value: current++, done: false };
        }
        return { done: true };
      }
    };
  }
};

// Can use with for-of
for (const num of range) {
  console.log(num); // 1, 2, 3, 4, 5
}

// Can spread
console.log([...range]); // [1, 2, 3, 4, 5]

// Can destructure
const [first, second] = range;
console.log(first, second); // 1, 2
```

### Built-in Iterables

```javascript
// Arrays
const arr = [1, 2, 3];
for (const val of arr) {
  console.log(val); // 1, 2, 3
}

// Strings
const str = 'Hello';
for (const char of str) {
  console.log(char); // 'H', 'e', 'l', 'l', 'o'
}

// Maps
const map = new Map([['a', 1], ['b', 2]]);
for (const [key, value] of map) {
  console.log(key, value); // 'a' 1, 'b' 2
}

// Sets
const set = new Set([1, 2, 3]);
for (const val of set) {
  console.log(val); // 1, 2, 3
}

// Typed Arrays
const uint8 = new Uint8Array([1, 2, 3]);
for (const val of uint8) {
  console.log(val); // 1, 2, 3
}

// Arguments object
function test() {
  for (const arg of arguments) {
    console.log(arg);
  }
}
test(1, 2, 3); // 1, 2, 3

// NodeLists
// for (const element of document.querySelectorAll('div')) { ... }
```

### Custom Iterator for Class

```javascript
class Range {
  constructor(start, end) {
    this.start = start;
    this.end = end;
  }
  
  [Symbol.iterator]() {
    let current = this.start;
    const last = this.end;
    
    return {
      next() {
        if (current <= last) {
          return { value: current++, done: false };
        }
        return { done: true };
      }
    };
  }
}

const range = new Range(1, 5);

for (const num of range) {
  console.log(num); // 1, 2, 3, 4, 5
}

console.log(Math.max(...range)); // 5
```

## Generator Functions

Generators provide a concise syntax for creating iterators using function*.

### Basic Generator

```javascript
// Generator function (function*)
function* countTo(n) {
  for (let i = 1; i <= n; i++) {
    yield i; // Pause and return value
  }
}

const gen = countTo(3);

console.log(gen.next()); // { value: 1, done: false }
console.log(gen.next()); // { value: 2, done: false }
console.log(gen.next()); // { value: 3, done: false }
console.log(gen.next()); // { value: undefined, done: true }

// Or use with for-of
for (const num of countTo(3)) {
  console.log(num); // 1, 2, 3
}
```

### Generator Execution

```javascript
function* example() {
  console.log('Started');
  yield 1;
  console.log('After first yield');
  yield 2;
  console.log('After second yield');
  yield 3;
  console.log('Done');
}

const gen = example();

console.log('Created generator');
console.log(gen.next()); // Logs: 'Started', Returns: { value: 1, done: false }
console.log(gen.next()); // Logs: 'After first yield', Returns: { value: 2, done: false }
console.log(gen.next()); // Logs: 'After second yield', Returns: { value: 3, done: false }
console.log(gen.next()); // Logs: 'Done', Returns: { value: undefined, done: true }
```

### yield Expression Value

```javascript
function* example() {
  const a = yield 1; // yield returns value passed to next()
  console.log('Received:', a);
  
  const b = yield 2;
  console.log('Received:', b);
  
  return 'done';
}

const gen = example();

console.log(gen.next());      // { value: 1, done: false }
console.log(gen.next('foo')); // Logs: 'Received: foo', Returns: { value: 2, done: false }
console.log(gen.next('bar')); // Logs: 'Received: bar', Returns: { value: 'done', done: true }
```

### Generator with Iterator

```javascript
class Range {
  constructor(start, end) {
    this.start = start;
    this.end = end;
  }
  
  // Generator as iterator (cleaner than manual iterator)
  *[Symbol.iterator]() {
    for (let i = this.start; i <= this.end; i++) {
      yield i;
    }
  }
}

const range = new Range(1, 5);
console.log([...range]); // [1, 2, 3, 4, 5]
```

### yield*

Delegation

```javascript
// yield* delegates to another iterable
function* inner() {
  yield 2;
  yield 3;
}

function* outer() {
  yield 1;
  yield* inner(); // Delegate to inner generator
  yield 4;
}

console.log([...outer()]); // [1, 2, 3, 4]

// yield* works with any iterable
function* gen() {
  yield* [1, 2, 3]; // Delegate to array
  yield* 'abc';     // Delegate to string
}

console.log([...gen()]); // [1, 2, 3, 'a', 'b', 'c']
```

### Generator Return and Throw

```javascript
function* example() {
  try {
    yield 1;
    yield 2;
    yield 3;
  } catch (error) {
    console.log('Caught:', error.message);
  }
}

const gen = example();

console.log(gen.next());  // { value: 1, done: false }
console.log(gen.throw(new Error('Oops'))); // Logs: 'Caught: Oops', Returns: { value: undefined, done: true }

// return() ends generator early
function* gen2() {
  yield 1;
  yield 2;
  yield 3;
}

const g = gen2();
console.log(g.next());        // { value: 1, done: false }
console.log(g.return('early')); // { value: 'early', done: true }
console.log(g.next());        // { value: undefined, done: true }
```

### Infinite Sequences

```javascript
// Generators enable infinite sequences
function* naturalNumbers() {
  let n = 1;
  while (true) {
    yield n++;
  }
}

const numbers = naturalNumbers();
console.log(numbers.next().value); // 1
console.log(numbers.next().value); // 2
console.log(numbers.next().value); // 3

// Fibonacci sequence
function* fibonacci() {
  let [a, b] = [0, 1];
  while (true) {
    yield a;
    [a, b] = [b, a + b];
  }
}

// Take first 10 Fibonacci numbers
function* take(n, iterable) {
  let count = 0;
  for (const value of iterable) {
    if (count++ >= n) return;
    yield value;
  }
}

console.log([...take(10, fibonacci())]); // [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

## Async Iterators

Async iterators handle asynchronous sequences.

### Async Iterator Protocol

```javascript
// Async iterator: next() returns Promise<{value, done}>
const asyncIterator = {
  current: 0,
  
  async next() {
    // Simulate async operation
    await new Promise(resolve => setTimeout(resolve, 100));
    
    if (this.current < 3) {
      return { value: this.current++, done: false };
    }
    return { done: true };
  }
};

// Manual consumption
(async () => {
  let result = await asyncIterator.next();
  while (!result.done) {
    console.log(result.value);
    result = await asyncIterator.next();
  }
})();
```

### Async Iterable Protocol

```javascript
// Async iterable: [Symbol.asyncIterator]() returns async iterator
const asyncIterable = {
  [Symbol.asyncIterator]() {
    let i = 0;
    
    return {
      async next() {
        await new Promise(resolve => setTimeout(resolve, 100));
        
        if (i < 3) {
          return { value: i++, done: false };
        }
        return { done: true };
      }
    };
  }
};

// Use with for-await-of
(async () => {
  for await (const value of asyncIterable) {
    console.log(value); // 0, 1, 2 (with 100ms delay each)
  }
})();
```

### Async Iterator Example: Pagination

```javascript
class PaginatedAPI {
  constructor(baseURL) {
    this.baseURL = baseURL;
  }
  
  [Symbol.asyncIterator]() {
    let page = 1;
    let done = false;
    
    return {
      async next() {
        if (done) {
          return { done: true };
        }
        
        // Simulate API call
        const response = await fetch(`${this.baseURL}?page=${page}`);
        const data = await response.json();
        
        page++;
        
        if (data.length === 0) {
          done = true;
          return { done: true };
        }
        
        return { value: data, done: false };
      }
    };
  }
}

// Usage
const api = new PaginatedAPI('https://api.example.com/users');

(async () => {
  for await (const users of api) {
    console.log('Page of users:', users);
  }
})();
```

## Async Generators

Async generators combine generators with async/await.

### Basic Async Generator

```javascript
async function* asyncCount(n) {
  for (let i = 1; i <= n; i++) {
    await new Promise(resolve => setTimeout(resolve, 100));
    yield i;
  }
}

// Use with for-await-of
(async () => {
  for await (const num of asyncCount(3)) {
    console.log(num); // 1, 2, 3 (with delays)
  }
})();

// Or manual consumption
(async () => {
  const gen = asyncCount(3);
  
  console.log(await gen.next()); // { value: 1, done: false }
  console.log(await gen.next()); // { value: 2, done: false }
  console.log(await gen.next()); // { value: 3, done: false }
  console.log(await gen.next()); // { value: undefined, done: true }
})();
```

### Real-World Example: Streaming Data

```javascript
async function* fetchPages(url) {
  let page = 1;
  
  while (true) {
    const response = await fetch(`${url}?page=${page}`);
    const data = await response.json();
    
    if (data.length === 0) break;
    
    yield data;
    page++;
  }
}

// Usage
(async () => {
  for await (const pageData of fetchPages('https://api.example.com/items')) {
    console.log('Processing page:', pageData);
    // Process each page as it arrives
  }
})();
```

### Async Generator Pipeline

```javascript
// Transform async sequences
async function* map(iterable, transform) {
  for await (const value of iterable) {
    yield transform(value);
  }
}

async function* filter(iterable, predicate) {
  for await (const value of iterable) {
    if (await predicate(value)) {
      yield value;
    }
  }
}

async function* take(n, iterable) {
  let count = 0;
  for await (const value of iterable) {
    if (count++ >= n) break;
    yield value;
  }
}

// Example usage
async function* numbers() {
  let i = 1;
  while (true) {
    await new Promise(resolve => setTimeout(resolve, 10));
    yield i++;
  }
}

(async () => {
  const pipeline = take(
    5,
    filter(
      map(numbers(), x => x * 2),
      x => x % 3 === 0
    )
  );
  
  for await (const value of pipeline) {
    console.log(value); // First 5 numbers divisible by 3, doubled
  }
})();
```

### Error Handling in Async Generators

```javascript
async function* fetchWithRetry(urls) {
  for (const url of urls) {
    let retries = 3;
    
    while (retries > 0) {
      try {
        const response = await fetch(url);
        const data = await response.json();
        yield { url, data, success: true };
        break;
      } catch (error) {
        retries--;
        
        if (retries === 0) {
          yield { url, error: error.message, success: false };
        } else {
          await new Promise(resolve => setTimeout(resolve, 1000));
        }
      }
    }
  }
}

// Usage
(async () => {
  const urls = ['https://api1.com', 'https://api2.com'];
  
  for await (const result of fetchWithRetry(urls)) {
    if (result.success) {
      console.log('Success:', result.data);
    } else {
      console.error('Failed:', result.error);
    }
  }
})();
```

## Advanced Patterns

### Observer Pattern with Generators

```javascript
function* observableGenerator() {
  const listeners = [];
  
  function emit(value) {
    listeners.forEach(listener => listener(value));
  }
  
  function subscribe(listener) {
    listeners.push(listener);
    return () => {
      const index = listeners.indexOf(listener);
      if (index > -1) listeners.splice(index, 1);
    };
  }
  
  yield { emit, subscribe };
}

// Usage
const gen = observableGenerator();
const { value: observable } = gen.next();

const unsubscribe = observable.subscribe(value => {
  console.log('Received:', value);
});

observable.emit('Hello'); // Logs: 'Received: Hello'
observable.emit('World'); // Logs: 'Received: World'

unsubscribe();
observable.emit('Ignored'); // Not logged
```

### Lazy Evaluation

```javascript
function* map(iterable, transform) {
  for (const value of iterable) {
    yield transform(value);
  }
}

function* filter(iterable, predicate) {
  for (const value of iterable) {
    if (predicate(value)) {
      yield value;
    }
  }
}

function* range(start, end) {
  for (let i = start; i <= end; i++) {
    console.log(`Generating ${i}`);
    yield i;
  }
}

// Lazy: Operations only execute when values are consumed
const pipeline = filter(
  map(range(1, 10), x => x * 2),
  x => x % 3 === 0
);

console.log('Pipeline created');
// Nothing logged yet!

for (const value of pipeline) {
  console.log('Result:', value);
  if (value >= 12) break; // Early exit, doesn't generate all values
}
```

### State Machines

```javascript
function* trafficLight() {
  while (true) {
    yield 'red';
    yield 'yellow';
    yield 'green';
  }
}

const light = trafficLight();

console.log(light.next().value); // 'red'
console.log(light.next().value); // 'yellow'
console.log(light.next().value); // 'green'
console.log(light.next().value); // 'red' (cycles)
```

### Coroutines

```javascript
function* task1() {
  console.log('Task 1: Start');
  yield;
  console.log('Task 1: Continue');
  yield;
  console.log('Task 1: End');
}

function* task2() {
  console.log('Task 2: Start');
  yield;
  console.log('Task 2: Continue');
  yield;
  console.log('Task 2: End');
}

function* scheduler(tasks) {
  while (tasks.some(t => !t.done)) {
    for (const task of tasks) {
      const result = task.next();
      if (!result.done) {
        yield; // Give control back to scheduler
      }
    }
  }
}

// Run tasks cooperatively
const tasks = [task1(), task2()];
const sched = scheduler(tasks);

while (!sched.next().done) {
  // Tasks interleave execution
}
```

## Common Misconceptions

### Misconception 1: "Generators Execute Immediately"

**Reality**: Generator functions return an iterator; execution starts on first next() call.

```javascript
function* gen() {
  console.log('Executing');
  yield 1;
}

const g = gen(); // Nothing logged
console.log('Created');
g.next(); // Now logs: 'Executing'
```

### Misconception 2: "yield Stops Forever"

**Reality**: yield pauses execution; next() resumes from that point.

```javascript
function* gen() {
  console.log('Before yield');
  yield 1;
  console.log('After yield');
  yield 2;
}

const g = gen();
g.next(); // Logs: 'Before yield'
g.next(); // Logs: 'After yield'
```

### Misconception 3: "Generators Are Just for Iteration"

**Reality**: Generators enable coroutines, state machines, lazy evaluation, async flow control, and more.

```javascript
// Example: Async flow control (pre-async/await)
function* fetchUser(id) {
  const user = yield fetch(`/api/users/${id}`);
  const posts = yield fetch(`/api/posts?user=${user.id}`);
  return { user, posts };
}

// Runner (simplified)
function run(generator) {
  const gen = generator();
  
  function step(value) {
    const result = gen.next(value);
    
    if (result.done) return result.value;
    
    result.value.then(step);
  }
  
  step();
}
```

### Misconception 4: "for-await-of Works with Regular Iterables"

**Reality**: for-await-of requires async iterables (Symbol.asyncIterator), though it can handle sync iterables that yield promises.

```javascript
// Regular iterable with promises
const iterable = {
  *[Symbol.iterator]() {
    yield Promise.resolve(1);
    yield Promise.resolve(2);
  }
};

(async () => {
  for await (const value of iterable) {
    console.log(value); // 1, 2 (awaits each promise)
  }
})();
```

## Performance Implications

### Generator Overhead

```javascript
// Generator (slightly slower creation)
function* genRange(n) {
  for (let i = 0; i < n; i++) {
    yield i;
  }
}

// Array (faster for small sequences)
function arrayRange(n) {
  return Array.from({ length: n }, (_, i) => i);
}

// Generators: Lazy, memory-efficient for large/infinite sequences
// Arrays: Eager, faster for small sequences that fit in memory

// Large sequence: Generator wins
console.time('generator');
for (const n of genRange(1000000)) {
  if (n > 100) break; // Only generates 101 values
}
console.timeEnd('generator'); // Fast

console.time('array');
for (const n of arrayRange(1000000)) {
  if (n > 100) break; // But created all 1M values first
}
console.timeEnd('array'); // Slower, more memory
```

### Lazy vs Eager

```javascript
// Eager (array methods)
const eager = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
  .map(x => x * 2)        // Processes all 10
  .filter(x => x > 5)     // Processes all 10 again
  .slice(0, 3);           // Takes first 3

// Lazy (generators)
function* lazy() {
  for (let i = 1; i <= 10; i++) {
    const doubled = i * 2;
    if (doubled > 5) {
      yield doubled;
    }
  }
}

const result = [];
for (const value of lazy()) {
  result.push(value);
  if (result.length >= 3) break; // Stops early
}

// Lazy evaluation more efficient when early exit possible
```

## Interview Questions

### Question 1: Explain the iterator protocol

**Answer**: The iterator protocol defines that an iterator must have a `next()` method that returns an object with `value` (current item) and `done` (boolean indicating if iteration complete). Optionally, iterators can have `return()` (cleanup when stopped early) and `throw()` (error injection) methods. This standardized interface enables iteration over custom data structures.

### Question 2: What's the difference between iterators and iterables?

**Answer**: An iterable is an object with a `[Symbol.iterator]` method that returns an iterator. An iterator is an object with a `next()` method. Arrays are iterables (have `[Symbol.iterator]`); calling it returns an iterator. You can iterate over iterables with for-of, spread, destructuring. Some objects are both (generators are iterators that are also iterable).

### Question 3: How do generators work internally?

**Answer**: Generators return a generator object that is both iterator and iterable. When created, the function body doesn't execute. Each `next()` call executes until the next `yield`, pausing execution and returning yielded value. The generator maintains execution context between calls, allowing state preservation. This is implemented via a state machine internally.

### Question 4: What are async generators and when would you use them?

**Answer**: Async generators (`async function*`) combine generators with async/await. They yield promises and can await inside. Use for asynchronous sequences: paginated APIs, streaming data, real-time events. They enable `for-await-of` loops and lazy async transformations. Better than promise arrays for large/infinite async sequences since values are produced on-demand.

### Question 5: Explain yield* delegation

**Answer**: `yield*` delegates to another iterable/generator. It yields all values from the delegated iterable before continuing. Useful for composition: combining generators, flattening nested structures, recursion. `yield* iterator` is equivalent to `for (const value of iterator) yield value`, but more efficient and preserves the delegation relationship.

## Key Takeaways

1. **Iterator protocol**: Objects with next() method returning {value, done}

2. **Iterable protocol**: Objects with [Symbol.iterator] method returning iterator

3. **Generators simplify iterators**: function* and yield provide clean syntax

4. **Lazy evaluation**: Generators produce values on-demand, memory-efficient

5. **Async iterators**: Handle asynchronous sequences with Symbol.asyncIterator

6. **Async generators**: Combine async/await with generators for async sequences

7. **yield* delegation**: Compose iterators, flatten structures

8. **Infinite sequences**: Generators enable infinite sequences safely

9. **Performance trade-offs**: Lazy generators vs eager arrays depends on use case

10. **Powerful patterns**: State machines, coroutines, pipelines, streaming

## Resources

### Official Documentation
- [MDN: Iterators and Generators](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Iterators_and_Generators)
- [MDN: Iteration protocols](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols)
- [MDN: Async iteration](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for-await...of)

### Articles
- "ES6 Generators in Depth" by Pony Foo
- "The Hidden Power of ES6 Generators" by Kyle Simpson
- "Async Iterators and Generators" by Axel Rauschmayer

### Books
- "You Don't Know JS: ES6 & Beyond" by Kyle Simpson
- "Exploring ES6" by Axel Rauschmayer
- "JavaScript: The Definitive Guide" by David Flanagan

### Videos
- "JavaScript Generators" by Fun Fun Function
- "Async Iterators in JavaScript" by JavaScript conferences
- "Generator Functions" by Traversy Media
