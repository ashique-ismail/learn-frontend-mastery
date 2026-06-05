# Storage: localStorage, sessionStorage, IndexedDB, and Cookies

## The Idea

**In plain English:** Browsers let websites save information directly on your device — like a notepad your browser keeps — so data can survive page reloads or even be shared across tabs without asking a server every time. "Storage" is the umbrella term for the different notepads browsers offer, each with its own rules about how much you can write, how long it stays, and who can read it.

**Real-world analogy:** Think of a hotel with different places to store your belongings. You check in and can use a small bedside drawer (localStorage), a temporary locker that gets emptied when you leave your room (sessionStorage), a large luggage room that can hold anything including suitcases (IndexedDB), and a wristband the hotel staff can read to know your preferences without you saying a word (cookies).

- The bedside drawer = `localStorage` (small, stays forever until you clean it out, anyone in that room can open it)
- The temporary locker = `sessionStorage` (gone when you check out of that tab/session)
- The luggage room = `IndexedDB` (holds lots of stuff in an organized way, you need to request it properly)
- The wristband = cookies (automatically shown to staff on every interaction, the hotel can also put it on you)

---

## Overview

Browsers provide multiple client-side storage mechanisms, each with different capacity limits, persistence rules, API styles, and use cases. Choosing the right storage primitive—and understanding their security implications—is a recurring senior-level interview topic. The four main mechanisms are: `localStorage`, `sessionStorage`, the `IndexedDB` database API, and HTTP cookies (accessible via `document.cookie` or the modern Cookie Store API).

## localStorage

### Characteristics

- Synchronous, string-only key-value store
- Persists until explicitly cleared (survives tab and browser restarts)
- Scoped to origin (`scheme + host + port`)
- ~5MB limit (browser-dependent)
- Accessible from any tab on the same origin

### API

```javascript
// Write
localStorage.setItem('user', JSON.stringify({ id: 42, name: 'Alice' }));
localStorage.setItem('theme', 'dark');

// Read
const raw = localStorage.getItem('user');    // string or null
const user = raw ? JSON.parse(raw) : null;

localStorage.getItem('missing');             // null (not undefined)

// Remove
localStorage.removeItem('theme');

// Clear all
localStorage.clear();

// Enumerate
localStorage.length;
localStorage.key(0);                         // key at index 0
for (let i = 0; i < localStorage.length; i++) {
  const key = localStorage.key(i);
  console.log(key, localStorage.getItem(key));
}
// Or via spread:
Object.entries(localStorage);               // [['key','value'], ...]
```

### Handling Serialization

```javascript
// Utility to avoid repetitive JSON handling
const storage = {
  get(key, fallback = null) {
    try {
      const raw = localStorage.getItem(key);
      return raw !== null ? JSON.parse(raw) : fallback;
    } catch {
      return fallback;
    }
  },
  set(key, value) {
    try {
      localStorage.setItem(key, JSON.stringify(value));
      return true;
    } catch (e) {
      // QuotaExceededError
      console.warn('localStorage full:', e);
      return false;
    }
  },
  remove(key) { localStorage.removeItem(key); },
};
```

### storage Event

```javascript
// Fires in OTHER tabs/windows on the same origin when localStorage changes
// Does NOT fire in the tab that made the change
window.addEventListener('storage', (e) => {
  console.log(e.key);        // key that changed (null if clear() was called)
  console.log(e.oldValue);   // previous value
  console.log(e.newValue);   // new value
  console.log(e.url);        // URL of the document that changed it
  console.log(e.storageArea); // localStorage or sessionStorage
});
```

## sessionStorage

### Characteristics

- Same synchronous API as `localStorage`
- Scoped to the tab/window — each tab has its own isolated storage
- Cleared when the tab is closed
- Survives page refreshes within the same tab
- NOT shared between tabs, even on the same origin

```javascript
// API is identical to localStorage
sessionStorage.setItem('step', '2');
sessionStorage.getItem('step');
sessionStorage.removeItem('step');
sessionStorage.clear();

// Session storage is isolated per tab
// Useful for multi-step forms where each tab should have independent state
```

## IndexedDB

### Characteristics

- Asynchronous, transactional, object-oriented database built into the browser
- Stores structured data including binary (Blobs, ArrayBuffers)
- Capacity: large (often hundreds of MB, sometimes GB)
- Supports indexes and cursors for querying
- Scoped to origin
- Available in Web Workers and Service Workers

### Opening a Database

```javascript
// Request to open (or create) a database
const request = indexedDB.open('MyApp', 2); // name, version

request.onupgradeneeded = (event) => {
  const db = event.target.result;
  const oldVersion = event.oldVersion; // 0 if new

  if (oldVersion < 1) {
    // Create object store (table) with keyPath as primary key
    const store = db.createObjectStore('users', { keyPath: 'id', autoIncrement: true });
    store.createIndex('by_email', 'email', { unique: true });
    store.createIndex('by_name',  'name',  { unique: false });
  }

  if (oldVersion < 2) {
    // Migration: add new store or index in version 2
    db.createObjectStore('sessions', { keyPath: 'token' });
  }
};

request.onsuccess = (event) => {
  const db = event.target.result;
  // Use db here
};

request.onerror = (event) => {
  console.error('DB error:', event.target.error);
};
```

### Promisified Wrapper

The native IDB API uses events; wrapping it in Promises makes it workable:

```javascript
function openDB(name, version, upgrade) {
  return new Promise((resolve, reject) => {
    const req = indexedDB.open(name, version);
    req.onupgradeneeded = (e) => upgrade(e.target.result, e.oldVersion);
    req.onsuccess = (e) => resolve(e.target.result);
    req.onerror = (e) => reject(e.target.error);
  });
}

function idbRequest(req) {
  return new Promise((resolve, reject) => {
    req.onsuccess = (e) => resolve(e.target.result);
    req.onerror = (e) => reject(e.target.error);
  });
}

// Usage
const db = await openDB('MyApp', 1, (db) => {
  db.createObjectStore('items', { keyPath: 'id' });
});

// Add
const tx = db.transaction('items', 'readwrite');
await idbRequest(tx.objectStore('items').add({ id: 1, name: 'Widget' }));

// Get
const tx2 = db.transaction('items', 'readonly');
const item = await idbRequest(tx2.objectStore('items').get(1));
```

### CRUD Operations

```javascript
// Add (fails if key exists)
const addTx = db.transaction('users', 'readwrite');
await idbRequest(addTx.objectStore('users').add({ name: 'Alice', email: 'a@b.com' }));

// Put (upsert — add or replace by key)
await idbRequest(addTx.objectStore('users').put({ id: 1, name: 'Alice Updated', email: 'a@b.com' }));

// Get by key
const getTx = db.transaction('users', 'readonly');
const user = await idbRequest(getTx.objectStore('users').get(1));

// Get by index
const byEmail = await idbRequest(
  getTx.objectStore('users').index('by_email').get('a@b.com')
);

// Get all
const all = await idbRequest(getTx.objectStore('users').getAll());

// Delete
const delTx = db.transaction('users', 'readwrite');
await idbRequest(delTx.objectStore('users').delete(1));

// Count
const count = await idbRequest(getTx.objectStore('users').count());
```

### Cursors for Iteration

```javascript
const tx = db.transaction('users', 'readonly');
const store = tx.objectStore('users');
const request = store.openCursor();

request.onsuccess = (e) => {
  const cursor = e.target.result;
  if (!cursor) return; // no more records

  console.log(cursor.key, cursor.value);
  cursor.continue(); // move to next
};
```

### Range Queries

```javascript
// IDBKeyRange for filtering
const range = IDBKeyRange.bound(10, 20);         // 10 <= key <= 20
const range2 = IDBKeyRange.lowerBound(10, true); // key > 10 (exclusive)
const range3 = IDBKeyRange.only(42);             // key === 42

const request = store.getAll(range);
const cursorReq = store.index('by_name').openCursor(IDBKeyRange.bound('A', 'M'));
```

## Cookies

### Characteristics

- Sent automatically with every matching HTTP request (unlike localStorage)
- Can be scoped by `Domain`, `Path`, `Secure`, `SameSite`, `HttpOnly`
- Max size ~4KB per cookie
- Can be set server-side (via `Set-Cookie` header) or client-side (`document.cookie`)
- `HttpOnly` cookies are inaccessible to JavaScript—server-only

### Reading and Writing via document.cookie

```javascript
// Setting a cookie
document.cookie = 'theme=dark; Path=/; Max-Age=31536000; SameSite=Lax; Secure';

// Reading — returns a single string of all cookies
document.cookie; // 'theme=dark; session=abc123'

// Parse into an object
function getCookies() {
  return document.cookie
    .split('; ')
    .filter(Boolean)
    .reduce((acc, pair) => {
      const [key, ...val] = pair.split('=');
      acc[decodeURIComponent(key)] = decodeURIComponent(val.join('='));
      return acc;
    }, {});
}

// Get single cookie
function getCookie(name) {
  return getCookies()[name] ?? null;
}

// Deleting — set Max-Age to 0 or Expires to the past
document.cookie = `theme=; Path=/; Max-Age=0`;
```

### Cookie Attributes

```javascript
// Max-Age (seconds) vs. Expires (date)
'session_id=xyz; Max-Age=3600'             // expires in 1 hour
'session_id=xyz; Expires=Fri, 31 Dec 2025 23:59:59 GMT'

// Scope
'pref=dark; Domain=example.com; Path=/app' // only /app and below

// Security
'token=abc; Secure'       // HTTPS only
'token=abc; HttpOnly'     // not accessible via JS (server-set only)

// SameSite
'token=abc; SameSite=Strict'  // never sent cross-site
'token=abc; SameSite=Lax'     // sent on top-level navigations (default in modern browsers)
'token=abc; SameSite=None; Secure' // always sent cross-site (requires Secure)
```

### Cookie Store API (Modern)

```javascript
// Available in Service Workers and modern browsers
const cookie = await cookieStore.get('session_id');
cookie?.value;

const allCookies = await cookieStore.getAll();

await cookieStore.set({
  name: 'pref',
  value: 'dark',
  maxAge: 3600,
  sameSite: 'lax',
});

await cookieStore.delete('pref');

// Change events
cookieStore.addEventListener('change', (e) => {
  for (const changed of e.changed) console.log('changed:', changed.name);
  for (const deleted of e.deleted) console.log('deleted:', deleted.name);
});
```

## Comparison Table

| Feature | localStorage | sessionStorage | IndexedDB | Cookie |
|---------|-------------|----------------|-----------|--------|
| Persistence | Until cleared | Tab session | Until cleared | Configurable |
| Capacity | ~5MB | ~5MB | ~50MB–unlimited | ~4KB |
| Data types | Strings | Strings | Any structured | Strings |
| API style | Synchronous | Synchronous | Asynchronous | Synchronous |
| Accessible in Workers | No | No | Yes | Yes (Store API) |
| Sent with requests | No | No | No | Yes |
| Server-settable | No | No | No | Yes |
| Shared across tabs | Yes | No | Yes | Yes |

## Best Practices

### 1. Never Store Sensitive Data in localStorage

```javascript
// Bad — accessible to any JS on the page (XSS risk)
localStorage.setItem('auth_token', jwt);

// Prefer HttpOnly cookies for auth tokens — inaccessible to JS
// Set-Cookie: auth_token=jwt; HttpOnly; Secure; SameSite=Strict
```

### 2. Use IndexedDB for Large or Structured Data

```javascript
// Use idb (tiny wrapper) to avoid callback hell
import { openDB } from 'idb';

const db = await openDB('app', 1, {
  upgrade(db) { db.createObjectStore('cache'); },
});

await db.put('cache', responseData, cacheKey);
const data = await db.get('cache', cacheKey);
```

### 3. Handle QuotaExceededError

```javascript
try {
  localStorage.setItem('key', largeValue);
} catch (e) {
  if (e.name === 'QuotaExceededError') {
    clearStaleCache();
    localStorage.setItem('key', largeValue); // retry
  }
}
```

### 4. Validate and Sanitize Data From Storage

```javascript
// Data from storage could be stale or corrupted
const raw = localStorage.getItem('settings');
try {
  const settings = raw ? JSON.parse(raw) : defaultSettings;
  // Validate shape
  if (typeof settings.theme !== 'string') throw new Error('invalid');
  return settings;
} catch {
  localStorage.removeItem('settings');
  return defaultSettings;
}
```

## Interview Questions

### Q1: What is the key difference between localStorage and sessionStorage?

**Answer:** Both share the same synchronous API and are scoped to the origin, but `sessionStorage` is isolated per tab and cleared when the tab closes. `localStorage` persists indefinitely across tabs and browser restarts until explicitly cleared. Each tab has its own `sessionStorage` even on the same origin, making it suitable for independent multi-tab workflows.

### Q2: Why should auth tokens generally not be stored in localStorage?

**Answer:** `localStorage` is accessible to any JavaScript running on the page, including code injected via XSS. A stolen token allows account takeover. `HttpOnly` cookies are inaccessible to JavaScript entirely—only the browser sends them with requests—which neutralizes this attack vector. The tradeoff is that cookies require CSRF protection, typically via `SameSite=Strict/Lax` or CSRF tokens.

### Q3: When would you choose IndexedDB over localStorage?

**Answer:** IndexedDB is appropriate when: (1) you need to store more than ~5MB; (2) you need to store binary data (Blobs, ArrayBuffers); (3) you need to query data by indexes rather than exact key lookup; (4) you need access from a Service Worker; (5) you need transactional writes. `localStorage` is simpler and adequate for small key-value settings and preferences.

### Q4: How does SameSite affect cookie behavior?

**Answer:** `SameSite=Strict` prevents cookies from being sent on any cross-site request, including top-level navigations from external sites. `SameSite=Lax` (the modern default) allows cookies on safe cross-site top-level navigations (GET from external link) but blocks cross-site subresource requests. `SameSite=None` sends cookies on all cross-site requests but requires `Secure`. `SameSite` is the primary CSRF defense for modern browsers.

### Q5: What happens if IndexedDB storage quota is exceeded?

**Answer:** The browser throws a `QuotaExceededError` (a `DOMException` with name `'QuotaExceededError'`). Unlike localStorage, IndexedDB quota is managed per origin and can be queried via `navigator.storage.estimate()`. You should handle this error, evict stale data, and retry. Persistent storage (`navigator.storage.persist()`) can request that the browser not evict data under storage pressure.

## Common Pitfalls

### 1. localStorage is Synchronous and Blocks the Main Thread

```javascript
// Reading/writing localStorage is synchronous I/O
// Avoid reading large values in hot paths (each frame, scroll handler, etc.)
// Cache the parsed value in a JS variable
const settings = localStorage.getItem('settings'); // read once
```

### 2. JSON.parse Failures

```javascript
// Data may be corrupted or from a different app version
const raw = localStorage.getItem('data');
const data = JSON.parse(raw); // throws SyntaxError if raw is malformed
// Always wrap in try/catch
```

### 3. Forgetting IndexedDB Transactions Are Short-Lived

```javascript
const tx = db.transaction('store', 'readwrite');
const store = tx.objectStore('store');

// BAD: awaiting something unrelated keeps the transaction open until it auto-commits
await fetch('/api'); // error: transaction may close before the next IDB request
store.add({ id: 2 }); // likely fails with 'TransactionInactiveError'

// GOOD: do all IDB work within the transaction before awaiting anything else
const data = await fetch('/api').then(r => r.json());
const tx2 = db.transaction('store', 'readwrite');
tx2.objectStore('store').add(data);
```

### 4. Cookies Not Sent Due to SameSite Mismatch

```javascript
// Embedding your API on a third-party site and expecting cookies
// without SameSite=None; Secure will fail silently in modern browsers
```

## Resources

- [MDN: Web Storage API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Storage_API)
- [MDN: IndexedDB API](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API)
- [MDN: Using IndexedDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API/Using_IndexedDB)
- [MDN: document.cookie](https://developer.mozilla.org/en-US/docs/Web/API/Document/cookie)
- [MDN: Cookie Store API](https://developer.mozilla.org/en-US/docs/Web/API/Cookie_Store_API)
- [idb library](https://github.com/jakearchibald/idb) — Promise-based IndexedDB wrapper
- [Storage for the Web (web.dev)](https://web.dev/storage-for-the-web/)
