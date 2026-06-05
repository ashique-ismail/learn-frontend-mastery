# Promise Practical Patterns

## The Idea

**In plain English:** Promise patterns are a set of proven recipes for handling tricky situations when your code has to wait for things — like fetching data from the internet or reading a file. A "Promise" is just JavaScript's way of saying "I'll give you the result later, I promise."

**Real-world analogy:** Imagine you're a restaurant manager juggling orders during a busy Friday dinner rush. You have rules for common problems: if the kitchen fails, retry the order; if two waiters ask for the same dish at once, make it only once and share it; if you need 10 dishes done but only have 3 stoves, run 3 at a time.

- The restaurant orders = async tasks (fetching data, uploading files)
- The kitchen failing and retrying = the retry-with-backoff pattern
- Two waiters asking for the same dish = in-flight request deduplication
- Running 3 stoves at a time = controlled concurrency (N tasks at once)

---

## Overview

This file is a cookbook of real-world Promise patterns. It assumes you already understand the basics (states, chaining, `.catch`) and focuses on recurring problems developers face: handling HTTP errors properly, retrying failures, deduplicating requests, processing items sequentially, and more.

Each pattern is self-contained and can be adapted directly to production code.

---

## 1. Proper HTTP Error Handling

`fetch` only rejects on network failure — a 404 or 500 response resolves the Promise. You must check `response.ok` yourself:

```javascript
async function apiFetch(url, options) {
  const response = await fetch(url, options);

  if (!response.ok) {
    const body = await response.text();
    const error = new Error(`HTTP ${response.status}: ${response.statusText}`);
    error.status = response.status;
    error.body = body;
    throw error;
  }

  return response.json();
}

// Usage
apiFetch('/api/users/42')
  .then(user => render(user))
  .catch(err => {
    if (err.status === 404) showNotFound();
    else if (err.status === 401) redirectToLogin();
    else showGenericError(err);
  });
```

**Why attach `.status` to the Error:** Downstream `.catch` handlers can inspect `err.status` to branch without parsing the message string.

---

## 2. Retry with Exponential Backoff

Automatically retry a failed async operation with increasing delays between attempts:

```javascript
async function withRetry(fn, { attempts = 3, baseDelay = 300 } = {}) {
  let lastError;

  for (let i = 0; i < attempts; i++) {
    try {
      return await fn();
    } catch (err) {
      lastError = err;
      if (i < attempts - 1) {
        await new Promise(resolve => setTimeout(resolve, baseDelay * 2 ** i));
      }
    }
  }

  throw lastError;
}

// Usage
const data = await withRetry(() => apiFetch('/api/data'), { attempts: 4, baseDelay: 500 });
// Waits: 500ms → 1000ms → 2000ms between attempts
```

**When to use:** Network requests, flaky third-party APIs, database connections on startup. Do not retry on 4xx errors (client mistakes that won't fix themselves on retry):

```javascript
async function withSmartRetry(fn, opts = {}) {
  return withRetry(async () => {
    try {
      return await fn();
    } catch (err) {
      // 4xx errors are permanent — don't retry
      if (err.status >= 400 && err.status < 500) throw err;
      throw err;
    }
  }, opts);
}
```

---

## 3. Deduplicating In-Flight Requests

If multiple callers request the same resource at the same time, send only one network request and share the result:

```javascript
const inFlight = new Map();

function fetchOnce(url) {
  if (inFlight.has(url)) {
    return inFlight.get(url); // Return the existing Promise
  }

  const promise = fetch(url)
    .then(res => res.json())
    .finally(() => inFlight.delete(url)); // Clean up after settlement

  inFlight.set(url, promise);
  return promise;
}

// Three simultaneous calls → one network request
const [a, b, c] = await Promise.all([
  fetchOnce('/api/config'),
  fetchOnce('/api/config'),
  fetchOnce('/api/config'),
]);
```

**Useful for:** Config loading, user profile fetching, any read-heavy resource.

---

## 4. Promise-Based Cache (with TTL)

Cache results and only re-fetch after a time-to-live expires:

```javascript
function createCache(ttlMs = 60_000) {
  const store = new Map();

  return {
    async get(key, fetchFn) {
      const cached = store.get(key);
      if (cached && Date.now() < cached.expiresAt) {
        return cached.value;
      }

      const value = await fetchFn();
      store.set(key, { value, expiresAt: Date.now() + ttlMs });
      return value;
    },
    invalidate(key) {
      store.delete(key);
    },
  };
}

const cache = createCache(30_000); // 30-second TTL

async function getUser(id) {
  return cache.get(`user:${id}`, () => apiFetch(`/api/users/${id}`));
}
```

---

## 5. Sequential Processing (One at a Time)

`Promise.all` runs everything in parallel. When you need strict ordering or want to avoid overwhelming a resource, process items one at a time:

```javascript
async function processSequentially(items, fn) {
  const results = [];
  for (const item of items) {
    results.push(await fn(item));
  }
  return results;
}

// Usage: upload files one by one
const receipts = await processSequentially(files, file => uploadFile(file));
```

**With reduce (functional style):**

```javascript
const results = await items.reduce(
  (chain, item) => chain.then(acc => fn(item).then(r => [...acc, r])),
  Promise.resolve([])
);
```

---

## 6. Controlled Concurrency (N at a Time)

Between "all parallel" and "all sequential" — run at most N tasks at once:

```javascript
async function mapWithLimit(items, fn, limit = 5) {
  const results = new Array(items.length);
  let nextIndex = 0;

  async function worker() {
    while (nextIndex < items.length) {
      const i = nextIndex++;
      results[i] = await fn(items[i]);
    }
  }

  const pool = Array.from({ length: Math.min(limit, items.length) }, worker);
  await Promise.all(pool);
  return results;
}

// Process 200 URLs, max 10 concurrent
const pages = await mapWithLimit(urls, fetchPage, 10);
```

---

## 7. Converting Callbacks to Promises

Wrap any Node.js-style callback API (`(err, result) => void`) into a Promise:

```javascript
function promisify(fn) {
  return function (...args) {
    return new Promise((resolve, reject) => {
      fn(...args, (err, result) => {
        if (err) reject(err);
        else resolve(result);
      });
    });
  };
}

const readFile = promisify(fs.readFile);
const content = await readFile('./config.json', 'utf8');
```

Node.js ships `util.promisify` for this purpose — the pattern above is its conceptual implementation.

---

## 8. Waiting for a DOM Event Once

Turn a one-time event into a Promise:

```javascript
function waitForEvent(target, eventName, { signal } = {}) {
  return new Promise((resolve, reject) => {
    function handler(event) {
      resolve(event);
      cleanup();
    }

    function onAbort() {
      reject(new DOMException('Aborted', 'AbortError'));
      cleanup();
    }

    function cleanup() {
      target.removeEventListener(eventName, handler);
      signal?.removeEventListener('abort', onAbort);
    }

    target.addEventListener(eventName, handler, { once: true });
    signal?.addEventListener('abort', onAbort, { once: true });
  });
}

// Wait for a button click
const event = await waitForEvent(button, 'click');

// With a timeout
const controller = new AbortController();
setTimeout(() => controller.abort(), 5000);
try {
  await waitForEvent(document, 'visibilitychange', { signal: controller.signal });
} catch (err) {
  if (err.name === 'AbortError') console.log('Timed out');
}
```

---

## 9. Loading State Pattern

Always show a loading indicator and hide it when done, regardless of success or failure:

```javascript
async function loadWithState({ onStart, onSuccess, onError, onFinish }) {
  onStart();
  try {
    const result = await fetchData();
    onSuccess(result);
    return result;
  } catch (err) {
    onError(err);
  } finally {
    onFinish();
  }
}

// React-style usage
loadWithState({
  onStart:   () => setLoading(true),
  onSuccess: data => setData(data),
  onError:   err  => setError(err.message),
  onFinish:  () => setLoading(false),
});
```

---

## 10. Branching in a Promise Chain

Pass accumulating context through a chain when multiple steps need shared data:

```javascript
// Problem: later steps need data from earlier steps
fetchUser(id)
  .then(user => fetchProfile(user.profileId).then(profile => ({ user, profile })))
  .then(({ user, profile }) => fetchPosts(user.id).then(posts => ({ user, profile, posts })))
  .then(({ user, profile, posts }) => {
    render(user, profile, posts);
  })
  .catch(handleError);
```

**Simpler with async/await when steps share scope:**

```javascript
async function loadPage(id) {
  const user    = await fetchUser(id);
  const profile = await fetchProfile(user.profileId);
  const posts   = await fetchPosts(user.id);
  render(user, profile, posts);
}
```

---

## 11. Error Categorization

Define error classes so `.catch` handlers can branch by type without parsing strings:

```javascript
class NetworkError extends Error {
  constructor(message, status) {
    super(message);
    this.name = 'NetworkError';
    this.status = status;
  }
}

class ValidationError extends Error {
  constructor(message, fields) {
    super(message);
    this.name = 'ValidationError';
    this.fields = fields;
  }
}

// In a catch handler
async function save(data) {
  try {
    return await submitForm(data);
  } catch (err) {
    if (err instanceof ValidationError) {
      showFieldErrors(err.fields);
    } else if (err instanceof NetworkError && err.status === 503) {
      showRetryMessage();
    } else {
      throw err; // Re-throw unexpected errors
    }
  }
}
```

**Always re-throw errors you don't handle** — swallowing unknown errors hides bugs.

---

## 12. Executing Cleanup Regardless of Outcome

Use `.finally()` to guarantee teardown — closing connections, hiding spinners, releasing locks:

```javascript
async function withLock(lock, fn) {
  await lock.acquire();
  try {
    return await fn();
  } finally {
    lock.release(); // Always runs — even if fn() throws
  }
}

// File handle cleanup
async function processFile(path) {
  const handle = await fs.promises.open(path, 'r');
  try {
    const content = await handle.readFile('utf8');
    return parse(content);
  } finally {
    await handle.close();
  }
}
```

---

## 13. Promisifying a Timer

Useful for rate limiting, artificial delays in tests, or staggered animations:

```javascript
const delay = ms => new Promise(resolve => setTimeout(resolve, ms));

// Rate-limit API calls
async function rateLimitedFetch(urls, delayMs = 100) {
  const results = [];
  for (const url of urls) {
    results.push(await fetch(url).then(r => r.json()));
    await delay(delayMs);
  }
  return results;
}
```

---

## Pattern Decision Guide

| Situation | Pattern |
| --------- | ------- |
| Multiple independent requests needed | `Promise.all` |
| Some requests can fail, want all results | `Promise.allSettled` |
| Same data requested from multiple places at once | In-flight deduplication |
| Flaky external API | Retry with backoff |
| Must not overwhelm downstream service | Controlled concurrency |
| Need result from all, cache for short period | Promise cache with TTL |
| Operations depend on previous result | Sequential chain or async/await |
| Cleanup must always run | `.finally()` |
| Need to wait for a user interaction | Event-to-Promise conversion |
| Different actions for different error types | Custom error classes + instanceof |

## Resources

- [MDN: Using Promises — practical examples](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises)
- [We Have a Problem with Promises](https://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html) — Nolan Lawson
- [Node.js: util.promisify](https://nodejs.org/api/util.html#utilpromisifyoriginal)
