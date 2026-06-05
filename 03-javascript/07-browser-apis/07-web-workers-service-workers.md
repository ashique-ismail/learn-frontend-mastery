# Web Workers, Service Workers, and SharedArrayBuffer

## The Idea

**In plain English:** Your browser runs JavaScript one task at a time on a single "main thread" — like one chef cooking every dish in a kitchen. Web Workers let you hire extra chefs (background threads) to handle heavy tasks without slowing down the main chef, while Service Workers act as a smart middleman between your browser and the internet, able to serve saved meals even when the kitchen (network) is closed.

**Real-world analogy:** Imagine a busy restaurant where one head chef (the main thread) takes orders, cooks food, and plates dishes all by themselves. To handle a big catering order without slowing down the dining room, they hire a back-kitchen crew and a pantry manager. The back-kitchen crew works independently on the big batch job without disturbing the head chef. The pantry manager intercepts every food request and decides whether to pull it from the pantry (cache) or order fresh ingredients (network), even keeping things running when the supplier is unavailable.

- The head chef = the main JavaScript thread (handles UI and user interactions)
- The back-kitchen crew = Web Workers (run CPU-heavy tasks in the background)
- The pantry manager = the Service Worker (intercepts network requests and manages the cache)
- The pantry = the browser cache (stored responses available offline)

---

## Overview

The browser's JavaScript runtime is single-threaded by design, but the platform provides three worker primitives for off-main-thread execution: **Web Workers** (dedicated background threads), **Service Workers** (network proxy and offline cache), and **Shared Workers** (shared across tabs). **SharedArrayBuffer** enables true shared memory between threads. Mastery of these APIs is a strong signal for senior roles, where interviewers expect knowledge of threading models, security constraints (COEP/COOP), and offline architecture.

## Web Workers (Dedicated Workers)

### Characteristics

- Run in a separate thread with no access to the DOM
- Communicate via message passing (`postMessage` / `onmessage`)
- Have access to: `fetch`, `IndexedDB`, `WebSockets`, `crypto`, `localStorage` (no), `setTimeout`, `setInterval`, `importScripts`, and most Web APIs
- No access to: `document`, `window`, `DOM`, `alert`, `localStorage`

### Creating a Worker

```javascript
// main.js
const worker = new Worker('./worker.js', { type: 'module' }); // module or classic

worker.postMessage({ task: 'processData', payload: largeArray });

worker.onmessage = (e) => {
  console.log('Result:', e.data);
};

worker.onerror = (e) => {
  console.error('Worker error:', e.message, e.filename, e.lineno);
};

// Terminate (stops execution immediately)
worker.terminate();
```

```javascript
// worker.js
self.onmessage = (e) => {
  const { task, payload } = e.data;
  if (task === 'processData') {
    const result = expensiveComputation(payload);
    self.postMessage(result);
  }
};

function expensiveComputation(data) {
  // CPU-intensive work — does not block the main thread
  return data.reduce((acc, n) => acc + n, 0);
}
```

### Transferable Objects

Transferring ownership avoids copying large buffers:

```javascript
// Copying 100MB buffer — slow
worker.postMessage(hugeBuffer);

// Transferring ownership — zero-copy (hugeBuffer becomes detached in main thread)
worker.postMessage(hugeBuffer, [hugeBuffer]);
console.log(hugeBuffer.byteLength); // 0 — transferred

// Types that can be transferred:
// ArrayBuffer, MessagePort, ImageBitmap, OffscreenCanvas, ReadableStream
```

### Structured Clone

Anything sent via `postMessage` is deep-cloned using the structured clone algorithm:

```javascript
// These types are cloneable (not all JS objects):
// Primitives, Date, RegExp, Blob, File, FileList, ArrayBuffer,
// TypedArray, Map, Set, Error, ImageData
// NOT cloneable: functions, DOM nodes, WeakMap, Symbol
worker.postMessage({ fn: () => {} }); // DataCloneError
```

### Worker Module Pattern

```javascript
// worker.js (module type)
import { processChunk } from './data-utils.js';

self.onmessage = async (e) => {
  const result = await processChunk(e.data);
  self.postMessage(result);
};
```

### Worker Pool

```javascript
class WorkerPool {
  #workers = [];
  #queue = [];

  constructor(url, size = navigator.hardwareConcurrency) {
    for (let i = 0; i < size; i++) {
      const worker = new Worker(url, { type: 'module' });
      worker.onmessage = (e) => this.#onResult(worker, e.data);
      this.#workers.push({ worker, busy: false });
    }
  }

  #onResult(worker, result) {
    const entry = this.#workers.find(w => w.worker === worker);
    entry.busy = false;
    if (entry.resolve) entry.resolve(result);

    const next = this.#queue.shift();
    if (next) this.#run(entry, next);
  }

  #run(entry, { data, resolve, reject }) {
    entry.busy = true;
    entry.resolve = resolve;
    entry.reject = reject;
    entry.worker.postMessage(data);
  }

  run(data) {
    return new Promise((resolve, reject) => {
      const idle = this.#workers.find(w => !w.busy);
      if (idle) {
        this.#run(idle, { data, resolve, reject });
      } else {
        this.#queue.push({ data, resolve, reject });
      }
    });
  }

  terminate() {
    this.#workers.forEach(({ worker }) => worker.terminate());
  }
}
```

## Shared Workers

A `SharedWorker` is accessible from multiple tabs, iframes, or workers on the same origin. Communication uses `MessagePort`.

```javascript
// main.js (in multiple tabs)
const shared = new SharedWorker('./shared-worker.js', { type: 'module' });
shared.port.start();
shared.port.postMessage({ type: 'subscribe', channel: 'notifications' });
shared.port.onmessage = (e) => console.log('notification:', e.data);

// shared-worker.js
const ports = new Set();

self.onconnect = (e) => {
  const port = e.ports[0];
  ports.add(port);
  port.start();

  port.onmessage = (e) => {
    // Broadcast to all connected tabs
    for (const p of ports) p.postMessage(e.data);
  };
};
```

## Service Workers

### Characteristics

- Act as a network proxy between the browser and the network
- Run independently of any page (lifecycle tied to registration, not page)
- Have no DOM access
- Must be HTTPS (or localhost)
- Scope-limited to the directory of the service worker file (and below)
- Used for: offline caching, background sync, push notifications, request interception

### Lifecycle

```
Install → Activate → Idle ↔ Fetch/Message
          ↑
     (waiting if old SW controls pages)
```

```javascript
// Registering a service worker
if ('serviceWorker' in navigator) {
  window.addEventListener('load', async () => {
    try {
      const reg = await navigator.serviceWorker.register('/sw.js', { scope: '/' });
      console.log('SW registered:', reg.scope);

      // Update found
      reg.onupdatefound = () => {
        const installing = reg.installing;
        installing.onstatechange = () => {
          if (installing.state === 'installed' && navigator.serviceWorker.controller) {
            console.log('New SW available — refresh to update');
          }
        };
      };
    } catch (e) {
      console.error('SW registration failed:', e);
    }
  });
}
```

### Service Worker File (sw.js)

```javascript
const CACHE_NAME = 'app-v2';
const STATIC_ASSETS = ['/index.html', '/main.js', '/style.css', '/offline.html'];

// Install: pre-cache static assets
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then(cache => cache.addAll(STATIC_ASSETS))
  );
  self.skipWaiting(); // activate immediately
});

// Activate: clean up old caches
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then(keys =>
      Promise.all(keys.filter(k => k !== CACHE_NAME).map(k => caches.delete(k)))
    )
  );
  self.clients.claim(); // take control of open pages immediately
});

// Fetch: network-first with cache fallback
self.addEventListener('fetch', (event) => {
  if (event.request.method !== 'GET') return; // pass through non-GET

  event.respondWith(
    fetch(event.request)
      .then(response => {
        const clone = response.clone();
        caches.open(CACHE_NAME).then(cache => cache.put(event.request, clone));
        return response;
      })
      .catch(() => caches.match(event.request)
        .then(cached => cached ?? caches.match('/offline.html'))
      )
  );
});
```

### Caching Strategies

```javascript
// Cache-first (good for static assets)
async function cacheFirst(request) {
  const cached = await caches.match(request);
  return cached ?? fetch(request);
}

// Network-first (good for API calls)
async function networkFirst(request) {
  try {
    const response = await fetch(request);
    const cache = await caches.open(CACHE_NAME);
    cache.put(request, response.clone());
    return response;
  } catch {
    return caches.match(request);
  }
}

// Stale-while-revalidate (good for user data)
async function staleWhileRevalidate(request) {
  const cache = await caches.open(CACHE_NAME);
  const cached = await cache.match(request);
  const fetchPromise = fetch(request).then(response => {
    cache.put(request, response.clone());
    return response;
  });
  return cached ?? fetchPromise;
}
```

### Background Sync

```javascript
// Register a sync event when offline
async function submitFormOffline(data) {
  const reg = await navigator.serviceWorker.ready;
  await saveToIndexedDB('pending-submissions', data);
  await reg.sync.register('submit-form');
}

// In sw.js
self.addEventListener('sync', (event) => {
  if (event.tag === 'submit-form') {
    event.waitUntil(
      getFromIndexedDB('pending-submissions').then(items =>
        Promise.all(items.map(item => fetch('/api/submit', {
          method: 'POST',
          body: JSON.stringify(item),
        })))
      )
    );
  }
});
```

### Push Notifications

```javascript
// Request permission and subscribe
const reg = await navigator.serviceWorker.ready;
const subscription = await reg.pushManager.subscribe({
  userVisibleOnly: true,
  applicationServerKey: urlBase64ToUint8Array(VAPID_PUBLIC_KEY),
});

// Send subscription to server
await fetch('/api/subscribe', {
  method: 'POST',
  body: JSON.stringify(subscription),
});

// In sw.js
self.addEventListener('push', (event) => {
  const data = event.data.json();
  event.waitUntil(
    self.registration.showNotification(data.title, {
      body: data.body,
      icon: '/icon.png',
    })
  );
});
```

## SharedArrayBuffer and Atomics

`SharedArrayBuffer` (SAB) enables true shared memory between the main thread and workers. Requires Cross-Origin Isolation (COOP/COEP headers).

### Required Headers (COEP/COOP)

```http
Cross-Origin-Embedder-Policy: require-corp
Cross-Origin-Opener-Policy: same-origin
```

```javascript
// Check if cross-origin isolated
console.log(crossOriginIsolated); // true if headers are set
```

### Creating and Using SharedArrayBuffer

```javascript
// main.js
const sab = new SharedArrayBuffer(4); // 4 bytes
const int32 = new Int32Array(sab);
int32[0] = 0; // initial value

const worker = new Worker('./worker.js', { type: 'module' });
worker.postMessage(sab); // SAB is shared, not cloned

// Worker updates int32[0]; read it without message passing
requestAnimationFrame(function tick() {
  console.log('Counter:', int32[0]);
  requestAnimationFrame(tick);
});
```

```javascript
// worker.js
self.onmessage = (e) => {
  const int32 = new Int32Array(e.data);
  for (let i = 0; i < 1000000; i++) {
    Atomics.add(int32, 0, 1); // atomic increment — race-free
  }
};
```

### Atomics

```javascript
const sab = new SharedArrayBuffer(16);
const ia = new Int32Array(sab);

// Atomic operations (all are race-condition safe)
Atomics.add(ia, 0, 5);         // ia[0] += 5, returns old value
Atomics.sub(ia, 0, 3);
Atomics.and(ia, 0, 0xFF);
Atomics.or(ia, 0, 0x01);
Atomics.xor(ia, 0, 0x01);
Atomics.store(ia, 0, 42);      // write
Atomics.load(ia, 0);           // read
Atomics.exchange(ia, 0, 99);   // write and return old value
Atomics.compareExchange(ia, 0, expected, replacement); // CAS

// Waiting (blocks the thread — only available in Workers, NOT main thread)
const result = Atomics.wait(ia, 0, 0); // sleep until ia[0] !== 0
// result: 'ok' | 'not-equal' | 'timed-out'

// Notify waiting workers
Atomics.notify(ia, 0, 1); // wake 1 waiter at index 0
```

## Comparison Table

| Feature | Dedicated Worker | Shared Worker | Service Worker |
|---------|-----------------|---------------|----------------|
| Scope | One page | All pages, same origin | Origin scope |
| Lifecycle | Page lifetime | While any page is open | Independent (survives page close) |
| DOM access | No | No | No |
| Multiple connections | No (1:1) | Yes | Via Clients API |
| Network interception | No | No | Yes |
| Background sync | No | No | Yes |
| Push notifications | No | No | Yes |
| Use case | CPU-bound tasks | Cross-tab state | Offline, caching, push |

## Best Practices

### 1. Always Feature-Detect Workers

```javascript
if (typeof Worker !== 'undefined') {
  const worker = new Worker('./worker.js');
} else {
  // Fallback: run synchronously on main thread
  const result = processDataSync(data);
}
```

### 2. Version Service Worker Caches

```javascript
const CACHE = 'app-v3'; // increment on each deploy
// Activate event cleans old caches automatically
```

### 3. Use Transferables for Large Buffers

```javascript
const buffer = new ArrayBuffer(100 * 1024 * 1024);
worker.postMessage(buffer, [buffer]); // zero-copy transfer
```

### 4. Handle skipWaiting Carefully

```javascript
// skipWaiting can cause version mismatches if old page JS
// calls new API endpoints. Only skip waiting if safe.
self.addEventListener('message', (e) => {
  if (e.data === 'SKIP_WAITING') self.skipWaiting();
});
// Trigger from page only when user confirms reload
```

## Interview Questions

### Q1: What is the difference between a Web Worker and a Service Worker?

**Answer:** A Web Worker is a dedicated background thread tied to the lifetime of the page that created it. It is used for CPU-intensive computation to avoid blocking the main thread. A Service Worker is an independent network proxy that runs separately from any page. It persists beyond page close (until idle termination), can intercept network requests, cache responses, and handle push notifications. It is not created by a single page but registered for an origin scope.

### Q2: How does postMessage avoid race conditions with shared data?

**Answer:** `postMessage` does not share references—it creates a deep clone of the data using the structured clone algorithm. Each thread has its own copy. Race conditions are not possible with copied data. If you need shared memory (via `SharedArrayBuffer`), you must use `Atomics` operations to synchronize access and prevent data races.

### Q3: Why is SharedArrayBuffer gated behind COEP/COOP headers?

**Answer:** After the Spectre vulnerability (2018), browsers restricted SAB because malicious pages could use high-resolution timers (achievable via shared memory) to perform speculative execution attacks on adjacent data. COOP (`Cross-Origin-Opener-Policy: same-origin`) isolates the browsing context group, and COEP (`Cross-Origin-Embedder-Policy: require-corp`) ensures all resources explicitly opt into cross-origin access. Together they establish a cross-origin isolated context where `performance.now()` precision is restored safely, re-enabling SAB.

### Q4: What is the stale-while-revalidate caching strategy?

**Answer:** Serve the cached response immediately (instant, possibly stale) while simultaneously fetching a fresh version in the background to update the cache for the next request. This gives the user fast responses while keeping content up to date. It is ideal for resources that change but where brief staleness is acceptable—news feeds, user profile data, etc.

### Q5: What does clients.claim() do in a Service Worker?

**Answer:** By default, a newly activated Service Worker only controls pages opened after it was registered. `clients.claim()` in the `activate` event causes the Service Worker to immediately take control of all open pages within its scope, without requiring a reload. Combined with `skipWaiting()`, this ensures the new Service Worker activates and controls pages immediately.

## Common Pitfalls

### 1. Trying to Access DOM in a Worker

```javascript
// ReferenceError — document and window do not exist in worker scope
self.onmessage = () => {
  document.getElementById('status').textContent = 'done'; // Error!
  // Fix: postMessage the result back to main thread
};
```

### 2. Service Worker Cache Versioning

```javascript
// Forgetting to update CACHE_NAME leaves stale assets cached forever
const CACHE = 'app-v1'; // Must increment on each deployment
```

### 3. Infinite Fetch Loop in Service Worker

```javascript
self.addEventListener('fetch', (event) => {
  // event.request.url may match the SW's own file!
  // Always filter out non-GET, chrome-extension, non-http, etc.
  if (!event.request.url.startsWith('http')) return;
  event.respondWith(networkFirst(event.request));
});
```

### 4. Race Condition Without Atomics

```javascript
// UNSAFE: both workers read-modify-write without atomics
const ia = new Int32Array(sab);
const val = ia[0]; // read
ia[0] = val + 1;   // write — may be overwritten by another thread

// SAFE:
Atomics.add(ia, 0, 1);
```

## Resources

- [MDN: Web Workers API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API)
- [MDN: Service Worker API](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API)
- [MDN: SharedArrayBuffer](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer)
- [MDN: Atomics](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics)
- [Service Worker Cookbook](https://serviceworke.rs/)
- [web.dev: Service Workers](https://web.dev/service-workers-cache-storage/)
- [Workbox](https://developer.chrome.com/docs/workbox/) — production-ready SW library
