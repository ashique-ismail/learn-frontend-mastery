# AbortController and Cancellation

## Overview

JavaScript's async model — Promises, `fetch`, async/await — has historically lacked a standard cancellation mechanism. `AbortController` (introduced in the DOM Living Standard and available in browsers since 2017, Node.js since v15) provides the official solution: a controller object paired with a signal that can be passed to cancellable APIs to request early termination.

The model is cooperative rather than preemptive: the signal communicates a cancellation request, and each API decides how to respond. `AbortController` is now the standard mechanism used by `fetch`, `addEventListener`, streams, IndexedDB operations, and user-space async utilities.

## Core Concepts

### AbortController and AbortSignal

```javascript
const controller = new AbortController();
const signal = controller.signal;

console.log(signal.aborted); // false
console.log(signal.reason);  // undefined

controller.abort();

console.log(signal.aborted); // true
console.log(signal.reason);  // DOMException: AbortError (default reason)
```

`AbortController` is the sender — it holds the only reference to `abort()`. `AbortSignal` is the receiver — it is read-only and can be passed to any number of cancellable operations. This separation enforces that only the code that owns the controller can trigger cancellation.

### Aborting with a Reason

```javascript
const controller = new AbortController();

controller.abort(new Error('User navigated away'));

console.log(controller.signal.reason); // Error: User navigated away
```

Passing a custom reason is useful for distinguishing intentional cancellation from timeout or error conditions.

## Cancelling fetch

The most common use case:

```javascript
async function fetchWithCancel(url) {
  const controller = new AbortController();

  // Cancel after 5 seconds
  const timeoutId = setTimeout(() => controller.abort(), 5000);

  try {
    const response = await fetch(url, { signal: controller.signal });
    const data = await response.json();
    return data;
  } catch (err) {
    if (err.name === 'AbortError') {
      console.log('Request was cancelled');
      return null;
    }
    throw err; // Re-throw unexpected errors
  } finally {
    clearTimeout(timeoutId);
  }
}
```

When `controller.abort()` is called, `fetch` rejects the returned Promise with a `DOMException` whose `name` is `'AbortError'`.

### Checking Signal State Before Starting

A signal may already be aborted before the operation starts. Check it at the beginning of long-running work:

```javascript
async function processItems(items, signal) {
  for (const item of items) {
    if (signal.aborted) {
      throw signal.reason ?? new DOMException('Aborted', 'AbortError');
    }
    await processItem(item, signal);
  }
}
```

## AbortSignal.timeout()

A static convenience method that returns a signal that automatically aborts after a given number of milliseconds (ES2022):

```javascript
// Clean timeout without manual controller management
const response = await fetch('/api/data', {
  signal: AbortSignal.timeout(5000),
});
```

This is equivalent to creating a controller, calling `setTimeout(() => controller.abort(), 5000)`, and passing `controller.signal` — but without the cleanup ceremony.

## AbortSignal.any()

Combines multiple signals into one that aborts when any of the inputs abort (ES2023):

```javascript
const userController = new AbortController();
const timeoutSignal = AbortSignal.timeout(10000);

// Abort when user cancels OR after 10 seconds
const combinedSignal = AbortSignal.any([
  userController.signal,
  timeoutSignal,
]);

const response = await fetch('/api/data', { signal: combinedSignal });
```

This is analogous to `Promise.race` but for abort signals.

## Listening to the Abort Event

You can react to cancellation by listening for the `'abort'` event on the signal:

```javascript
function cancellableDelay(ms, signal) {
  return new Promise((resolve, reject) => {
    const timeoutId = setTimeout(resolve, ms);

    signal.addEventListener('abort', () => {
      clearTimeout(timeoutId);
      reject(signal.reason ?? new DOMException('Aborted', 'AbortError'));
    }, { once: true });
  });
}

// Usage
const controller = new AbortController();
setTimeout(() => controller.abort(), 1000);

try {
  await cancellableDelay(5000, controller.signal);
  console.log('Delay completed');
} catch (err) {
  if (err.name === 'AbortError') console.log('Delay was cancelled');
}
```

### signal.throwIfAborted()

A convenience method that throws the abort reason if the signal is already aborted:

```javascript
async function doWork(items, signal) {
  for (const item of items) {
    signal.throwIfAborted(); // Throws DOMException if aborted

    await processItem(item);
  }
}
```

## Integrating AbortController into Custom Async Functions

### Pattern: Pass Signal Down the Call Stack

The idiomatic way to make a function cancellable is to accept a signal parameter and pass it down to any internal async operations:

```javascript
async function loadUserDashboard(userId, signal) {
  // Pass signal to all fetch calls
  const [user, posts] = await Promise.all([
    fetch(`/api/users/${userId}`, { signal }).then(r => r.json()),
    fetch(`/api/posts?userId=${userId}`, { signal }).then(r => r.json()),
  ]);

  signal.throwIfAborted();

  const enrichedPosts = await enrichPosts(posts, signal);

  return { user, enrichedPosts };
}

// Caller controls the controller
const controller = new AbortController();
cancelButton.onclick = () => controller.abort();

try {
  const dashboard = await loadUserDashboard(userId, controller.signal);
  render(dashboard);
} catch (err) {
  if (err.name !== 'AbortError') throw err;
}
```

### Pattern: Cancellable Polling

```javascript
async function pollUntilReady(jobId, { signal, interval = 1000 } = {}) {
  while (true) {
    signal?.throwIfAborted();

    const status = await fetch(`/api/jobs/${jobId}`, { signal })
      .then(r => r.json());

    if (status.complete) return status.result;
    if (status.failed) throw new Error(status.error);

    // Wait before next poll, but allow cancellation during wait
    await new Promise((resolve, reject) => {
      const id = setTimeout(resolve, interval);
      signal?.addEventListener('abort', () => {
        clearTimeout(id);
        reject(signal.reason ?? new DOMException('Aborted', 'AbortError'));
      }, { once: true });
    });
  }
}
```

### Pattern: Debounced Search with Cancellation

```javascript
let searchController = null;

async function handleSearchInput(query) {
  // Cancel previous in-flight request
  searchController?.abort();
  searchController = new AbortController();

  try {
    const results = await fetch(`/api/search?q=${encodeURIComponent(query)}`, {
      signal: searchController.signal,
    }).then(r => r.json());

    renderResults(results);
  } catch (err) {
    if (err.name !== 'AbortError') {
      renderError(err);
    }
    // AbortError means a newer search was triggered — silently ignore
  }
}
```

### Pattern: React Component Cleanup

```javascript
function UserProfile({ userId }) {
  const [user, setUser] = React.useState(null);

  React.useEffect(() => {
    const controller = new AbortController();

    fetch(`/api/users/${userId}`, { signal: controller.signal })
      .then(r => r.json())
      .then(data => setUser(data))
      .catch(err => {
        if (err.name !== 'AbortError') console.error(err);
      });

    // Cleanup: abort when component unmounts or userId changes
    return () => controller.abort();
  }, [userId]);

  return user ? <div>{user.name}</div> : <div>Loading...</div>;
}
```

## AbortController with addEventListener

The DOM's `addEventListener` accepts an `AbortSignal` to automatically remove the listener when aborted:

```javascript
const controller = new AbortController();

document.addEventListener('click', handleClick, { signal: controller.signal });
document.addEventListener('keydown', handleKeydown, { signal: controller.signal });
window.addEventListener('resize', handleResize, { signal: controller.signal });

// Remove ALL listeners at once
controller.abort();
```

This is a powerful pattern for managing event listener lifecycles, especially for single-use listeners or listeners tied to component lifetimes.

## Comparison Table

| Feature | AbortController | Manual Flags | setTimeout |
|---------|----------------|-------------|-----------|
| Native fetch support | Yes | No | No |
| Composable signals | Yes (AbortSignal.any) | No | No |
| Event listener removal | Yes | No | No |
| Standardized | Yes (WHATWG) | No | No |
| Automatic timeout | AbortSignal.timeout() | No | Yes |
| Cancels in-flight | Yes (fetch) | Depends | No |
| Reason propagation | Yes | Manual | No |

## Best Practices

### 1. Always Check for AbortError Before Re-Throwing

```javascript
try {
  const data = await fetch(url, { signal }).then(r => r.json());
} catch (err) {
  if (err.name === 'AbortError') return; // Expected — handle gracefully
  throw err; // Unexpected — propagate
}
```

### 2. Use signal.throwIfAborted() for Checkpoints in Long Loops

Insert checks at natural checkpoints — before each iteration or expensive computation:

```javascript
for (const batch of batches) {
  signal.throwIfAborted();
  await processBatch(batch);
}
```

### 3. Use AbortSignal.timeout() for Simple Timeouts

Avoid manual controller creation and `setTimeout` management when you just need a deadline:

```javascript
// Before
const controller = new AbortController();
const id = setTimeout(() => controller.abort(), 5000);
try {
  await fetch(url, { signal: controller.signal });
} finally {
  clearTimeout(id);
}

// After
await fetch(url, { signal: AbortSignal.timeout(5000) });
```

### 4. Propagate Signals Rather Than Creating New Controllers

When building an API that wraps async operations, accept and propagate a signal rather than creating your own:

```javascript
// Preferred
async function fetchUserPosts(userId, signal) {
  return fetch(`/api/posts?user=${userId}`, { signal }).then(r => r.json());
}

// Avoid: creates a separate unlinked controller
async function fetchUserPosts(userId) {
  const controller = new AbortController(); // Caller can't cancel this
  return fetch(`/api/posts?user=${userId}`, { signal: controller.signal }).then(r => r.json());
}
```

### 5. Do Not Reuse AbortControllers

Once aborted, an `AbortSignal` stays aborted permanently. Create a new `AbortController` for each operation:

```javascript
// Wrong
const controller = new AbortController();
await fetch(url1, { signal: controller.signal });
controller.abort();

await fetch(url2, { signal: controller.signal }); // Always immediately aborted!

// Correct
for (const url of urls) {
  const controller = new AbortController();
  await fetch(url, { signal: controller.signal });
}
```

## Interview Questions

### Q1: What is the difference between AbortController and a manual cancellation flag?

**Answer:** A manual flag (e.g., `let cancelled = false`) requires all internal code to check the flag manually and has no integration with browser APIs. `AbortController` provides a standardized interface that natively integrates with `fetch`, `addEventListener`, streams, and other platform APIs. It also supports composing multiple signals (`AbortSignal.any`), carries a reason for the abort, and emits an `'abort'` event that cooperating code can listen to.

### Q2: What happens to an in-flight fetch when you call controller.abort()?

**Answer:** The fetch Promise rejects with a `DOMException` with `name === 'AbortError'`. The underlying network request is cancelled if the browser hasn't yet sent it; if it has already been sent, the browser abandons waiting for the response but the server may already be processing it. The server is not notified of the cancellation — it continues processing unless it too has abort logic.

### Q3: Can you cancel a Promise?

**Answer:** Not directly — Promises have no cancellation mechanism. You can cancel the underlying async operation (e.g., a fetch using AbortController), which causes the Promise to reject with an AbortError. Custom Promises can implement cooperative cancellation by checking a signal at checkpoints. True preemptive Promise cancellation does not exist in JavaScript.

### Q4: What is AbortSignal.timeout() and when should you use it?

**Answer:** `AbortSignal.timeout(ms)` is a static factory that returns an `AbortSignal` that automatically fires after `ms` milliseconds. It eliminates the boilerplate of creating an `AbortController`, scheduling a `setTimeout`, calling `abort()`, and clearing the timeout in a `finally` block. Use it when you only need a simple deadline without needing to manually abort from other code paths.

### Q5: How do you properly remove multiple event listeners at once using AbortController?

**Answer:** Pass `{ signal: controller.signal }` as the options object to each `addEventListener` call. When `controller.abort()` is called, all listeners registered with that signal are automatically removed, as if `removeEventListener` had been called for each. This is cleaner than storing references to each handler and calling `removeEventListener` individually.

## Common Pitfalls

### 1. Re-Using an Aborted Controller

Once a controller is aborted, its signal is permanently in the aborted state. Creating a new `AbortController` for each operation is required.

### 2. Not Distinguishing AbortError from Other Errors

```javascript
// Bug: silently swallows real errors
fetch(url, { signal }).catch(() => {}); // Hides network errors too

// Fix: only swallow AbortError
fetch(url, { signal }).catch(err => {
  if (err.name !== 'AbortError') throw err;
});
```

### 3. Forgetting That the Server Continues Processing

Aborting a `fetch` cancels the client-side wait, not the server-side operation. For state-modifying operations (POST, PUT, DELETE), aborting may leave the server in an intermediate state. Design APIs to be idempotent or implement server-side cancellation separately.

### 4. Passing signal to fetch but Not Checking It in Custom Code

```javascript
async function processLargeDataset(data, signal) {
  const results = [];
  for (const item of data) {
    // Bug: signal not checked — loop runs to completion despite abort
    results.push(await transformItem(item));
  }
  return results;
}
```

Add `signal.throwIfAborted()` inside the loop.

## Resources

### Official Documentation
- [MDN: AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)
- [MDN: AbortSignal](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal)
- [MDN: AbortSignal.timeout()](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal/timeout_static)
- [MDN: AbortSignal.any()](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal/any_static)
- [WHATWG: AbortController specification](https://dom.spec.whatwg.org/#aborting-ongoing-activities)

### Articles and Guides
- [Abortable fetch](https://web.dev/fetch-api-error-handling/) — Google Web Dev
- [AbortController is your friend](https://whistlr.info/2021/abortcontroller-is-your-friend/) — Practical patterns
- [Don't cancel your fetch](https://medium.com/@steveruiz/dont-cancel-your-fetch-request-b92c5b9c00d4) — When NOT to cancel
