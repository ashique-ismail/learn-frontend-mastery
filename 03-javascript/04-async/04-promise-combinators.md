# Promise Combinators: all, allSettled, race, any

## The Idea

**In plain English:** Promise combinators are built-in JavaScript tools that let you kick off several tasks at the same time and then decide how to handle the results — for example, wait for all of them to finish, or just grab whichever one finishes first. A "Promise" is JavaScript's way of representing a task that hasn't completed yet, like loading a file or fetching data from the internet.

**Real-world analogy:** Imagine you and three friends each order food from a different counter at a food court. You can decide to eat together only when everyone has their tray (like `Promise.all`), eat as soon as the first person gets theirs (like `Promise.race`), or note down who got their food and who didn't without anyone holding up the group (like `Promise.allSettled`).

- The food court = your JavaScript program running multiple async operations
- Each person ordering food = one Promise (an ongoing task)
- Getting your tray = a Promise fulfilling (finishing successfully)
- A counter being closed = a Promise rejecting (failing)

---

## Overview

Promise combinators are static methods on the `Promise` constructor that accept an iterable of Promises and return a single new Promise whose resolution depends on the outcomes of the input Promises. They are the primary tools for managing concurrency in JavaScript — running multiple async operations in parallel and composing their results.

The four combinators introduced to the language at different points each answer a different question about a collection of Promises:

| Combinator | Introduced | Short-circuit | Resolves when |
|------------|-----------|---------------|---------------|
| `Promise.all` | ES2015 | On first rejection | All fulfill |
| `Promise.race` | ES2015 | On first settlement | First settles |
| `Promise.allSettled` | ES2020 | Never | All settle |
| `Promise.any` | ES2021 | On first fulfillment | First fulfills |

## Promise.all

Accepts an iterable of Promises. Returns a Promise that fulfills with an array of all fulfilled values (in the same order as the input), or rejects as soon as any one of the input Promises rejects.

```javascript
const [user, posts, notifications] = await Promise.all([
  fetchUser(userId),
  fetchPosts(userId),
  fetchNotifications(userId),
]);
```

### Fail-Fast Behavior

```javascript
const p1 = Promise.resolve('a');
const p2 = Promise.reject(new Error('b failed'));
const p3 = Promise.resolve('c');

Promise.all([p1, p2, p3])
  .then(values => console.log(values))
  .catch(err => console.error(err.message)); // 'b failed'

// p3 may have already resolved by the time we handle the error,
// but its value is discarded — there's no way to retrieve it
```

### Order Is Preserved

Results come back in the same order as the input iterable, regardless of completion order:

```javascript
const fast = delay(100).then(() => 'fast');
const slow = delay(500).then(() => 'slow');

const [a, b] = await Promise.all([slow, fast]);
console.log(a, b); // 'slow', 'fast' — order matches input, not completion
```

### Empty Iterable

`Promise.all([])` resolves immediately with `[]`.

### Use Cases

- Fetching multiple independent resources needed before rendering
- Running independent database queries in parallel
- Parallelizing a batch of the same operation

```javascript
// Parallel fetch of user records
async function getUsersBatch(ids) {
  return Promise.all(ids.map(id => fetchUser(id)));
}

// Parallel file operations
async function readAllConfigs(paths) {
  return Promise.all(paths.map(p => fs.promises.readFile(p, 'utf8')));
}
```

### Controlling Concurrency

`Promise.all` fires all operations simultaneously. With large arrays this can overwhelm resources. Use a concurrency limiter:

```javascript
async function mapWithConcurrency(items, fn, limit = 5) {
  const results = [];
  let index = 0;

  async function worker() {
    while (index < items.length) {
      const i = index++;
      results[i] = await fn(items[i]);
    }
  }

  const workers = Array.from({ length: Math.min(limit, items.length) }, worker);
  await Promise.all(workers);
  return results;
}

// Process 100 URLs, max 5 concurrent
const results = await mapWithConcurrency(urls, fetchAndParse, 5);
```

## Promise.allSettled

Returns a Promise that fulfills once all input Promises have settled (fulfilled OR rejected). The resolved value is an array of descriptor objects — never rejects.

```javascript
const results = await Promise.allSettled([
  fetchUser(1),
  fetchUser(2),
  fetchUser(3),
]);

results.forEach((result, i) => {
  if (result.status === 'fulfilled') {
    console.log(`User ${i + 1}:`, result.value);
  } else {
    console.error(`User ${i + 1} failed:`, result.reason);
  }
});
```

### Descriptor Shape

Each element is one of:

```javascript
// Fulfilled descriptor
{ status: 'fulfilled', value: <resolved value> }

// Rejected descriptor
{ status: 'rejected', reason: <rejection reason> }
```

### Use Cases

- Best-effort batch operations where partial success is acceptable
- Audit/logging: you want to know which operations failed and which succeeded
- Cleanup after parallel operations: running tear-down regardless of success

```javascript
async function sendNotifications(userIds) {
  const results = await Promise.allSettled(
    userIds.map(id => sendEmail(id))
  );

  const failures = results
    .filter(r => r.status === 'rejected')
    .map(r => r.reason);

  if (failures.length > 0) {
    logger.warn(`${failures.length} notifications failed`, failures);
  }

  return results.filter(r => r.status === 'fulfilled').length;
}
```

### Implementing allSettled with all (Pre-ES2020 Polyfill)

```javascript
function allSettled(promises) {
  return Promise.all(
    promises.map(p =>
      Promise.resolve(p).then(
        value => ({ status: 'fulfilled', value }),
        reason => ({ status: 'rejected', reason })
      )
    )
  );
}
```

## Promise.race

Returns a Promise that settles (fulfills or rejects) as soon as the first input Promise settles. Subsequent settlements are ignored.

```javascript
const result = await Promise.race([
  fetchFromPrimaryDB(),
  fetchFromReplicaDB(),
]);
// Whichever responds first wins
```

### Timeout Pattern

The most common use case:

```javascript
function withTimeout(promise, ms, message = `Timed out after ${ms}ms`) {
  const timeout = new Promise((_, reject) =>
    setTimeout(() => reject(new Error(message)), ms)
  );
  return Promise.race([promise, timeout]);
}

const data = await withTimeout(fetch('/api/slow-endpoint'), 3000);
```

### Cancellation Awareness

Note that `Promise.race` does not cancel the "losing" Promises. They continue running; their results are just ignored. If you need to stop the losing operation, you need `AbortController`:

```javascript
async function fetchWithRace(urls) {
  const controller = new AbortController();

  const promises = urls.map(url =>
    fetch(url, { signal: controller.signal })
      .then(r => r.json())
  );

  try {
    const result = await Promise.race(promises);
    controller.abort(); // Cancel remaining requests
    return result;
  } catch (err) {
    if (err.name !== 'AbortError') throw err;
  }
}
```

### Empty Iterable

`Promise.race([])` returns a Promise that never settles. This is generally a mistake.

### Edge Cases

```javascript
// race with an already-resolved Promise
Promise.race([
  Promise.resolve('instant'),
  new Promise(resolve => setTimeout(() => resolve('delayed'), 1000)),
]).then(v => console.log(v)); // 'instant' — but runs in next microtask turn
```

## Promise.any

Returns a Promise that fulfills as soon as the first input Promise fulfills. Rejects only if ALL input Promises reject, in which case it rejects with an `AggregateError` containing all rejection reasons.

```javascript
const result = await Promise.any([
  fetchFromCDN1(resource),
  fetchFromCDN2(resource),
  fetchFromCDN3(resource),
]);
// First successful response wins; failures are ignored unless all fail
```

### AggregateError

```javascript
try {
  await Promise.any([
    Promise.reject(new Error('CDN1 down')),
    Promise.reject(new Error('CDN2 down')),
    Promise.reject(new Error('CDN3 down')),
  ]);
} catch (err) {
  console.log(err instanceof AggregateError); // true
  console.log(err.message);   // 'All promises were rejected'
  console.log(err.errors);    // [Error: CDN1 down, Error: CDN2 down, Error: CDN3 down]
}
```

### Use Cases

- Fetching from multiple CDNs or mirrors, using whichever responds first
- Trying multiple authentication strategies
- Racing between cache and network with a preference for either

```javascript
// Stale-while-revalidate pattern
async function getResource(key) {
  return Promise.any([
    getFromCache(key),           // Fast — may throw if miss
    fetchFromNetwork(key),       // Slower — authoritative
  ]);
}
```

### Difference from race

- `race` rejects on the first rejection; `any` only rejects if ALL reject.
- `any` is optimistic; `race` is neutral about settlement type.

```javascript
// race: first rejection wins
Promise.race([
  Promise.reject(new Error('fast fail')),
  Promise.resolve('slow success'),
]).catch(err => console.error(err.message)); // 'fast fail'

// any: first fulfillment wins — rejection is ignored
Promise.any([
  Promise.reject(new Error('fast fail')),
  Promise.resolve('slow success'),
]).then(v => console.log(v)); // 'slow success'
```

### Empty Iterable

`Promise.any([])` immediately rejects with an empty `AggregateError`.

## Comparison Table

| Feature | all | allSettled | race | any |
|---------|-----|-----------|------|-----|
| Resolves with | Array of values | Array of descriptors | First settled value | First fulfilled value |
| Short-circuits on | First rejection | Never | First settlement | First fulfillment |
| Rejects when | Any input rejects | Never | First input rejects | All inputs reject |
| Rejection value | First rejection | N/A | First rejection | AggregateError |
| Empty input | Resolves `[]` | Resolves `[]` | Never settles | Rejects AggregateError |
| Use case | All must succeed | Partial success OK | Fastest wins | First success wins |

## Implementing All Four from Scratch

Understanding internals cements interview recall:

```javascript
function myAll(promises) {
  return new Promise((resolve, reject) => {
    const results = [];
    let remaining = 0;

    const arr = Array.from(promises);
    if (arr.length === 0) return resolve([]);

    remaining = arr.length;

    arr.forEach((p, i) => {
      Promise.resolve(p).then(value => {
        results[i] = value;
        if (--remaining === 0) resolve(results);
      }, reject);
    });
  });
}

function myAllSettled(promises) {
  return myAll(
    Array.from(promises).map(p =>
      Promise.resolve(p).then(
        value  => ({ status: 'fulfilled', value }),
        reason => ({ status: 'rejected', reason })
      )
    )
  );
}

function myRace(promises) {
  return new Promise((resolve, reject) => {
    for (const p of promises) {
      Promise.resolve(p).then(resolve, reject);
    }
  });
}

function myAny(promises) {
  return new Promise((resolve, reject) => {
    const errors = [];
    let remaining = 0;

    const arr = Array.from(promises);
    if (arr.length === 0) {
      return reject(new AggregateError([], 'All promises were rejected'));
    }

    remaining = arr.length;

    arr.forEach((p, i) => {
      Promise.resolve(p).then(resolve, reason => {
        errors[i] = reason;
        if (--remaining === 0) {
          reject(new AggregateError(errors, 'All promises were rejected'));
        }
      });
    });
  });
}
```

## Best Practices

### 1. Use allSettled for Non-Critical Parallel Operations

If one failure shouldn't block the entire operation, use `allSettled` and handle each result individually rather than losing all results on a single failure.

### 2. Use all When All Operations Must Succeed

If all operations are required and a single failure is fatal to the overall goal, use `all`. The fail-fast behavior is intentional.

### 3. Attach Error Handlers to all Input Promises Before Passing to race

Unhandled rejections on the "losing" Promises in `race` can still trigger unhandled rejection warnings. Attach `.catch(() => {})` to non-winner Promises if you don't care about their errors:

```javascript
const safePromises = [p1, p2, p3].map(p => p.catch(() => undefined));
const result = await Promise.race(safePromises);
```

### 4. Do Not Use race for Retry Logic

`race` picks the fastest — not the most successful. Use `any` when you want the first success:

```javascript
// Wrong for retry: fails if the fastest is a rejection
const result = await Promise.race(attempts);

// Correct for retry: takes first success
const result = await Promise.any(attempts);
```

### 5. Always Handle the AggregateError from any

```javascript
try {
  const result = await Promise.any(sources);
} catch (err) {
  if (err instanceof AggregateError) {
    // All sources failed — handle gracefully
    console.error('All sources failed:', err.errors);
  } else {
    throw err; // Unexpected error
  }
}
```

## Interview Questions

### Q1: What is the difference between Promise.all and Promise.allSettled?

**Answer:** `Promise.all` short-circuits on the first rejection and rejects with that error, discarding any results already collected. `Promise.allSettled` never rejects — it waits for every Promise to settle and returns an array of descriptor objects with a `status` field. Use `all` when all operations must succeed; use `allSettled` when you want a full picture of what succeeded and what failed.

### Q2: What does Promise.race return if the input array is empty?

**Answer:** A Promise that never settles. This is almost always a bug. In contrast, `Promise.all([])` resolves to `[]`, `Promise.allSettled([])` resolves to `[]`, and `Promise.any([])` rejects with an empty `AggregateError`.

### Q3: You use Promise.race to implement a timeout. Does the original request get cancelled?

**Answer:** No. The original Promise continues executing; its resolution is simply ignored once the timeout Promise settles. For actual cancellation, you need to pair `Promise.race` with `AbortController` and pass the signal to the fetch call. Without this, the underlying network request remains in flight, consuming browser connections and server resources.

### Q4: Why might you see unhandled rejection warnings even when using Promise.race?

**Answer:** Because the "losing" Promises in a `race` may reject after the winner has settled. If those Promises have no `.catch()` handler, the runtime detects an unhandled rejection. Fix by either attaching `.catch(() => {})` to the losing Promises before passing them to `race`, or using `AbortController` to cancel them.

### Q5: Implement a function that runs Promises with a concurrency limit.

**Answer:** See the `mapWithConcurrency` implementation above — spawn N worker coroutines that each pull from a shared index counter. All workers run in `Promise.all`, and each worker processes items sequentially until the list is exhausted.

## Common Pitfalls

### 1. Expecting Fail-Fast to Cancel Other Operations

```javascript
// Misconception: rejected Promise cancels others
await Promise.all([expensiveOp1(), expensiveOp2(), expensiveOp3()]);
// Reality: all three ops run; the error just prevents waiting for others
```

### 2. Treating allSettled Results as Values Directly

```javascript
const results = await Promise.allSettled([p1, p2]);
results.forEach(r => console.log(r.value)); // Bug: rejected items have 'reason', not 'value'

// Correct
results.forEach(r => {
  if (r.status === 'fulfilled') console.log(r.value);
  else console.error(r.reason);
});
```

### 3. Using race When Parallel Operations Can Reject Quickly

If the first operation that settles is a fast rejection, `race` rejects immediately even though a slower operation might have succeeded. `any` is usually the right choice for "first success" patterns.

### 4. Passing Non-Promise Values to Combinators

All combinators accept thenables and plain values — these are wrapped in `Promise.resolve()` automatically. This is intentional and safe, but can be confusing:

```javascript
Promise.all([1, 2, Promise.resolve(3)]).then(v => console.log(v)); // [1, 2, 3]
```

## Resources

### Official Documentation
- [MDN: Promise.all](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all)
- [MDN: Promise.allSettled](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled)
- [MDN: Promise.race](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/race)
- [MDN: Promise.any](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/any)
- [TC39: Promise.allSettled proposal](https://github.com/tc39/proposal-promise-allSettled)
- [TC39: Promise.any proposal](https://github.com/tc39/proposal-promise-any)

### Articles and Guides
- [JavaScript Promise combinators](https://v8.dev/features/promise-combinators) — V8 team overview
- [AggregateError](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/AggregateError) — MDN reference for error type introduced with any
