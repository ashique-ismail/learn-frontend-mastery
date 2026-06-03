# Web Locks API

## Overview

The Web Locks API provides a mechanism for coordinating access to shared resources across browser tabs, Web Workers, and Service Workers within the same origin. It is the web platform's equivalent of a mutex or semaphore — ensuring that only one execution context holds a named "lock" at a time, preventing race conditions in multi-tab or multi-worker applications.

```
Problem: multi-tab race condition
  Tab A (user opens cart) → reads cart from IndexedDB → "2 items"
  Tab B (background sync) → reads cart from IndexedDB → "2 items"
  Tab B writes → "3 items" (added 1)
  Tab A writes → "3 items" (added 1 — but OVERWRITES Tab B's change)
  Result: cart has 3 items instead of 4. One add is lost.

Solution with Web Locks:
  Tab A requests lock('cart-write')
    └─► Granted → reads "2 items" → writes "3 items" → releases lock
  Tab B requests lock('cart-write') [queued while Tab A holds it]
    └─► Granted → reads "3 items" → writes "4 items" → releases lock
  Result: cart correctly has 4 items. No data loss.
```

## Core Concepts

### Lock Types

```
Exclusive lock (mode: 'exclusive') — the default:
  Only one holder at a time
  Use for: writes, single-instance enforcement, critical sections

Shared lock (mode: 'shared'):
  Multiple holders simultaneously
  No exclusive lock can be granted while shared locks are held
  Use for: reads, analytics, monitoring

Lock hierarchy (like a read-write lock):
  State          | Exclusive request | Shared request
  ───────────────|─────────────────────────────────────
  No lock held   | Granted           | Granted
  Shared held    | Queued            | Granted
  Exclusive held | Queued            | Queued
```

### The navigator.locks API

```typescript
// navigator.locks is available in:
//   - Main thread
//   - Web Workers
//   - Service Workers
// All within the same origin

// Basic lock acquisition
navigator.locks.request(
  'my-lock-name',   // The lock name — string identifier for the resource
  async (lock) => { // Callback — holds the lock while this runs
    // Critical section
    await doSomething();
    // Lock is automatically released when this function returns/throws
  }
);
```

## Basic Usage

### Exclusive Locks

```typescript
// Prevent concurrent database writes
async function updateUserProfile(userId: string, updates: Partial<UserProfile>): Promise<void> {
  await navigator.locks.request(
    `user-profile-${userId}`, // Scoped to specific user — other users' profiles can update concurrently
    async (lock) => {
      // Guaranteed: no other code holds this lock
      const existing = await db.getUserProfile(userId);
      const merged = { ...existing, ...updates, updatedAt: Date.now() };
      await db.setUserProfile(userId, merged);
    }
  );
}
```

```typescript
// Single-tab enforcement — prevent multiple instances of a process
async function startBackgroundSync(): Promise<void> {
  const lockName = 'background-sync-process';

  // Try to acquire without waiting — if already held, don't run
  const acquired = await navigator.locks.request(
    lockName,
    { ifAvailable: true }, // Don't queue — get lock only if immediately available
    async (lock) => {
      if (lock === null) {
        console.log('Background sync already running in another tab');
        return;
      }

      // This is the only tab running background sync
      await runBackgroundSync();
    }
  );
}
```

### Shared Locks (Read-Write Pattern)

```typescript
class ReadWriteLock {
  private readLockName: string;
  private writeLockName: string;

  constructor(resourceName: string) {
    this.readLockName = `${resourceName}-read`;
    this.writeLockName = `${resourceName}-write`;
  }

  async read<T>(fn: () => Promise<T>): Promise<T> {
    return navigator.locks.request(
      this.writeLockName,
      { mode: 'shared' }, // Multiple readers allowed simultaneously
      fn
    );
  }

  async write<T>(fn: () => Promise<T>): Promise<T> {
    return navigator.locks.request(
      this.writeLockName,
      { mode: 'exclusive' }, // Exclusive write — waits for all readers to finish
      fn
    );
  }
}

const catalogLock = new ReadWriteLock('product-catalog');

// Multiple tabs can read concurrently
async function readProductCatalog(): Promise<Product[]> {
  return catalogLock.read(async () => {
    return db.getAllProducts();
  });
}

// Only one write at a time, and writes wait for readers to finish
async function updateProductCatalog(products: Product[]): Promise<void> {
  return catalogLock.write(async () => {
    await db.setProducts(products);
  });
}
```

## Single-Tab Enforcement

A common real-world use case: ensure only one tab plays audio, manages a WebSocket connection, or runs a expensive polling loop:

```typescript
class SingleTabManager {
  private lockName: string;
  private controller: AbortController | null = null;

  constructor(private resourceName: string) {
    this.lockName = `single-tab-${resourceName}`;
  }

  async start(onPromoted: () => void, onDemoted: () => void): Promise<void> {
    this.controller = new AbortController();

    try {
      await navigator.locks.request(
        this.lockName,
        { signal: this.controller.signal },
        async (lock) => {
          // This tab is now the "leader" — call setup
          onPromoted();

          // Hold the lock indefinitely by never resolving
          // When this tab closes, the lock releases and another tab takes over
          await new Promise<void>((_, reject) => {
            this.controller!.signal.addEventListener('abort', () => {
              reject(new DOMException('Lock released by user', 'AbortError'));
            });
          });
        }
      );
    } catch (err) {
      if ((err as DOMException).name !== 'AbortError') throw err;
    } finally {
      onDemoted();
    }
  }

  stop(): void {
    this.controller?.abort();
  }
}

// Usage: only the "leader" tab manages the WebSocket connection
const tabManager = new SingleTabManager('websocket');

tabManager.start(
  () => {
    console.log('This tab is now the WebSocket leader');
    wsConnection.connect();
    wsConnection.onMessage(broadcastToOtherTabs);
  },
  () => {
    console.log('Another tab took over — disconnecting WebSocket');
    wsConnection.disconnect();
  }
);
```

## Lock with Timeout and Abort

```typescript
async function requestLockWithTimeout(
  lockName: string,
  timeoutMs: number,
  fn: (lock: Lock) => Promise<void>
): Promise<boolean> {
  const controller = new AbortController();

  // Abort after timeout
  const timeoutId = setTimeout(() => {
    controller.abort();
  }, timeoutMs);

  try {
    await navigator.locks.request(
      lockName,
      { signal: controller.signal },
      fn
    );
    return true; // Acquired and completed successfully
  } catch (err) {
    if ((err as DOMException).name === 'AbortError') {
      console.log(`Lock '${lockName}' not acquired within ${timeoutMs}ms`);
      return false; // Timed out waiting
    }
    throw err; // Re-throw unexpected errors
  } finally {
    clearTimeout(timeoutId);
  }
}

// Example: try to acquire lock for 500ms, otherwise skip
const didAcquire = await requestLockWithTimeout(
  'payment-processing',
  500,
  async () => {
    await processPayment(orderId);
  }
);

if (!didAcquire) {
  // Another tab/worker is already processing — show "processing" UI
  showProcessingMessage();
}
```

## Lock Queue Inspection

```typescript
// Query current lock state — useful for debugging
async function inspectLocks(): Promise<void> {
  const state = await navigator.locks.query();

  console.log('Held locks:', state.held);
  /*
  [
    { name: 'user-profile-123', mode: 'exclusive', clientId: 'abc-123' },
    { name: 'product-catalog', mode: 'shared', clientId: 'abc-123' },
    { name: 'product-catalog', mode: 'shared', clientId: 'def-456' }, // Two tabs reading
  ]
  */

  console.log('Pending locks:', state.pending);
  /*
  [
    { name: 'product-catalog', mode: 'exclusive', clientId: 'ghi-789' }, // Waiting for readers
  ]
  */
}
```

## Cross-Tab State Synchronization

Combine Web Locks with BroadcastChannel for safe multi-tab state sync:

```typescript
class CrossTabStore<T> {
  private channel: BroadcastChannel;
  private lockName: string;
  private localState: T;

  constructor(
    private storeName: string,
    private initialState: T,
    private persist: (state: T) => Promise<void>,
    private load: () => Promise<T>
  ) {
    this.channel = new BroadcastChannel(storeName);
    this.lockName = `cross-tab-store-${storeName}`;
    this.localState = initialState;

    // Listen for updates from other tabs
    this.channel.addEventListener('message', (event) => {
      if (event.data.type === 'STATE_UPDATE') {
        this.localState = event.data.state;
        this.onStateChange(this.localState);
      }
    });
  }

  async update(updater: (state: T) => T): Promise<void> {
    await navigator.locks.request(this.lockName, async () => {
      // Load latest state from persistent storage
      const current = await this.load();
      const next = updater(current);

      // Persist
      await this.persist(next);

      // Update local state
      this.localState = next;

      // Notify other tabs
      this.channel.postMessage({ type: 'STATE_UPDATE', state: next });
    });
  }

  get state(): T {
    return this.localState;
  }

  // Override this to react to state changes
  onStateChange(state: T): void {}
}

// Usage
const cartStore = new CrossTabStore<CartState>(
  'shopping-cart',
  { items: [], total: 0 },
  (state) => indexedDB.setItem('cart', state),
  () => indexedDB.getItem<CartState>('cart')
);

// Safely add item from any tab
await cartStore.update(cart => ({
  ...cart,
  items: [...cart.items, newItem],
  total: cart.total + newItem.price,
}));
```

## Service Worker Integration

Web Locks are available in Service Workers, enabling coordination between page threads and service workers:

```typescript
// service-worker.ts — coordinate cache writes with page reads
self.addEventListener('fetch', (event: FetchEvent) => {
  if (event.request.url.includes('/api/products')) {
    event.respondWith(handleProductRequest(event.request));
  }
});

async function handleProductRequest(request: Request): Promise<Response> {
  const cacheKey = request.url;

  // Shared lock: multiple requests can read simultaneously
  // Exclusive lock prevents reads while cache is being refreshed
  return navigator.locks.request(
    `cache-${cacheKey}`,
    { mode: 'shared' },
    async () => {
      const cached = await caches.match(request);
      if (cached) return cached;

      // Cache miss — need to fetch and write
      // Release shared lock and request exclusive
      // (In practice: use a separate lock for writes)
      const response = await fetch(request);
      await cacheResponse(cacheKey, response.clone());
      return response;
    }
  );
}

// In page: acquire exclusive lock when forcing cache refresh
async function refreshProductCache(): Promise<void> {
  await navigator.locks.request(
    `cache-/api/products`,
    { mode: 'exclusive' }, // Waits for all ongoing reads
    async () => {
      await caches.delete('products-cache');
      const fresh = await fetch('/api/products');
      const cache = await caches.open('products-cache');
      await cache.put('/api/products', fresh);
    }
  );
}
```

## Comparison with Alternatives

```
Mechanism              | Cross-tab | Cross-worker | Queuing  | Timeout  | Use case
───────────────────────|───────────|──────────────|──────────|──────────|──────────────────
Web Locks API          | Yes       | Yes          | Built-in | Via abort| Critical sections
BroadcastChannel       | Yes       | Yes          | No       | No       | Event broadcasting
SharedArrayBuffer      | Yes       | Yes          | No       | No       | Binary data sharing
  + Atomics.wait       | Yes       | Yes          | Yes      | Yes      | Low-level sync
localStorage events    | Yes       | No           | No       | No       | Simple notifications
IndexedDB transactions | No        | No           | Built-in | No       | DB-layer locking
sessionStorage         | No        | No           | No       | No       | Tab-local storage

Key distinctions:
  Web Locks: Application-level coordination of arbitrary async work
  Atomics: Synchronous, low-level, byte-level — for SharedArrayBuffer
  BroadcastChannel: One-way messaging, not locking — no mutual exclusion
  localStorage events: Only fires in OTHER tabs, not the writing tab
```

### BroadcastChannel vs Web Locks

```typescript
// BroadcastChannel: message passing (not locking)
const bc = new BroadcastChannel('app-events');
bc.postMessage({ type: 'USER_LOGGED_IN', userId: '123' }); // Fire and forget

// Web Locks: mutual exclusion
await navigator.locks.request('auth-state', async () => {
  await updateAuthState(); // Only ONE tab runs this at a time
});

// They complement each other:
// - Use Web Locks to safely write shared state
// - Use BroadcastChannel to notify other tabs of the change
```

## Browser Support

```
Web Locks API:
  Chrome 69+   — full support
  Edge 79+     — full support
  Firefox 96+  — full support
  Safari 15.4+ — full support

Feature detection:
  'locks' in navigator → true if supported

Workers:
  Available in: Web Workers, Service Workers, Shared Workers
  Not available in: Worklets (Paint, Audio, Animation)
```

```typescript
// Feature detection
if (!('locks' in navigator)) {
  console.warn('Web Locks API not supported — falling back to no-lock strategy');
  // Implement a fallback or skip coordination for older browsers
}
```

## Debugging Locks in DevTools

```typescript
// Debug helper: log lock state periodically
async function debugLocks(intervalMs = 2000): Promise<() => void> {
  const id = setInterval(async () => {
    const { held, pending } = await navigator.locks.query();

    if (held.length || pending.length) {
      console.group('Web Locks State');
      if (held.length) {
        console.log('Held:', held.map(l => `${l.name}(${l.mode})`).join(', '));
      }
      if (pending.length) {
        console.log('Pending:', pending.map(l => `${l.name}(${l.mode})`).join(', '));
      }
      console.groupEnd();
    }
  }, intervalMs);

  return () => clearInterval(id);
}

// In Chrome DevTools, you can also inspect locks:
// Application panel → Storage → Indexed DB... (not directly, but:)
// Application panel → Background Services → Background Fetch (locks are shown in newer DevTools)
```

## Common Pitfalls

### Lock Leak — Never Holding the Lock Forever Accidentally

```typescript
// WRONG — unhandled promise means lock is held forever if fetch throws
await navigator.locks.request('data', async () => {
  const data = await fetch('/api/data'); // If this rejects, the lock releases (good)
  // But if you catch inside and do something async that never resolves...
  try {
    await unreliableOperation();
  } catch (e) {
    await new Promise(() => {}); // NEVER RESOLVES — LOCK HELD FOREVER
  }
});

// CORRECT — always ensure the callback resolves or rejects
await navigator.locks.request('data', async (lock) => {
  try {
    await doWork();
  } catch (e) {
    console.error('Work failed, releasing lock:', e);
    // Just let the catch block end — lock is released when function returns
  }
});
```

### Lock Name Granularity

```typescript
// TOO COARSE — one lock for all users blocks everyone
await navigator.locks.request('user-data', async () => {
  await updateUser(userId); // All tabs for ALL users wait
});

// CORRECT — scoped to specific resource
await navigator.locks.request(`user-data-${userId}`, async () => {
  await updateUser(userId); // Only tabs for THIS user wait
});
```

### Deadlock Risk

```typescript
// DEADLOCK RISK — nested locks requested in different orders
async function tabA() {
  await navigator.locks.request('lock-1', async () => {
    await navigator.locks.request('lock-2', async () => { /* ... */ });
  });
}

async function tabB() {
  await navigator.locks.request('lock-2', async () => {   // Tab A holds lock-1, waiting for lock-2
    await navigator.locks.request('lock-1', async () => { // Tab B holds lock-2, waiting for lock-1
      /* DEADLOCK */
    });
  });
}

// SAFE — always acquire locks in the same order across all code paths
// Or: acquire both locks in a single request (not possible with current API — use a single compound lock name)
```

## Interview Questions

1. **What problem does the Web Locks API solve, and what are its two lock modes?**
   The Web Locks API provides mutual exclusion for asynchronous operations across execution contexts (tabs, workers, service workers) within the same origin. Without it, multiple tabs accessing shared resources (IndexedDB, cache, localStorage) can corrupt data through read-modify-write races. The two modes are: **exclusive** — only one holder allowed at a time, used for writes and critical sections; and **shared** — multiple holders allowed simultaneously, but an exclusive lock cannot be granted while shared locks are held. This implements the classic readers-writers lock pattern.

2. **How does `ifAvailable: true` differ from the default lock request behavior?**
   By default, `navigator.locks.request()` queues the request — if the lock is held, the callback waits until it's released. With `{ ifAvailable: true }`, the browser tries to acquire the lock immediately: if it's available, the callback is called with the lock object; if it's held, the callback is called immediately with `null` as the lock argument — the caller must check for null and handle the "not acquired" case. This is used for non-blocking checks: "run this only if no other instance is running, otherwise skip."

3. **Explain the relationship between Web Locks, BroadcastChannel, and IndexedDB for multi-tab state management.**
   These three APIs serve complementary roles: IndexedDB is the persistent shared storage (the "database" for cross-tab state); Web Locks provides mutual exclusion around read-modify-write operations on that storage (preventing lost updates); BroadcastChannel notifies other tabs of state changes (event propagation). A complete pattern: acquire a Web Lock, read from IndexedDB, compute the new state, write to IndexedDB, release the lock, then broadcast the new state via BroadcastChannel. Each tab updates its in-memory state from the broadcast without re-reading IndexedDB.

4. **How would you use the Web Locks API to implement "tab leader election" — ensuring only one tab runs a background process?**
   Request a lock with a well-known name using the default exclusive mode and a callback that never resolves (holds the lock for the tab's lifetime). The first tab to request the lock wins — it's the leader and runs the background process. Other tabs queue behind it. When the leader tab closes, the browser automatically releases the lock, and the next queued tab acquires it, becoming the new leader. To trigger cleanup on demotion, use `{ signal: controller.signal }` and abort the request when stepping down intentionally.

5. **What happens to a held lock when the tab that holds it crashes or is closed?**
   The browser automatically releases all locks held by a closed or crashed execution context. This is the critical safety property of the Web Locks API — there are no lock leaks from abrupt termination. Queued requests waiting for that lock will be granted to the next in queue. This is in contrast to manual mutex implementations (e.g., using localStorage to track a "locked" state), where a crash leaves the "locked" flag set forever, permanently blocking all other tabs.

6. **Compare Web Locks with `Atomics.wait()` for cross-tab synchronization. When would you use each?**
   `Atomics.wait()` operates on a `SharedArrayBuffer` — it's synchronous (blocks the calling thread), operates at the byte level, and requires transferring the SharedArrayBuffer to all threads. It's designed for tight loops in performance-critical code (WebAssembly, audio processing) and cannot be used on the main thread. `Web Locks API` is promise-based (async), operates at the application level with named string identifiers, works across tabs without shared memory setup, and is safe for main thread use. Choose Web Locks for application-level coordination (guarding database writes, preventing duplicate processes); choose Atomics for low-level binary data synchronization in workers.

7. **How does lock name scoping affect performance in a multi-user application?**
   Lock names are strings that define the resource being protected. A single global lock name (e.g., `'database-write'`) serializes ALL writes across all users in all tabs — Tab A updating User 1's profile blocks Tab B updating User 2's profile, even though there's no conflict. Scoped names (e.g., `'user-profile-${userId}'`) allow concurrent writes for different resources. The rule of thumb: the lock name should identify the smallest logical resource that must be accessed atomically. Over-scoping creates unnecessary serialization; under-scoping allows races. Use hierarchical naming (`'entity-type/entity-id/property-group'`) for complex resources.

8. **Describe a production scenario where missing cross-tab locking caused a data corruption bug and how Web Locks would fix it.**
   Scenario: A PWA with offline support stores order state in IndexedDB. The user opens the order in two tabs simultaneously (tab A to view, tab B to edit). Tab B reads the order (version 1), modifies the quantity, writes back (version 2). Concurrently, tab A's background sync reads the order (version 1), marks an item as shipped, writes back (version 1 + shipped = version 2). Tab A's write, which completed after Tab B's, overwrites the quantity change — the quantity update is lost. Fix: wrap every read-modify-write on order state in `navigator.locks.request('order-${orderId}', ...)`. Both operations queue on the same lock. The second operation reads the version written by the first, applies its change, and writes version 3 — no data loss.
