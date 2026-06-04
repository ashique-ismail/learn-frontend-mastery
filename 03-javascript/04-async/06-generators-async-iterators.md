# Generators and Async Iterators

## Overview

Generators and async iterators are two related but distinct JavaScript features that provide lazy, pull-based data production. Where Promises push values to consumers, generators let consumers pull values from producers — enabling lazy evaluation, infinite sequences, custom iteration protocols, and cooperative multitasking.

Generators (ES2015) are functions that can pause and resume execution, yielding values on demand. Async generators (ES2018) combine generators with async/await, enabling lazy streams of asynchronous data. The `for...of` and `for await...of` loops are the primary consumers of these constructs.

## Generators

### Generator Functions

A generator function is declared with `function*` and can contain `yield` expressions:

```javascript
function* counter() {
  let i = 0;
  while (true) {
    yield i++;
  }
}

const gen = counter();

console.log(gen.next()); // { value: 0, done: false }
console.log(gen.next()); // { value: 1, done: false }
console.log(gen.next()); // { value: 2, done: false }
// Infinite — never returns { done: true }
```

Calling a generator function does NOT execute its body — it returns a Generator object (an iterator). Execution begins only when `.next()` is called.

### Generator Execution Model

```javascript
function* greet() {
  console.log('1 - before first yield');
  const input = yield 'first';           // Pauses here, yields 'first'
  console.log('2 - received:', input);
  yield 'second';                         // Pauses here, yields 'second'
  console.log('3 - after second yield');
  return 'done';                          // Generator completes
}

const gen = greet();

console.log(gen.next());         // Logs '1 - before first yield', returns { value: 'first', done: false }
console.log(gen.next('hello'));  // Logs '2 - received: hello', returns { value: 'second', done: false }
console.log(gen.next());         // Logs '3 - after second yield', returns { value: 'done', done: true }
console.log(gen.next());         // { value: undefined, done: true } — generator exhausted
```

The argument passed to `.next(value)` becomes the result of the `yield` expression inside the generator. The first `.next()` call cannot pass a value (there is no `yield` to receive it yet).

### Finite Generators

```javascript
function* range(start, end, step = 1) {
  for (let i = start; i < end; i += step) {
    yield i;
  }
}

for (const n of range(0, 10, 2)) {
  console.log(n); // 0, 2, 4, 6, 8
}

// Spread operator works too
console.log([...range(1, 6)]); // [1, 2, 3, 4, 5]
```

### Generator Return and Throw

```javascript
const gen = counter();
gen.next();            // { value: 0, done: false }
gen.return('goodbye'); // { value: 'goodbye', done: true } — terminates early

const gen2 = greet();
gen2.next();
gen2.throw(new Error('abort')); // Throws inside generator at current yield point
// If generator doesn't catch it, the error propagates to the caller
```

### yield*: Delegating to Another Iterable

```javascript
function* concat(...iterables) {
  for (const iterable of iterables) {
    yield* iterable; // Delegates to sub-iterable
  }
}

console.log([...concat([1, 2], [3, 4], [5])]); // [1, 2, 3, 4, 5]

// yield* also works with other generators
function* flatten(arr) {
  for (const item of arr) {
    if (Array.isArray(item)) {
      yield* flatten(item); // Recursive delegation
    } else {
      yield item;
    }
  }
}

console.log([...flatten([1, [2, [3, 4]], 5])]); // [1, 2, 3, 4, 5]
```

## The Iteration Protocol

Generators implement the Iterator and Iterable protocols, which is why they work with `for...of`, spread, and destructuring:

```javascript
// An object is iterable if it has Symbol.iterator returning an iterator
// An iterator has a .next() method returning { value, done }

const myIterable = {
  [Symbol.iterator]() {
    let n = 0;
    return {
      next() {
        return n < 3
          ? { value: n++, done: false }
          : { value: undefined, done: true };
      }
    };
  }
};

for (const v of myIterable) console.log(v); // 0, 1, 2

// Generators implement both protocols:
function* gen() { yield 1; yield 2; }
const g = gen();
console.log(g[Symbol.iterator]() === g); // true — generator is its own iterator
```

## Practical Generator Use Cases

### Infinite Sequences

```javascript
function* fibonacci() {
  let [a, b] = [0, 1];
  while (true) {
    yield a;
    [a, b] = [b, a + b];
  }
}

function take(n, iterable) {
  const result = [];
  for (const value of iterable) {
    result.push(value);
    if (result.length >= n) break;
  }
  return result;
}

console.log(take(8, fibonacci())); // [0, 1, 1, 2, 3, 5, 8, 13]
```

### Lazy Evaluation / Infinite Lists

```javascript
function* naturals() {
  let n = 1;
  while (true) yield n++;
}

function* map(fn, iterable) {
  for (const v of iterable) yield fn(v);
}

function* filter(pred, iterable) {
  for (const v of iterable) if (pred(v)) yield v;
}

function* take(n, iterable) {
  let count = 0;
  for (const v of iterable) {
    if (count++ >= n) return;
    yield v;
  }
}

// First 5 even squares — no array ever created for the full sequence
const result = [...take(5, filter(
  n => n % 2 === 0,
  map(n => n * n, naturals())
))];
console.log(result); // [4, 16, 36, 64, 100]
```

### State Machines

```javascript
function* trafficLight() {
  while (true) {
    yield 'green';
    yield 'yellow';
    yield 'red';
  }
}

const light = trafficLight();
const states = Array.from({ length: 7 }, () => light.next().value);
console.log(states); // ['green', 'yellow', 'red', 'green', 'yellow', 'red', 'green']
```

### Generators as Coroutines (Pre-async/await)

Before `async/await`, libraries like `co` used generators for async control flow:

```javascript
// This is how co.js worked
function run(generatorFn) {
  const gen = generatorFn();

  function step(nextValue) {
    const { value, done } = gen.next(nextValue);
    if (done) return;
    Promise.resolve(value).then(step, err => gen.throw(err));
  }

  step();
}

// Usage resembled async/await
run(function* () {
  const user = yield fetchUser(id);     // yield a Promise
  const posts = yield fetchPosts(user.id);
  console.log(posts);
});
```

This pattern is the historical basis for how `async/await` was designed.

## Async Generators

Async generators combine `async function*` syntax. They yield Promises implicitly and can use `await` inside:

```javascript
async function* paginate(fetchPage, startPage = 1) {
  let page = startPage;

  while (true) {
    const result = await fetchPage(page);
    yield result.items;

    if (!result.hasNextPage) return;
    page++;
  }
}

// Consumer
for await (const items of paginate(fetchProducts)) {
  console.log('Got page of', items.length, 'products');
  renderItems(items);
}
```

### Async Generator Return Values

```javascript
async function* asyncRange(start, end) {
  for (let i = start; i <= end; i++) {
    await delay(100); // Can await inside
    yield i;
  }
}

for await (const n of asyncRange(1, 5)) {
  console.log(n); // 1, 2, 3, 4, 5 with 100ms delays
}
```

### Reading a Node.js Stream with Async Iteration

```javascript
const fs = require('fs');

async function* readLines(filePath) {
  const stream = fs.createReadStream(filePath, { encoding: 'utf8' });
  let buffer = '';

  for await (const chunk of stream) {
    buffer += chunk;
    const lines = buffer.split('\n');
    buffer = lines.pop(); // Keep incomplete last line

    for (const line of lines) {
      yield line;
    }
  }

  if (buffer) yield buffer; // Yield any remaining partial line
}

for await (const line of readLines('large-file.txt')) {
  processLine(line);
}
```

### Implementing an Async Queue as an Async Iterable

```javascript
class AsyncQueue {
  #queue = [];
  #resolvers = [];
  #closed = false;

  enqueue(value) {
    if (this.#resolvers.length > 0) {
      this.#resolvers.shift()({ value, done: false });
    } else {
      this.#queue.push(value);
    }
  }

  close() {
    this.#closed = true;
    for (const resolve of this.#resolvers) {
      resolve({ value: undefined, done: true });
    }
    this.#resolvers = [];
  }

  [Symbol.asyncIterator]() {
    return {
      next: () => {
        if (this.#queue.length > 0) {
          return Promise.resolve({ value: this.#queue.shift(), done: false });
        }
        if (this.#closed) {
          return Promise.resolve({ value: undefined, done: true });
        }
        return new Promise(resolve => this.#resolvers.push(resolve));
      }
    };
  }
}

// Usage
const queue = new AsyncQueue();

(async () => {
  for await (const message of queue) {
    console.log('Received:', message);
  }
})();

queue.enqueue('hello');
queue.enqueue('world');
setTimeout(() => queue.close(), 2000);
```

## for await...of

Designed for consuming async iterables — it awaits each item before proceeding to the next:

```javascript
async function processAsync(asyncIterable) {
  for await (const item of asyncIterable) {
    // Awaits each item's Promise before continuing
    await process(item);
  }
}
```

### Difference from for...of on Promises

```javascript
const promises = [fetchA(), fetchB(), fetchC()];

// for...of iterates synchronously — does NOT await
for (const p of promises) {
  const result = await p; // These still run sequentially
}

// for await...of is for async iterables (Symbol.asyncIterator)
// On a regular array, it behaves the same as above
for await (const p of promises) {
  // Works but equivalent to the above for arrays
}
```

The real purpose of `for await...of` is consuming async generators and objects implementing `Symbol.asyncIterator` — not awaiting arrays of Promises (use `Promise.all` for that).

## Comparison Table

| Feature | Sync Generator | Async Generator | Array | Observable |
|---------|---------------|----------------|-------|-----------|
| Lazy evaluation | Yes | Yes | No | Yes |
| Async values | No | Yes | Via .map() | Yes |
| Cancellable | return() | return() | No | Yes (subscription) |
| Push vs Pull | Pull | Pull | Pull | Push |
| Multiple consumers | No (exhausted) | No | Yes | Yes (multicast) |
| Backpressure | Natural | Natural | No | Via operators |
| Browser native | Yes | Yes | Yes | No (RxJS) |

## Best Practices

### 1. Use for await...of with async Generators, Not .next() Manually

Manual calls to `gen.next()` in async code are error-prone. `for await...of` handles the iteration protocol, cleanup, and early return:

```javascript
// Prefer
for await (const item of asyncGen()) {
  process(item);
}

// Over manual iteration
const gen = asyncGen();
while (true) {
  const { value, done } = await gen.next();
  if (done) break;
  process(value);
}
```

### 2. Handle Cleanup with try/finally in Generators

```javascript
async function* withCleanup(resource) {
  try {
    yield* processResource(resource);
  } finally {
    await resource.close(); // Runs even if consumer breaks early
  }
}
```

When `for await...of` breaks or throws, the runtime calls `gen.return()`, which triggers the `finally` block.

### 3. Use Generators for Laziness, Not Just Syntax

The value of generators is deferred computation — don't use them when you need all results eagerly anyway. Use arrays and `Promise.all` for that.

### 4. Combine with AbortSignal for Cancellable Iteration

```javascript
async function* fetchPages(url, signal) {
  let page = 1;
  while (true) {
    signal.throwIfAborted();
    const data = await fetch(`${url}?page=${page}`, { signal }).then(r => r.json());
    yield data.items;
    if (!data.hasMore) return;
    page++;
  }
}
```

## Interview Questions

### Q1: What is the difference between a generator and a regular function?

**Answer:** A regular function runs to completion when called, returning a single value. A generator function returns a Generator object immediately (without executing the body). The body executes lazily on each `.next()` call, pausing at each `yield`. Generators can produce multiple values over time and maintain internal state between calls.

### Q2: What does yield* do?

**Answer:** `yield*` delegates to another iterable (array, string, another generator, etc.), yielding each of its values in turn. It is equivalent to looping over the sub-iterable and yielding each value individually. It also forwards the `return` value of a delegated generator (the value paired with `done: true`).

### Q3: How does the co.js library work, and why was it replaced by async/await?

**Answer:** `co` takes a generator function and runs it, intercepting each yielded Promise, waiting for it to resolve, then resuming the generator with the resolved value via `.next(value)`. Errors caused the generator to throw via `.throw(err)`. This mimicked the behavior now built into `async/await`. It was replaced because `async/await` is a native syntax transform with better engine optimization, clearer semantics, and no dependency on a library.

### Q4: What is the Async Iterable protocol?

**Answer:** An object is an async iterable if it has a `[Symbol.asyncIterator]()` method that returns an async iterator. An async iterator has a `.next()` method that returns a Promise resolving to `{ value, done }`. `for await...of` consumes async iterables by repeatedly calling `.next()` and awaiting the result. Native async iterables include: async generators, Node.js Readable streams (Node 10+), and `ReadableStream` from the Streams API (with `[Symbol.asyncIterator]`).

### Q5: When would you use an async generator over Promise.all?

**Answer:** Use `Promise.all` when you need all results and can start all operations simultaneously. Use an async generator when: results need to be consumed as they arrive (streaming), the dataset is too large to load entirely into memory, each operation depends on the previous, you need backpressure (consumer controls the pace), or you need lazy pagination where not all pages may be needed.

## Common Pitfalls

### 1. Generators Are Exhausted After One Iteration

```javascript
function* nums() { yield 1; yield 2; yield 3; }

const gen = nums();
console.log([...gen]); // [1, 2, 3]
console.log([...gen]); // [] — already exhausted

// Fix: call the generator function again each time
console.log([...nums()]); // [1, 2, 3]
```

### 2. Missing await on Async Generator Iteration

```javascript
// Bug: doesn't await — iterates the async generator synchronously
for (const item of asyncGen()) { // Should be 'for await'
  // item is a Promise, not the resolved value
}
```

### 3. Forgetting That for await...of Can Throw

```javascript
// Don't forget to handle errors from async iterables
for await (const item of asyncGen()) {
  // If asyncGen() throws, error propagates here
}

// Wrap in try/catch
try {
  for await (const item of asyncGen()) {
    processItem(item);
  }
} catch (err) {
  handleError(err);
}
```

### 4. break Triggering Generator Cleanup

When you `break` out of a `for...of` or `for await...of`, the runtime calls `generator.return()`, triggering any `finally` blocks. This is correct behavior, but be aware that cleanup code runs even on early exit.

## Resources

### Official Documentation
- [MDN: Generator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Generator)
- [MDN: function*](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*)
- [MDN: for await...of](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for-await...of)
- [MDN: Symbol.asyncIterator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol/asyncIterator)

### Articles and Guides
- [ES6 In Depth: Generators](https://hacks.mozilla.org/2015/05/es6-in-depth-generators/) — Mozilla Hacks
- [Exploring JS: Generators](https://exploringjs.com/es6/ch_generators.html) — Axel Rauschmayer
- [Async iterators and generators](https://javascript.info/async-iterators-generators) — javascript.info
- [co.js](https://github.com/tj/co) — Historical generator-based async flow control

### Proposals
- [TC39: Async Iteration proposal](https://github.com/tc39/proposal-async-iteration)
