# Fetch API: Headers, Request, Response

## Overview

The Fetch API is the modern standard for making HTTP requests from the browser, replacing the verbose `XMLHttpRequest` (XHR) interface. It is Promise-based, composable, and exposes first-class `Request`, `Response`, and `Headers` objects that map closely to HTTP semantics. Senior interviews often explore error handling nuances, streaming, `AbortController` integration, request/response manipulation, and CORS behavior.

## Basic Usage

```javascript
// Minimal GET request
const response = await fetch('https://api.example.com/users');
const data = await response.json();

// Two-step pattern is by design:
// Step 1: network response (headers received)
// Step 2: body consumption (streaming body into a specific format)
```

### Why Two Awaits?

```javascript
// Step 1 resolves when headers arrive; the body may still be streaming
const res = await fetch('/large-file');
// res.ok, res.status, res.headers are already available here

// Step 2 reads and parses the body
const blob = await res.blob(); // could stream megabytes
```

## The Response Object

```javascript
const res = await fetch('/api/data');

// Status
res.status;       // 200, 404, 500, etc.
res.statusText;   // 'OK', 'Not Found', etc.
res.ok;           // true if status is 200-299

// Headers
res.headers.get('content-type');
res.headers.get('x-request-id');

// Body consumption methods (each can only be called once per response)
await res.json();        // Parse body as JSON
await res.text();        // Body as string
await res.blob();        // Body as Blob (binary)
await res.arrayBuffer(); // Body as ArrayBuffer (raw bytes)
await res.formData();    // Body as FormData

// Body stream (manual consumption)
res.body; // ReadableStream

// Check if already consumed
res.bodyUsed; // true after any body method has been called

// Clone a response to consume it multiple times
const clone = res.clone();
const [json1, text1] = await Promise.all([res.json(), clone.text()]); // error — can't read both
// Correct approach:
const resA = await fetch('/api');
const resB = resA.clone();
const json = await resA.json();
const text = await resB.text();
```

### fetch Does Not Reject on HTTP Errors

```javascript
// fetch only rejects on network failure (no connection, DNS failure, etc.)
// HTTP 4xx/5xx responses resolve normally with res.ok === false

async function safeFetch(url, options) {
  const res = await fetch(url, options);
  if (!res.ok) {
    const errorBody = await res.text();
    throw new Error(`HTTP ${res.status}: ${errorBody}`);
  }
  return res;
}
```

## The Request Object

```javascript
// Construct a reusable Request object
const request = new Request('https://api.example.com/items', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`,
  },
  body: JSON.stringify({ name: 'Widget', price: 9.99 }),
  mode: 'cors',         // 'cors' | 'no-cors' | 'same-origin'
  credentials: 'same-origin', // 'omit' | 'same-origin' | 'include'
  cache: 'no-store',    // 'default' | 'no-store' | 'no-cache' | 'reload' | 'force-cache'
  redirect: 'follow',   // 'follow' | 'error' | 'manual'
  referrerPolicy: 'strict-origin-when-cross-origin',
  signal: controller.signal,
});

const res = await fetch(request);
```

### Cloning Requests

```javascript
// Requests, like Responses, are single-use
const req = new Request('/api', { method: 'POST', body: JSON.stringify(data) });

// To retry or log, clone before use
async function fetchWithRetry(req, retries = 3) {
  for (let i = 0; i < retries; i++) {
    const res = await fetch(req.clone()); // clone for each attempt
    if (res.ok) return res;
  }
  throw new Error('Request failed after retries');
}
```

## The Headers Object

```javascript
// Construction
const headers = new Headers({
  'Content-Type': 'application/json',
  'X-Custom': 'value',
});

// Mutation
headers.set('Authorization', 'Bearer token');  // replace
headers.append('Accept', 'application/json');  // add (allows multiple values)
headers.delete('X-Custom');

// Reading
headers.get('content-type');  // case-insensitive
headers.has('authorization'); // boolean
headers.getSetCookie();       // ['Set-Cookie: ...' , ...]

// Iteration
for (const [name, value] of headers) {
  console.log(`${name}: ${value}`);
}
headers.forEach((value, name) => console.log(name, value));
[...headers.entries()];
[...headers.keys()];
[...headers.values()];

// Note: in browser, some headers are guard-protected and cannot be set
// ('Content-Length', 'Host', 'Origin', etc.)
```

## Aborting Requests

```javascript
const controller = new AbortController();
const { signal } = controller;

// Attach signal to request
const fetchPromise = fetch('/api/heavy-data', { signal });

// Cancel after 5 seconds
const timeout = setTimeout(() => controller.abort(), 5000);

try {
  const res = await fetchPromise;
  clearTimeout(timeout);
  return await res.json();
} catch (err) {
  if (err.name === 'AbortError') {
    console.log('Request was aborted');
  } else {
    throw err;
  }
}
```

### Timeout Helper

```javascript
async function fetchWithTimeout(url, options = {}, ms = 5000) {
  const controller = new AbortController();
  const id = setTimeout(() => controller.abort(), ms);
  try {
    const res = await fetch(url, { ...options, signal: controller.signal });
    clearTimeout(id);
    return res;
  } catch (err) {
    clearTimeout(id);
    throw err;
  }
}
```

### AbortSignal.timeout (Modern)

```javascript
// Built-in timeout without manual AbortController
const res = await fetch('/api/data', {
  signal: AbortSignal.timeout(5000),
});
```

## Sending Different Body Types

```javascript
// JSON
await fetch('/api', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ key: 'value' }),
});

// FormData (multipart, including file upload)
const form = new FormData();
form.append('username', 'alice');
form.append('avatar', fileInput.files[0]);
await fetch('/upload', { method: 'POST', body: form });
// Do NOT set Content-Type manually — browser sets it with the boundary

// URL-encoded form
const params = new URLSearchParams({ q: 'search', page: '1' });
await fetch('/search', {
  method: 'POST',
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
  body: params,
});

// Raw binary (Blob or ArrayBuffer)
await fetch('/api/upload', {
  method: 'PUT',
  headers: { 'Content-Type': 'application/octet-stream' },
  body: arrayBuffer,
});
```

## Streaming Responses

```javascript
// Read a large response incrementally
async function streamDownload(url) {
  const res = await fetch(url);
  const reader = res.body.getReader();
  const decoder = new TextDecoder();
  let result = '';

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    result += decoder.decode(value, { stream: true });
    // update progress UI here
  }

  return result;
}

// Streaming JSON (NDJSON)
async function* streamNDJSON(url) {
  const res = await fetch(url);
  const reader = res.body.pipeThrough(new TextDecoderStream()).getReader();
  let buffer = '';

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    buffer += value;
    const lines = buffer.split('\n');
    buffer = lines.pop(); // incomplete last line
    for (const line of lines) {
      if (line.trim()) yield JSON.parse(line);
    }
  }
}
```

## CORS and Credentials

```javascript
// By default, fetch sends no cookies to cross-origin requests
const res = await fetch('https://api.other.com/data');
// No cookies, no Authorization header added automatically

// Send cookies with cross-origin request (server must allow it)
const res = await fetch('https://api.other.com/data', {
  credentials: 'include',
});
// Server must respond with:
// Access-Control-Allow-Origin: https://your-origin.com (not *)
// Access-Control-Allow-Credentials: true

// Same-origin: send cookies (default)
credentials: 'same-origin'

// Never send cookies
credentials: 'omit'
```

## Building a Fetch Wrapper / HTTP Client

```javascript
class HttpClient {
  #baseUrl;
  #defaultHeaders;

  constructor(baseUrl, defaultHeaders = {}) {
    this.#baseUrl = baseUrl;
    this.#defaultHeaders = defaultHeaders;
  }

  async #request(path, options = {}) {
    const url = `${this.#baseUrl}${path}`;
    const headers = new Headers({ ...this.#defaultHeaders, ...options.headers });

    const res = await fetch(url, { ...options, headers });

    if (!res.ok) {
      const body = await res.text();
      throw Object.assign(new Error(`HTTP ${res.status}`), { status: res.status, body });
    }

    const contentType = res.headers.get('content-type') ?? '';
    return contentType.includes('application/json') ? res.json() : res.text();
  }

  get(path, options)  { return this.#request(path, { ...options, method: 'GET' }); }
  post(path, data, options) {
    return this.#request(path, {
      ...options,
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });
  }
  put(path, data, options) {
    return this.#request(path, {
      ...options,
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });
  }
  delete(path, options) { return this.#request(path, { ...options, method: 'DELETE' }); }
}

const api = new HttpClient('https://api.example.com', {
  Authorization: `Bearer ${token}`,
});

const user = await api.get('/users/42');
await api.post('/users', { name: 'Bob' });
```

## Comparison Table

| Feature | fetch | XMLHttpRequest |
|---------|-------|----------------|
| API style | Promise-based | Callback/event |
| Streaming | Yes (ReadableStream) | Partial (progress events) |
| Abort | AbortController | xhr.abort() |
| Service Worker support | Yes | No |
| Progress tracking | No (upload: yes) | Yes (onprogress) |
| Synchronous mode | No | Yes (deprecated) |
| CORS | Same as XHR | Same as XHR |

## Best Practices

### 1. Always Check res.ok

```javascript
const res = await fetch(url);
if (!res.ok) throw new Error(`HTTP ${res.status}`);
```

### 2. Set Content-Type for JSON Requests

```javascript
// Without Content-Type, the server may not parse the body correctly
headers: { 'Content-Type': 'application/json' }
```

### 3. Use AbortController for User-Cancelable Operations

```javascript
// Allow users to cancel ongoing navigation, search, etc.
const ac = new AbortController();
cancelBtn.addEventListener('click', () => ac.abort());
const res = await fetch('/api/search', { signal: ac.signal });
```

### 4. Avoid Setting Content-Type for FormData

```javascript
// Browser must set the multipart boundary automatically
const body = new FormData();
// Do NOT add 'Content-Type' header — the browser will set it correctly
await fetch('/upload', { method: 'POST', body });
```

## Interview Questions

### Q1: Why does fetch not reject on 404 or 500 responses?

**Answer:** `fetch` rejects only on network-level failures (DNS resolution failure, no internet connection, request blocked by CORS preflight, etc.). HTTP error status codes (4xx, 5xx) represent valid HTTP responses—the server communicated a meaningful status—so `fetch` resolves. It is the developer's responsibility to check `response.ok` or `response.status` and throw accordingly.

### Q2: Why are there two await calls when using fetch?

**Answer:** The first `await` resolves when the HTTP response headers arrive. At this point the response status and headers are available but the body may still be streaming. The second `await` (on `.json()`, `.text()`, etc.) reads and parses the full response body. This separation allows you to make early decisions (like checking `res.ok`) before committing to consuming the potentially large body.

### Q3: How do you handle request cancellation in fetch?

**Answer:** Create an `AbortController`, pass its `signal` to the fetch options, and call `controller.abort()` when cancellation is needed. The fetch Promise rejects with an `AbortError`. For simple timeouts, `AbortSignal.timeout(ms)` can be used directly without manually managing a controller.

### Q4: What happens if you try to read a response body twice?

**Answer:** The second read throws a `TypeError: body stream already read`. Response bodies are `ReadableStream` instances and can only be consumed once. To read a response body multiple times, call `response.clone()` before the first read and read each clone independently.

### Q5: How do you send cookies to a cross-origin fetch request?

**Answer:** Set `credentials: 'include'` in the fetch options. The server must respond with `Access-Control-Allow-Credentials: true` and a specific (non-wildcard) `Access-Control-Allow-Origin`. Without these server-side headers, the browser will block the response.

## Common Pitfalls

### 1. Not Checking res.ok

```javascript
const res = await fetch('/api/data');
const json = await res.json(); // may be an error body, no exception thrown
```

### 2. Setting Content-Type for FormData

```javascript
// Breaks multipart boundary
fetch('/upload', {
  method: 'POST',
  headers: { 'Content-Type': 'multipart/form-data' }, // missing boundary!
  body: formData,
});
```

### 3. Double-Consuming the Body

```javascript
const res = await fetch('/api');
const text = await res.text();
const json = await res.json(); // TypeError: body stream already read
```

### 4. Using fetch in a Service Worker Without Error Handling

```javascript
// In a service worker, unhandled fetch errors may silently fail
self.addEventListener('fetch', (e) => {
  e.respondWith(
    fetch(e.request).catch(() => caches.match(e.request))
  );
});
```

## Resources

- [MDN: Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)
- [MDN: Using Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch)
- [MDN: Request](https://developer.mozilla.org/en-US/docs/Web/API/Request)
- [MDN: Response](https://developer.mozilla.org/en-US/docs/Web/API/Response)
- [MDN: Headers](https://developer.mozilla.org/en-US/docs/Web/API/Headers)
- [MDN: AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)
- [Streams API](https://developer.mozilla.org/en-US/docs/Web/API/Streams_API)
