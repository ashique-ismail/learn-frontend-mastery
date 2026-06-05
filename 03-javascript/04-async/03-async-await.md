# async/await: Error Handling, Parallel vs Sequential

## The Idea

**In plain English:** async/await is a way to write code that waits for slow tasks (like fetching data from the internet) without freezing everything else. "async" marks a function as one that might take time, and "await" tells the code to pause and wait for the slow task to finish before moving on.

**Real-world analogy:** Imagine you are at a restaurant. You place your food order with the waiter and then sit back and chat with your friend instead of standing at the kitchen window staring. When your food is ready, the waiter brings it to you and you start eating.

- The waiter taking your order = calling an `async` function (starting the task)
- Waiting at your table and chatting = the event loop handling other code while the Promise is pending
- The moment your food arrives = the Promise resolving
- You starting to eat = the code after `await` running

---

## Overview

`async/await` is syntactic sugar over Promises, introduced in ES2017. It allows you to write asynchronous code that looks synchronous, making control flow and error handling dramatically easier to reason about. Under the hood, the JavaScript engine transforms `async` functions into state machines backed by Promises — using `async/await` is never "different" from using Promises; it is always Promises.

The two keywords work in tandem: `async` marks a function as asynchronous (ensuring it always returns a Promise), and `await` suspends execution of that function until the awaited Promise settles, without blocking the thread.

## The async Keyword

### What It Does

Declaring a function `async` does three things:

1. Ensures the function always returns a Promise
2. Enables use of `await` inside the function body
3. Wraps thrown values in a rejected Promise

```javascript
async function greet() {
  return 'Hello'; // Equivalent to return Promise.resolve('Hello')
}

greet().then(v => console.log(v)); // 'Hello'
```

```javascript
async function fail() {
  throw new Error('Oops'); // Equivalent to return Promise.reject(new Error('Oops'))
}

fail().catch(err => console.error(err.message)); // 'Oops'
```

### Async Arrow Functions and Methods

```javascript
const fetchUser = async (id) => {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
};

class UserService {
  async getUser(id) {
    const res = await fetch(`/api/users/${id}`);
    return res.json();
  }
}
```

## The await Keyword

`await` pauses execution of the containing `async` function and waits for the Promise to settle. Control is returned to the caller, and the event loop can process other tasks in the interim.

```javascript
async function loadDashboard(userId) {
  const user = await fetchUser(userId);      // Waits for fetchUser
  const posts = await fetchPosts(user.id);   // Then waits for fetchPosts
  return { user, posts };
}
```

### Awaiting Non-Promises

`await` wraps the value in `Promise.resolve()` first, so you can safely await any value:

```javascript
async function demo() {
  const a = await 42;           // Same as await Promise.resolve(42)
  const b = await 'hello';      // Fine
  const c = await null;         // Fine
  console.log(a, b, c);         // 42, 'hello', null
}
```

### Execution Order

Understanding when execution pauses and resumes is critical for interviews:

```javascript
console.log('1 - sync');

async function run() {
  console.log('2 - async start (sync part)');
  const result = await Promise.resolve('3 - resumed');
  console.log(result);
  console.log('4 - after await');
}

run();
console.log('5 - after run() call');

// Output:
// 1 - sync
// 2 - async start (sync part)
// 5 - after run() call
// 3 - resumed
// 4 - after await
```

Everything before the first `await` runs synchronously. The rest is scheduled as a microtask after the awaited Promise resolves.

## Error Handling

### try/catch/finally

`async/await` brings async error handling in line with synchronous error handling:

```javascript
async function loadUserData(id) {
  try {
    const res = await fetch(`/api/users/${id}`);

    if (!res.ok) {
      throw new Error(`HTTP ${res.status}: ${res.statusText}`);
    }

    return await res.json();
  } catch (err) {
    logger.error('Failed to load user', { id, err });
    throw err; // Re-throw if you can't recover
  } finally {
    hideLoadingSpinner(); // Always runs
  }
}
```

### Granular Error Handling

A single `try/catch` catches all errors from all awaited operations in the block. For granular control, use separate try/catch blocks or `.catch()` on individual awaits:

```javascript
// Option 1: Separate try/catch blocks
async function processOrder(id) {
  let order;
  try {
    order = await fetchOrder(id);
  } catch (err) {
    throw new OrderNotFoundError(id, err);
  }

  let payment;
  try {
    payment = await processPayment(order);
  } catch (err) {
    throw new PaymentFailedError(order.id, err);
  }

  return { order, payment };
}

// Option 2: Per-await .catch()
async function processOrder(id) {
  const order = await fetchOrder(id)
    .catch(err => { throw new OrderNotFoundError(id, err); });

  const payment = await processPayment(order)
    .catch(err => { throw new PaymentFailedError(order.id, err); });

  return { order, payment };
}
```

### The "Go-Style" Error Handling Pattern

Some codebases use a wrapper to avoid try/catch verbosity:

```javascript
async function to(promise) {
  try {
    const result = await promise;
    return [null, result];
  } catch (err) {
    return [err, undefined];
  }
}

// Usage
const [err, user] = await to(fetchUser(id));
if (err) return handleError(err);

const [err2, posts] = await to(fetchPosts(user.id));
if (err2) return handleError(err2);
```

This pattern (popularized by the `await-to-js` library) mimics Go's `err, value` idiom. It avoids nested try/catch but at the cost of visual noise and the risk of ignoring errors.

### Unhandled Rejections in async Functions

An `async` function that throws or rejects without a caller catching it creates an unhandled rejection:

```javascript
// Dangerous — nobody awaits or catches this
async function background() {
  await someRiskyOperation();
}

background(); // Fire-and-forget — rejection is unhandled
```

Fix: always handle or re-propagate errors from fire-and-forget async calls:

```javascript
background().catch(err => logger.error('Background task failed', err));
```

## Sequential vs. Parallel Execution

This is one of the most important distinctions to master for senior-level interviews.

### Sequential (Series)

Each `await` blocks until the previous operation completes. Total time is the sum of all operation durations:

```javascript
async function sequential() {
  const user     = await fetchUser(id);     // 200ms
  const profile  = await fetchProfile(id);  // 150ms
  const settings = await fetchSettings(id); // 100ms

  // Total: ~450ms — each waits for the previous
  return { user, profile, settings };
}
```

Use sequential when later operations depend on earlier results:

```javascript
async function getPostsForUser(userId) {
  const user = await fetchUser(userId);             // Must happen first
  const posts = await fetchPosts(user.preferredFeed); // Needs user.preferredFeed
  return posts;
}
```

### Parallel (Concurrent)

Start all operations simultaneously. Total time is the duration of the slowest single operation:

```javascript
async function parallel() {
  const [user, profile, settings] = await Promise.all([
    fetchUser(id),     // 200ms
    fetchProfile(id),  // 150ms
    fetchSettings(id), // 100ms
  ]);

  // Total: ~200ms — all run concurrently
  return { user, profile, settings };
}
```

### The Common Mistake: Sequential await of Independent Promises

```javascript
// BUG: Looks concurrent but is actually sequential
async function slow() {
  const userPromise     = fetchUser(id);     // Started immediately
  const profilePromise  = fetchProfile(id);  // Started immediately
  const settingsPromise = fetchSettings(id); // Started immediately

  // BUT: these awaits are sequential — each waits before starting the next await
  const user     = await userPromise;        // 200ms
  const profile  = await profilePromise;     // 150ms (but already resolved by now!)
  const settings = await settingsPromise;    // 100ms (already resolved)
}

// This IS concurrent! The Promises were created (and started) before any await.
// Total time: 200ms (the slowest). This works correctly.
```

Actually, saving Promise references and awaiting them later IS concurrent, because the Promises start executing immediately when constructed. The `await` just waits for already-running Promises. This code is fine — but using `Promise.all` is clearer in intent.

### When Not to Use Promise.all

```javascript
// WRONG: Promise.all with dependent operations
async function wrong() {
  const [user, posts] = await Promise.all([
    fetchUser(id),
    fetchPosts(id),  // What if posts require user data first?
  ]);
}
```

If `fetchPosts` needs the user's `preferredFeed` field, `Promise.all` won't work — you need the user first.

### Hybrid: Mixed Sequential and Parallel

Real applications often need both:

```javascript
async function loadDashboard(userId) {
  // Step 1: Must fetch user first (sequential)
  const user = await fetchUser(userId);

  // Step 2: Once we have user, fetch remaining resources in parallel
  const [posts, notifications, settings] = await Promise.all([
    fetchPosts(user.feedPreference),
    fetchNotifications(userId),
    fetchSettings(userId),
  ]);

  return { user, posts, notifications, settings };
}
```

## Advanced Patterns

### Async Iteration (for await...of)

Iterating over async iterables one at a time:

```javascript
async function processStream(asyncIterable) {
  for await (const chunk of asyncIterable) {
    await processChunk(chunk); // Each chunk awaited before reading next
  }
}

// Reading a Node.js Readable stream
async function readStream(stream) {
  const chunks = [];
  for await (const chunk of stream) {
    chunks.push(chunk);
  }
  return Buffer.concat(chunks).toString('utf8');
}
```

### Async IIFE

For running async code at module top level (before top-level await was available):

```javascript
(async () => {
  try {
    const config = await loadConfig();
    await startServer(config);
  } catch (err) {
    console.error('Failed to start:', err);
    process.exit(1);
  }
})();
```

### Retry Logic

Async/await makes retry loops straightforward:

```javascript
async function withRetry(fn, { maxAttempts = 3, delay = 1000 } = {}) {
  let lastError;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (err) {
      lastError = err;
      if (attempt < maxAttempts) {
        await new Promise(resolve => setTimeout(resolve, delay * attempt)); // Exponential backoff
      }
    }
  }

  throw lastError;
}

const data = await withRetry(() => fetch('/api/unstable').then(r => r.json()), {
  maxAttempts: 3,
  delay: 500,
});
```

### Memoizing Async Functions

```javascript
function memoizeAsync(fn) {
  const cache = new Map();

  return async function(...args) {
    const key = JSON.stringify(args);

    if (cache.has(key)) {
      return cache.get(key); // Returns cached Promise — won't re-run
    }

    const promise = fn.apply(this, args);
    cache.set(key, promise);

    // On rejection, remove from cache so next call retries
    promise.catch(() => cache.delete(key));

    return promise;
  };
}

const fetchUserCached = memoizeAsync(fetchUser);
```

## Async/Await and Classes

### async Class Methods

```javascript
class ApiClient {
  async get(endpoint) {
    const res = await fetch(this.baseUrl + endpoint, {
      headers: this.headers,
    });
    if (!res.ok) throw new Error(`API error: ${res.status}`);
    return res.json();
  }
}
```

### Async Constructors (Anti-Pattern)

Constructors cannot be async. The solution is a static factory:

```javascript
// Cannot do this:
class Database {
  async constructor() {
    this.connection = await connect(); // SyntaxError
  }
}

// Use a static factory instead:
class Database {
  static async create(config) {
    const db = new Database();
    db.connection = await connect(config);
    return db;
  }
}

const db = await Database.create(config);
```

## Comparison Table

| Feature | Promise chains | async/await |
| ------- | ------------ | ---------- |
| Readability | Moderate | High |
| Error handling | .catch() | try/catch |
| Debugging | Poor stack traces (pre-V8) | Good stack traces |
| Parallel execution | Promise.all | Promise.all (still needed) |
| Conditional logic | Awkward | Natural (if/else in function body) |
| Loops | .reduce() hacks | for/while loops |
| Cancellation | AbortController | AbortController |
| Interop | Native | Compiles to Promises |

## Best Practices

### 1. Always Await or Return Promises from async Functions

```javascript
// Bug: Promise not awaited or returned — errors swallowed
async function save(data) {
  db.insert(data); // Missing await — error is lost
}

// Fixed
async function save(data) {
  await db.insert(data);
}
// Or
async function save(data) {
  return db.insert(data); // Return the Promise — let caller handle it
}
```

### 2. Avoid Unnecessary await on Return

Returning a Promise directly vs `await`ing it then returning is nearly equivalent, except that `await` in the return creates a new stack frame that aids debugging:

```javascript
// Equivalent in most cases
async function getUser(id) {
  return fetchUser(id); // No await — returns the Promise directly
}

// Slightly better for stack traces in try/catch
async function getUser(id) {
  return await fetchUser(id); // Creates stack frame in async fn
}
```

Note: if there's a `try/catch` wrapping a `return await`, the `await` is required to ensure errors are caught:

```javascript
async function getUser(id) {
  try {
    return await fetchUser(id); // await required — otherwise catch won't fire
  } catch (err) {
    return defaultUser;
  }
}
```

### 3. Parallelize When Possible

Identify independent operations and run them with `Promise.all`.

### 4. Handle Errors at the Right Level

Not every async function needs its own try/catch. Let errors propagate up to where they can be meaningfully handled.

## Interview Questions

### Q1: What happens to the event loop when await is encountered?

**Answer:** `await` suspends execution of the current `async` function and schedules the continuation as a microtask once the awaited Promise resolves. Control is returned to the caller. The thread is not blocked — the event loop can process other macrotasks and microtasks while waiting. When the Promise resolves, the continuation is added to the microtask queue and runs before the next macrotask.

### Q2: What is the output of this code?

```javascript
async function main() {
  console.log('A');
  await null;
  console.log('B');
}

console.log('C');
main();
console.log('D');
```

**Answer:** `C`, `A`, `D`, `B`. `C` runs synchronously. `main()` is called, `A` logs synchronously, then `await null` suspends `main`. `D` logs synchronously. Then in the microtask queue, `B` logs.

### Q3: What is wrong with this code?

```javascript
async function getAll(ids) {
  const results = [];
  for (const id of ids) {
    results.push(await fetchItem(id));
  }
  return results;
}
```

**Answer:** Each `fetchItem` awaits sequentially — the loop does not proceed to the next ID until the current one resolves. Total time is the sum of all request times. For independent items, the fix is `return Promise.all(ids.map(id => fetchItem(id)))`, which runs all requests concurrently.

### Q4: Why do you need await on return inside try/catch?

**Answer:** Without `await`, `return fetchUser(id)` returns the Promise before the `try/catch` can catch any rejection — the rejection propagates to the caller outside the `catch`. With `await`, the Promise is settled inside the function, so any rejection is caught by the local `catch` block.

### Q5: How does async/await compare to generators for async flow control?

**Answer:** `async/await` is essentially a specialized version of the generator + Promise pattern that libraries like `co` implemented. Generators use `yield` to pause and can receive injected values when resumed. Async functions use `await` to pause on Promises and automatically resume with resolved values. The key differences: async functions are native (no runtime required), always work with Promises specifically, and have cleaner syntax. Generators are more flexible (any value can be yielded, consumers control resumption) but require a runner.

## Common Pitfalls

### 1. Using await in Non-async Callbacks

```javascript
// Error: await outside of async function
[1, 2, 3].forEach(async (id) => {
  const data = await fetch(id); // Works inside async, but...
  // forEach doesn't await the async callback!
  // The loop completes before any data is fetched
});

// Fix: use for...of with await, or Promise.all
for (const id of [1, 2, 3]) {
  const data = await fetch(id); // Awaited properly
}

// Or parallel:
await Promise.all([1, 2, 3].map(id => fetch(id)));
```

### 2. Async event Handlers

```javascript
// Error from async event handler is an unhandled rejection
button.addEventListener('click', async () => {
  const data = await fetchData(); // If this rejects, error is swallowed
});

// Fix: wrap in try/catch
button.addEventListener('click', async () => {
  try {
    const data = await fetchData();
    renderData(data);
  } catch (err) {
    showErrorMessage(err);
  }
});
```

### 3. Missing await on Void async Functions

```javascript
async function sendNotification(userId) {
  await db.insertNotification(userId);
  await emailService.send(userId);
}

async function handleRequest(req) {
  sendNotification(req.userId); // Bug: missing await — errors are unhandled
  return res.send('OK');
}
```

## Resources

### Official Documentation

- [MDN: async function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)
- [MDN: await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await)
- [ECMAScript: Async Functions proposal](https://tc39.es/ecma262/#sec-async-function-definitions)

### Articles and Guides

- [Async functions — making promises friendly](https://web.dev/async-functions/) — Google Web Dev
- [Faster async functions and promises](https://v8.dev/blog/fast-async) — V8 blog on implementation
- [You Don't Know JS: Async & Performance — Chapter 4](https://github.com/getify/You-Dont-Know-JS/blob/1st-ed/async%20%26%20performance/ch4.md)

### Tools

- [eslint-plugin-promise: no-return-await](https://github.com/eslint-community/eslint-plugin-promise) — Detects unnecessary await on return
- [TypeScript: AsyncFunction types](https://www.typescriptlang.org/docs/handbook/2/functions.html) — Typing async functions
