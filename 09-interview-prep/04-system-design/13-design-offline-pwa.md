# Design an Offline-First PWA

## Core Technologies

1. **Service Worker** — intercepts network requests, implements caching
2. **Cache API** — stores request/response pairs
3. **IndexedDB** — stores structured application data offline
4. **Background Sync API** — queues mutations when offline, syncs on reconnect
5. **Web App Manifest** — enables install to home screen

---

## Caching Strategies

### Cache-First (for Static Assets)

```js
// sw.js
self.addEventListener('fetch', (event) => {
  if (isStaticAsset(event.request)) {
    event.respondWith(
      caches.match(event.request).then(cached => {
        return cached ?? fetch(event.request).then(response => {
          const clone = response.clone();
          caches.open('static-v1').then(cache => cache.put(event.request, clone));
          return response;
        });
      })
    );
  }
});
```

Use for: JS/CSS/images that change with each deploy (use versioned cache names).

### Network-First (for API Data)

```js
// Try network, fall back to cache
event.respondWith(
  fetch(event.request)
    .then(response => {
      const clone = response.clone();
      caches.open('api-v1').then(cache => cache.put(event.request, clone));
      return response;
    })
    .catch(() => caches.match(event.request))
);
```

Use for: API requests where fresh data is preferred but stale data is acceptable.

### Stale-While-Revalidate (for Frequently Updated Content)

```js
event.respondWith(
  caches.open('dynamic-v1').then(cache => {
    return cache.match(event.request).then(cached => {
      const networkFetch = fetch(event.request).then(response => {
        cache.put(event.request, response.clone());
        return response;
      });
      return cached ?? networkFetch; // return cache immediately, update in background
    });
  })
);
```

Use for: News feeds, product catalogs — show instantly, refresh in background.

---

## Offline Data with IndexedDB

```ts
import { openDB } from 'idb'; // lightweight wrapper

const db = await openDB('my-app', 1, {
  upgrade(db) {
    db.createObjectStore('todos', { keyPath: 'id' });
    db.createObjectStore('sync-queue', { autoIncrement: true });
  }
});

// Save locally
async function saveTodo(todo: Todo) {
  await db.put('todos', todo);
  if (!navigator.onLine) {
    await db.add('sync-queue', { type: 'create-todo', data: todo });
  } else {
    await api.createTodo(todo);
  }
}

// Read offline
async function getTodos(): Promise<Todo[]> {
  return db.getAll('todos');
}
```

---

## Background Sync for Mutations

```js
// sw.js — handle sync event (fired when connectivity restored)
self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-todos') {
    event.waitUntil(syncPendingTodos());
  }
});

async function syncPendingTodos() {
  const db = await openDB('my-app', 1);
  const pending = await db.getAll('sync-queue');
  
  for (const item of pending) {
    try {
      await fetch('/api/todos', {
        method: 'POST',
        body: JSON.stringify(item.data),
      });
      await db.delete('sync-queue', item.id);
    } catch {
      // Will retry on next sync event
    }
  }
}

// Register sync in client
async function queueTodoSync() {
  const registration = await navigator.serviceWorker.ready;
  await registration.sync.register('sync-todos');
}
```

---

## App Manifest

```json
// manifest.json
{
  "name": "My App",
  "short_name": "App",
  "start_url": "/",
  "display": "standalone",
  "theme_color": "#3b82f6",
  "background_color": "#ffffff",
  "icons": [
    { "src": "/icons/192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ]
}
```

---

## Service Worker Lifecycle & Update Flow

```
Install → Activate → Fetch (intercept requests)
         ↑
     New SW waits here if old SW is active

// Force new SW to take control
self.addEventListener('install', e => e.waitUntil(self.skipWaiting()));
self.addEventListener('activate', e => e.waitUntil(clients.claim()));
```

**Update notification pattern:**
```ts
const registration = await navigator.serviceWorker.register('/sw.js');
registration.addEventListener('updatefound', () => {
  const newWorker = registration.installing;
  newWorker.addEventListener('statechange', () => {
    if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
      showUpdateBanner(() => {
        newWorker.postMessage({ type: 'skipWaiting' });
        window.location.reload();
      });
    }
  });
});
```

---

## Conflict Resolution on Reconnect

When offline edits conflict with server state:

```ts
async function syncWithConflictResolution(localData: Todo, serverId: string) {
  const serverData = await api.getTodo(serverId);
  
  if (localData.updatedAt > serverData.updatedAt) {
    // Local is newer — push to server
    await api.updateTodo(serverId, localData);
  } else if (serverData.updatedAt > localData.updatedAt) {
    // Server is newer — update local
    await db.put('todos', serverData);
  } else {
    // Same timestamp — last-write-wins or show merge UI
    await showConflictDialog(localData, serverData);
  }
}
```

---

## Interview Discussion Points

**Q: What's the most important caching strategy decision?**
Understanding whether staleness is acceptable. Static assets → cache-first. User data → network-first. Content feeds → stale-while-revalidate. The wrong strategy either wastes bandwidth or serves stale data unexpectedly.

**Q: How do you handle cache versioning?**
Name caches with versions (`static-v2`). In the `activate` event, delete all caches except the current version. This ensures users get fresh assets after a deploy.

**Q: What's the difference between PWA and a regular web app?**
A PWA adds: installability (manifest), offline capability (service worker), and push notifications. It uses web standards to deliver app-like experiences without an app store.
