# Promises: States, Chaining, and Error Handling

## The Idea

**In plain English:** A Promise is a placeholder for a value you don't have yet but expect to receive in the future — your code can keep running while waiting for it, and you attach instructions for what to do once the answer arrives (or something goes wrong).

**Real-world analogy:** Ordering a meal at a restaurant and receiving a buzzer. When you order, the kitchen is still cooking — you don't wait frozen at the counter. The buzzer is your promise that a result is coming. When it goes off, you either pick up your food or find out they ran out.

- The buzzer = the Promise object (a stand-in for the future result)
- The kitchen finishing your food = the async operation completing (fulfillment)
- The kitchen running out of ingredients = the async operation failing (rejection)

---

## Overview

A Promise is an object representing the eventual completion or failure of an asynchronous operation. Introduced natively in ES6 (ES2015) and specified in the Promises/A+ standard, Promises solve the two core problems of callbacks: inversion of control and composition difficulty.

Unlike callbacks, a Promise is a returned object to which you attach callbacks rather than passing callbacks into a function. This seemingly small difference restores control to the caller and enables the compositional patterns that make async code readable.

## Promise States

A Promise is always in one of three mutually exclusive states:

### Pending

The initial state. The asynchronous operation has not yet completed:

```javascript
const p = new Promise((resolve, reject) => {
  // As long as neither resolve nor reject is called, p is pending
  setTimeout(() => resolve('done'), 1000);
});

console.log(p); // Promise { <pending> }
```

### Fulfilled

The operation completed successfully. The Promise has a resolved value:

```javascript
const fulfilled = Promise.resolve(42);
console.log(fulfilled); // Promise { 42 }

fulfilled.then(value => console.log(value)); // 42
```

### Rejected

The operation failed. The Promise has a rejection reason (typically an `Error`):

```javascript
const rejected = Promise.reject(new Error('Something went wrong'));

rejected.catch(err => console.error(err.message)); // Something went wrong
```

### Settlement Is Permanent

Once a Promise settles (fulfills or rejects), its state and value are locked. Calling `resolve` or `reject` a second time has no effect:

```javascript
const p = new Promise((resolve, reject) => {
  resolve('first');
  resolve('second'); // Ignored
  reject(new Error('too late')); // Ignored
});

p.then(v => console.log(v)); // 'first' — always
```

## The Promise Constructor

```javascript
const promise = new Promise((resolve, reject) => {
  // This function is called synchronously (the executor runs immediately)
  console.log('Executor running');

  performAsyncWork((err, result) => {
    if (err) {
      reject(err); // Reject with any value, but Error objects are convention
    } else {
      resolve(result); // Resolve with any value (including another Promise)
    }
  });
});

console.log('After new Promise');
// Output: 'Executor running', then 'After new Promise'
```

Important: the executor function runs synchronously. Only the resolution is asynchronous.

### Resolving with a Promise

If you resolve a Promise with another Promise (or any "thenable"), the outer Promise adopts the state of the inner:

```javascript
const inner = new Promise((resolve) => setTimeout(() => resolve('inner value'), 1000));

const outer = new Promise((resolve) => {
  resolve(inner); // outer "follows" inner
});

outer.then(v => console.log(v)); // 'inner value' after 1 second
```

## Chaining

The key insight of Promises: `.then()` always returns a new Promise, enabling method chaining.

### Basic Chain

```javascript
fetch('/api/user')
  .then(response => response.json())      // transforms value
  .then(user => fetch(`/api/posts/${user.id}`))  // returns new Promise
  .then(response => response.json())      // waits for that Promise
  .then(posts => renderPosts(posts))
  .catch(err => handleError(err));
```

### Return Value Semantics in .then()

The value returned from a `.then()` callback becomes the resolution value of the next Promise in the chain:

```javascript
Promise.resolve(1)
  .then(n => n + 1)        // returns 2 (plain value)
  .then(n => n * 3)        // returns 6
  .then(n => Promise.resolve(n + 10)) // returns Promise<16>
  .then(n => console.log(n)); // 16
```

Returning a rejected Promise or throwing inside `.then()` rejects the next Promise in the chain:

```javascript
Promise.resolve('ok')
  .then(v => {
    throw new Error('handler threw');
  })
  .then(v => console.log('never reached'))
  .catch(err => console.error(err.message)); // 'handler threw'
```

### Flattening Nested Async Calls

Promises automatically unwrap returned Promises, avoiding nesting:

```javascript
// Without unwrapping you'd get Promise<Promise<User>>
// With unwrapping, you get Promise<User>
getUser(id)
  .then(user => getUserSettings(user.id)) // Returns a Promise — chain flattens
  .then(settings => console.log(settings)); // Gets the resolved settings value
```

## Error Handling

### .catch()

`.catch(handler)` is syntactic sugar for `.then(undefined, handler)`:

```javascript
fetch('/api/data')
  .then(res => res.json())
  .then(data => processData(data))
  .catch(err => {
    // Catches errors from ANY of the above steps
    console.error('Pipeline failed:', err);
  });
```

### .then() with Two Arguments

`.then(onFulfilled, onRejected)` handles both cases, but the `onRejected` only covers the Promise it's attached to, not the `onFulfilled` handler in the same call:

```javascript
promise
  .then(
    value => riskyTransform(value),  // if THIS throws...
    err => console.error(err)         // ...this does NOT catch it
  )
  .catch(err => console.error(err)); // This catches errors from riskyTransform
```

In most cases, prefer a terminal `.catch()` over a second argument to `.then()`.

### Error Recovery

You can recover from errors inside `.catch()` by returning a value — the chain continues as fulfilled:

```javascript
fetch('/api/primary')
  .catch(() => fetch('/api/fallback')) // Recover to a fallback
  .then(res => res.json())             // Continues normally
  .then(data => render(data))
  .catch(err => showError(err));       // Only fires if fallback also fails
```

### .finally()

`.finally(callback)` runs regardless of outcome, useful for cleanup. It does not receive any argument and passes through the original settled value:

```javascript
function fetchWithLoading(url) {
  showSpinner();

  return fetch(url)
    .then(res => res.json())
    .finally(() => hideSpinner()); // Always hides spinner
    // Note: .finally() returns a Promise that settles with the same value/reason
}

fetchWithLoading('/api/data')
  .then(data => render(data))
  .catch(err => showError(err));
```

If `.finally()` throws or returns a rejected Promise, that rejection propagates, overriding the original settlement.

## Promise Static Methods

### Promise.resolve() and Promise.reject()

```javascript
// Lift a value into a resolved Promise
const p1 = Promise.resolve(42);

// Lift an error into a rejected Promise
const p2 = Promise.reject(new Error('fail'));

// If you pass a thenable, it assimilates it
const p3 = Promise.resolve(fetch('/api'));  // Same as fetch('/api')
```

### Promise.all()

Waits for all Promises to fulfill. Rejects immediately if any rejects:

```javascript
const [user, settings, posts] = await Promise.all([
  fetchUser(id),
  fetchSettings(id),
  fetchPosts(id),
]);
```

See `promise-combinators.md` for deep coverage of all combinators.

## Microtask Queue

Promise callbacks execute in the microtask queue, which drains between tasks in the event loop — before `setTimeout`, `setInterval`, and I/O callbacks:

```javascript
console.log('1');

setTimeout(() => console.log('2 - macrotask'), 0);

Promise.resolve()
  .then(() => console.log('3 - microtask'))
  .then(() => console.log('4 - microtask'));

console.log('5');

// Output: 1, 5, 3, 4, 2
```

This means chaining many `.then()` handlers does not block I/O or rendering between callbacks — they all run synchronously once the microtask queue is entered.

### Unhandled Rejections

If a Promise rejects with no `.catch()` handler, it becomes an unhandled rejection. In Node.js 15+ and modern browsers, this crashes the process or throws an uncaught error:

```javascript
// Dangerous — will emit UnhandledPromiseRejectionWarning or crash
const p = Promise.reject(new Error('oops'));

// Fix 1: Attach a catch
p.catch(err => logError(err));

// Fix 2: Global handler (last resort)
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection:', reason);
  process.exit(1);
});
```

## Creating Promisified Utilities

### Delay

```javascript
function delay(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

await delay(1000); // Waits 1 second
```

### Timeout Wrapper

```javascript
function withTimeout(promise, ms) {
  const timeout = new Promise((_, reject) =>
    setTimeout(() => reject(new Error(`Timed out after ${ms}ms`)), ms)
  );
  return Promise.race([promise, timeout]);
}

const data = await withTimeout(fetch('/api/slow'), 5000);
```

### Converting Callbacks

```javascript
function readFileAsync(path, encoding) {
  return new Promise((resolve, reject) => {
    fs.readFile(path, encoding, (err, data) => {
      if (err) reject(err);
      else resolve(data);
    });
  });
}
```

## Comparison Table

| Feature | Callbacks | Promises |
|---------|-----------|---------|
| Composability | Manual wiring | Chainable via .then() |
| Error propagation | Manual at each step | Automatic through chain |
| Inversion of control | High | Low (caller-owned chain) |
| Cancellation | Ad-hoc flags | No native (AbortController needed) |
| Multiple values | Manual | Promise.all / .race |
| Sync/async consistency | Potential Zalgo | Always async (microtask) |
| Stack traces | Good | Poor (pre-async/await) |
| Debugging | Familiar | Requires understanding microtasks |

## Best Practices

### 1. Always Return Promises in Chains

```javascript
// Bug: missing return breaks the chain
.then(user => {
  fetchProfile(user.id); // Not returned — next .then gets undefined
})

// Fixed
.then(user => {
  return fetchProfile(user.id);
})
// Or concisely:
.then(user => fetchProfile(user.id))
```

### 2. Always Have a Terminal .catch()

Every Promise chain should end with `.catch()` unless the caller is explicitly expected to handle rejections:

```javascript
initiateOperation()
  .then(step1)
  .then(step2)
  .catch(err => {
    logger.error(err);
    notifyUser('Something went wrong');
  });
```

### 3. Avoid the Promise Constructor Anti-Pattern

Don't wrap Promises in `new Promise()`:

```javascript
// Anti-pattern: deferred/explicit-construction anti-pattern
function fetchUser(id) {
  return new Promise((resolve, reject) => {
    fetch(`/api/users/${id}`)
      .then(res => res.json())
      .then(resolve)
      .catch(reject);
  });
}

// Correct: just return the chain
function fetchUser(id) {
  return fetch(`/api/users/${id}`).then(res => res.json());
}
```

### 4. Don't Nest — Chain

```javascript
// Nested (still callback hell with Promises)
fetchUser(id).then(user => {
  fetchProfile(user.profileId).then(profile => {
    fetchPosts(profile.id).then(posts => {
      render(user, profile, posts);
    });
  });
});

// Chained (correct)
fetchUser(id)
  .then(user => fetchProfile(user.profileId).then(profile => ({ user, profile })))
  .then(({ user, profile }) => fetchPosts(profile.id).then(posts => ({ user, profile, posts })))
  .then(({ user, profile, posts }) => render(user, profile, posts));

// Best: use async/await for sequential steps that share scope
```

### 5. Reject with Error Objects

Always reject with actual `Error` instances to preserve stack traces:

```javascript
reject(new Error('Validation failed')); // Good
reject('Validation failed');            // Bad — no stack trace
```

## Interview Questions

### Q1: What is the difference between .then(fn, fn) and .then(fn).catch(fn)?

**Answer:** In `.then(onFulfilled, onRejected)`, the `onRejected` handler only catches rejections from the Promise it's attached to — it will NOT catch errors thrown by `onFulfilled`. In contrast, `.then(onFulfilled).catch(onRejected)` the `.catch` covers both the original Promise and any error thrown inside `onFulfilled`. The second pattern is almost always preferred.

### Q2: What happens if you throw inside a .then() callback?

**Answer:** Throwing synchronously inside a `.then()` handler causes the returned Promise to reject with the thrown value. This is equivalent to returning `Promise.reject(thrownValue)`. The error propagates down the chain to the next `.catch()`.

### Q3: What are microtasks and how do Promises use them?

**Answer:** Microtasks are a queue in the JavaScript event loop that is processed after every macrotask (script, setTimeout callback, I/O callback) but before the next macrotask runs. Promise resolution handlers (`.then`, `.catch`, `.finally`) are scheduled as microtasks. This guarantees consistent async behavior — Promise handlers never run synchronously, but they run before timers, I/O, and rendering callbacks.

### Q4: What is the "explicit construction" or "deferred" anti-pattern?

**Answer:** It's when you wrap an existing Promise in `new Promise()` unnecessarily. Since `new Promise` already exists for converting non-Promise async code to Promises, wrapping an existing Promise just adds indirection and an extra nested scope. The fix is to simply return or chain off the existing Promise.

### Q5: Can a Promise be cancelled?

**Answer:** No. Native Promises have no cancellation mechanism. Once created, a Promise runs to completion. The common workaround is `AbortController` for `fetch`-based operations, or maintaining a `cancelled` flag for custom async work. Libraries like RxJS (Observables) and some userland Promise libraries provide cancellable constructs.

## Common Pitfalls

### 1. Forgetting to Return in a Chain

```javascript
// Bug: produces undefined in next step
.then(user => {
  processUser(user); // Returns undefined implicitly
})
.then(result => result.toUpperCase()) // TypeError: result is undefined
```

### 2. Swallowing Errors with Empty .catch()

```javascript
.catch(() => {}) // Silently swallows — chain resumes as fulfilled with undefined
```

### 3. Creating Promises Inside Loops Without Collecting Them

```javascript
const ids = [1, 2, 3];

// Bug: Promises created but never awaited
ids.forEach(id => fetchUser(id));

// Fix
const users = await Promise.all(ids.map(id => fetchUser(id)));
```

### 4. Async Constructor

Constructors cannot be async. Attempting to `await` inside `new Promise()` executor is a mistake:

```javascript
// This doesn't make the constructor async-aware
class Service {
  constructor() {
    this.data = await fetchData(); // SyntaxError: cannot use await here
  }
}

// Fix: use a static factory method
class Service {
  static async create() {
    const service = new Service();
    service.data = await fetchData();
    return service;
  }
}
```

## Resources

### Official Documentation
- [MDN: Using Promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises)
- [MDN: Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)
- [Promises/A+ Specification](https://promisesaplus.com/)

### Articles and Guides
- [You Don't Know JS: Async & Performance — Chapter 3: Promises](https://github.com/getify/You-Dont-Know-JS/blob/1st-ed/async%20%26%20performance/ch3.md)
- [We Have a Problem with Promises](https://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html) — Nolan Lawson on common mistakes
- [Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/) — Jake Archibald

### Tools
- [ESLint: no-floating-promises](https://typescript-eslint.io/rules/no-floating-promises/) — TypeScript ESLint rule for unhandled Promises
- [Why Promise rejection tracking](https://v8.dev/blog/fast-async) — V8 performance article on Promises
