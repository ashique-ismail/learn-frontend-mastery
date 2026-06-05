# Local-First Architecture

## The Idea

**In plain English:** Local-first architecture means an app stores your data directly on your own device (like your phone or laptop) and only uses the internet to keep your devices in sync with each other — so the app works perfectly even when you have no internet connection, and your data belongs to you, not a company's server.

**Real-world analogy:** Imagine keeping a personal journal on paper that you carry with you everywhere. Whenever you visit a friend who also has a copy of the same journal, you compare notes and add anything the other person wrote that you missed.

- The paper journal = the local database stored on your device
- The act of comparing and copying entries with your friend = the sync process between devices and the cloud
- Being able to write in your journal even when your friend is away = working offline without needing a server

---

## Overview

Local-first software is a paradigm where the user's device holds the primary copy of data, and the cloud is used for sync and backup rather than as the authoritative source of truth. First articulated by Kleppmann et al. in the 2019 paper "Local-First Software: You Own Your Data, in Spite of the Cloud," it prioritizes: **offline capability**, **performance** (no network round-trips for reads/writes), **data ownership**, **privacy**, and **long-term archivability**.

```
Traditional cloud-first:
  Device ──────────────────► Cloud DB (authoritative)
  Device A reads ──────────► Cloud ──► response (100ms+)
  Offline? ──────────────────────────► Error / no data

Local-first:
  Device A [local DB] ◄──sync──► Cloud [relay]
  Device A reads ──────────────► local DB (< 1ms)
  Offline? ──────────────────────────► still works, syncs later
  Device B [local DB] ◄──sync──► Cloud [relay]
```

## The Seven Ideals of Local-First Software

1. **No spinners** — everything that doesn't require network is instant
2. **Works offline** — full app functionality without connectivity
3. **User data ownership** — data is yours, not locked in a SaaS
4. **Multi-device sync** — works across devices transparently
5. **Collaboration** — multiple people can work on the same document
6. **Security and privacy** — end-to-end encryption possible
7. **Longevity** — data survives even if the vendor shuts down

## IndexedDB as the Local Store

IndexedDB is the browser's native key-value/object database with support for indexes, transactions, and large data sets. It's asynchronous and works in workers.

### Raw IndexedDB (Verbose but Foundation)

```typescript
interface TodoItem {
  id: string;
  text: string;
  done: boolean;
  updatedAt: number;
}

function openDB(): Promise<IDBDatabase> {
  return new Promise((resolve, reject) => {
    const request = indexedDB.open('local-first-db', 1);

    request.onupgradeneeded = (event) => {
      const db = (event.target as IDBOpenDBRequest).result;

      // Create object store
      const store = db.createObjectStore('todos', { keyPath: 'id' });
      store.createIndex('updatedAt', 'updatedAt', { unique: false });
      store.createIndex('done', 'done', { unique: false });
    };

    request.onsuccess = () => resolve(request.result);
    request.onerror = () => reject(request.error);
  });
}

async function putTodo(db: IDBDatabase, todo: TodoItem): Promise<void> {
  return new Promise((resolve, reject) => {
    const tx = db.transaction('todos', 'readwrite');
    const store = tx.objectStore('todos');
    const request = store.put(todo);
    request.onsuccess = () => resolve();
    request.onerror = () => reject(request.error);
  });
}

async function getAllTodos(db: IDBDatabase): Promise<TodoItem[]> {
  return new Promise((resolve, reject) => {
    const tx = db.transaction('todos', 'readonly');
    const store = tx.objectStore('todos');
    const request = store.getAll();
    request.onsuccess = () => resolve(request.result);
    request.onerror = () => reject(request.error);
  });
}
```

### idb — Clean IndexedDB Wrapper

```typescript
import { openDB, DBSchema, IDBPDatabase } from 'idb';

interface MyDB extends DBSchema {
  todos: {
    key: string;
    value: TodoItem;
    indexes: {
      'by-updatedAt': number;
      'by-done': boolean;
    };
  };
  syncQueue: {
    key: string;
    value: {
      id: string;
      operation: 'upsert' | 'delete';
      data: TodoItem | null;
      createdAt: number;
    };
  };
}

const db = await openDB<MyDB>('local-first-db', 1, {
  upgrade(db) {
    const todoStore = db.createObjectStore('todos', { keyPath: 'id' });
    todoStore.createIndex('by-updatedAt', 'updatedAt');
    todoStore.createIndex('by-done', 'done');

    db.createObjectStore('syncQueue', { keyPath: 'id' });
  },
});

// Clean async API
await db.put('todos', { id: '1', text: 'Buy milk', done: false, updatedAt: Date.now() });
const all = await db.getAll('todos');
const recent = await db.getAllFromIndex('todos', 'by-updatedAt');
```

## Sync Queue Pattern

Every write to the local DB also adds to a sync queue. A background process drains the queue when online.

```typescript
class LocalFirstStore {
  private db: IDBPDatabase<MyDB>;
  private isSyncing = false;

  constructor(db: IDBPDatabase<MyDB>) {
    this.db = db;
    this.setupOnlineListener();
  }

  private setupOnlineListener() {
    window.addEventListener('online', () => {
      this.drainSyncQueue();
    });
  }

  // Write locally AND enqueue sync
  async upsertTodo(todo: Omit<TodoItem, 'updatedAt'>): Promise<void> {
    const fullTodo: TodoItem = { ...todo, updatedAt: Date.now() };

    // Atomic transaction: write to both stores
    const tx = this.db.transaction(['todos', 'syncQueue'], 'readwrite');
    await tx.objectStore('todos').put(fullTodo);
    await tx.objectStore('syncQueue').put({
      id: crypto.randomUUID(),
      operation: 'upsert',
      data: fullTodo,
      createdAt: Date.now(),
    });
    await tx.done;

    // Try to sync immediately if online
    if (navigator.onLine) {
      this.drainSyncQueue();
    }
  }

  async deleteTodo(id: string): Promise<void> {
    const tx = this.db.transaction(['todos', 'syncQueue'], 'readwrite');
    await tx.objectStore('todos').delete(id);
    await tx.objectStore('syncQueue').put({
      id: crypto.randomUUID(),
      operation: 'delete',
      data: null,
      createdAt: Date.now(),
    });
    await tx.done;

    if (navigator.onLine) {
      this.drainSyncQueue();
    }
  }

  async drainSyncQueue(): Promise<void> {
    if (this.isSyncing) return;
    this.isSyncing = true;

    try {
      const pending = await this.db.getAll('syncQueue');
      if (pending.length === 0) return;

      // Batch upload to server
      const response = await fetch('/api/sync/batch', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ operations: pending }),
      });

      if (!response.ok) throw new Error(`Sync failed: ${response.status}`);

      // Remove successfully synced items
      const tx = this.db.transaction('syncQueue', 'readwrite');
      for (const item of pending) {
        await tx.objectStore('syncQueue').delete(item.id);
      }
      await tx.done;

      console.log(`Synced ${pending.length} operations`);
    } catch (err) {
      console.error('Sync drain failed:', err);
      // Will retry on next online event or next call
    } finally {
      this.isSyncing = false;
    }
  }
}
```

## Conflict Resolution Strategies

When the same item is modified on two devices while offline, you get a conflict.

### Last-Write-Wins (LWW)

The simplest approach — the operation with the highest timestamp wins. Easy to implement, loses data.

```typescript
function mergeWithLWW(local: TodoItem, remote: TodoItem): TodoItem {
  return local.updatedAt >= remote.updatedAt ? local : remote;
}
```

### Three-Way Merge

Requires the common ancestor (the version before both changes were made):

```typescript
function threeWayMerge<T extends Record<string, unknown>>(
  ancestor: T,
  local: T,
  remote: T
): T {
  const result = { ...ancestor };

  for (const key of Object.keys(ancestor) as Array<keyof T>) {
    const localChanged = local[key] !== ancestor[key];
    const remoteChanged = remote[key] !== ancestor[key];

    if (localChanged && remoteChanged) {
      // True conflict — apply domain-specific logic
      // Default: remote wins (or prompt user)
      result[key] = remote[key] as T[typeof key];
    } else if (localChanged) {
      result[key] = local[key] as T[typeof key];
    } else if (remoteChanged) {
      result[key] = remote[key] as T[typeof key];
    }
    // Both unchanged: keep ancestor value
  }

  return result;
}
```

### CRDT-Based Resolution

Use a CRDT library (Yjs, Automerge) for automatic conflict-free merging. See `crdt-realtime-collaboration.md`.

## Replicache

Replicache is a commercial library that implements local-first with a server-side "poke" model. It uses optimistic mutations (apply locally immediately) with server-authoritative reconciliation.

```typescript
import Replicache, { WriteTransaction, ReadTransaction } from 'replicache';

const rep = new Replicache({
  name: 'my-user-id',
  licenseKey: 'your-license-key',
  pushURL: '/api/replicache/push',
  pullURL: '/api/replicache/pull',

  // Define mutations — run locally (optimistic) and server-side
  mutators: {
    async createTodo(tx: WriteTransaction, todo: TodoItem) {
      await tx.set(`todo/${todo.id}`, todo);
    },
    async updateTodo(tx: WriteTransaction, todo: Partial<TodoItem> & { id: string }) {
      const existing = await tx.get<TodoItem>(`todo/${todo.id}`);
      if (existing) {
        await tx.set(`todo/${todo.id}`, { ...existing, ...todo });
      }
    },
    async deleteTodo(tx: WriteTransaction, id: string) {
      await tx.del(`todo/${id}`);
    },
  },
});

// Call a mutator — runs locally immediately, then syncs to server
await rep.mutate.createTodo({
  id: crypto.randomUUID(),
  text: 'Buy milk',
  done: false,
  updatedAt: Date.now(),
});

// Subscribe to live data (reactive, re-runs when data changes)
rep.subscribe(
  async (tx: ReadTransaction) => {
    return tx.scan({ prefix: 'todo/' }).values().toArray();
  },
  {
    onData: (todos: TodoItem[]) => {
      // Update UI
      setTodos(todos);
    },
  }
);
```

### Replicache Server Push Handler (Next.js)

```typescript
// app/api/replicache/push/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { prisma } from '@/lib/prisma';

export async function POST(req: NextRequest) {
  const push = await req.json();
  const { clientGroupID, mutations } = push;

  for (const mutation of mutations) {
    const { name, args } = mutation;

    try {
      if (name === 'createTodo') {
        await prisma.todo.upsert({
          where: { id: args.id },
          create: args,
          update: args,
        });
      } else if (name === 'deleteTodo') {
        await prisma.todo.delete({ where: { id: args } });
      }
    } catch (e) {
      console.error('Mutation error:', name, e);
      // Continue — Replicache will retry
    }
  }

  return NextResponse.json({});
}
```

## ElectricSQL

ElectricSQL syncs a PostgreSQL database to SQLite in the browser or on the device. Queries run locally against SQLite, and the server syncs changes bidirectionally.

```typescript
import { ElectricClient } from 'electric-sql/client/model';
import { electrify } from 'electric-sql/wa-sqlite';
import { schema } from './generated/client';

// Initialize Electric with wa-sqlite (WebAssembly SQLite)
const conn = await electrify(schema, {
  url: 'https://your-electric-server.com',
});

const electric = conn.db;

// Subscribe to a shape — partial sync (only relevant rows)
await electric.syncItems({
  where: { user_id: currentUser.id },
});

// Query runs locally against SQLite — zero latency
const todos = await electric.items.findMany({
  where: { done: false },
  orderBy: { createdAt: 'desc' },
});

// Write — immediately visible locally, syncs in background
await electric.items.create({
  data: {
    id: crypto.randomUUID(),
    text: 'New task',
    done: false,
    userId: currentUser.id,
  },
});
```

## PouchDB

PouchDB is a JavaScript database inspired by CouchDB with built-in CouchDB replication:

```typescript
import PouchDB from 'pouchdb';

// Local database
const localDB = new PouchDB('my-todos');

// Sync with CouchDB-compatible server
const remoteDB = new PouchDB('https://your-couchdb.com/todos');

// Bidirectional live sync
const sync = localDB.sync(remoteDB, {
  live: true,
  retry: true,
}).on('change', (info) => {
  console.log('Synced:', info);
}).on('paused', () => {
  console.log('Sync paused (offline or caught up)');
}).on('error', (err) => {
  console.error('Sync error:', err);
});

// Write locally — syncs automatically
await localDB.put({
  _id: new Date().toISOString(),
  text: 'Hello PouchDB',
  done: false,
});

// Query with Mango (MongoDB-like) queries
const result = await localDB.find({
  selector: { done: false },
  sort: [{ _id: 'desc' }],
  limit: 20,
});
```

## PowerSync

PowerSync is a newer service that syncs a Postgres server to a local SQLite database, similar to ElectricSQL but with a managed cloud service:

```typescript
import { PowerSyncDatabase } from '@powersync/web';
import { PowerSyncConnector } from './connector';

const db = new PowerSyncDatabase({
  schema: AppSchema,
  database: { dbFilename: 'powersync.db' },
});

await db.connect(new PowerSyncConnector());

// React hook for live queries
import { useQuery } from '@powersync/react';

function TodoList() {
  const { data: todos } = useQuery<TodoItem>(
    'SELECT * FROM todos WHERE done = 0 ORDER BY created_at DESC'
  );

  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}>{todo.text}</li>
      ))}
    </ul>
  );
}
```

## Background Sync via Service Workers

Pair local-first storage with Service Worker Background Sync for reliable delivery:

```typescript
// service-worker.ts
self.addEventListener('sync', (event: SyncEvent) => {
  if (event.tag === 'sync-todos') {
    event.waitUntil(syncPendingTodos());
  }
});

async function syncPendingTodos() {
  // Read from IndexedDB and push to server
  const db = await openDB();
  const pending = await db.getAll('syncQueue');

  if (pending.length === 0) return;

  const response = await fetch('/api/sync/batch', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(pending),
  });

  if (response.ok) {
    const tx = db.transaction('syncQueue', 'readwrite');
    for (const item of pending) {
      tx.store.delete(item.id);
    }
    await tx.done;
  } else {
    throw new Error('Sync failed'); // SW will retry
  }
}

// main.ts — register sync when writing
async function writeTodoWithSync(todo: TodoItem) {
  await localDB.upsert(todo);

  const registration = await navigator.serviceWorker.ready;
  if ('sync' in registration) {
    await registration.sync.register('sync-todos');
  }
}
```

## Architecture Decision: Which Tool to Use?

```
Simple offline + sync?
  ├── Need SQL queries locally?   ──► ElectricSQL / PowerSync
  ├── Need CouchDB ecosystem?     ──► PouchDB
  ├── Need minimal bundle?        ──► idb + custom sync
  └── Need fine-grained reactivity + transactions?  ──► Replicache

Collaborative text editing?
  ──► Yjs or Automerge (see crdt-realtime-collaboration.md)

Enterprise / existing Postgres?
  ├── ElectricSQL (open source)
  └── PowerSync (managed cloud)
```

## Interview Questions

1. **What is the key philosophical difference between local-first and offline-first architectures?**
   Offline-first means an app gracefully degrades when offline (caches reads, queues writes). Local-first goes further: the local device is the primary copy; the server is a sync relay, not the source of truth. Local-first implies full offline capability and data ownership semantics. You could have offline-first without CRDTs (just caching), but local-first typically requires principled conflict resolution.

2. **What are the trade-offs of last-write-wins vs CRDT-based conflict resolution?**
   LWW is simple, requires no extra metadata, but silently discards changes — dangerous for important data. CRDTs are mathematically conflict-free but require per-character or per-field metadata (tombstones, vector clocks), increasing storage size 2–10x. CRDT algorithms also have higher CPU cost. The choice depends on data sensitivity and scale.

3. **How does Replicache's mutator model differ from a regular optimistic UI?**
   Regular optimistic UI maintains the speculative state in React state and reconciles with server response manually. Replicache's mutators run in two contexts: (1) locally for instant optimistic updates, and (2) on the server for canonical application. If the server result differs from the optimistic result, Replicache automatically re-runs local mutations on top of the server state and re-renders. This is called "speculative execution."

4. **Why is IndexedDB asynchronous, and what are the implications for UI code?**
   IndexedDB is asynchronous to avoid blocking the main thread (which would freeze the UI). Implications: reads must be awaited, which means they can't be synchronous in render functions. Libraries like Replicache solve this by maintaining an in-memory cache that's synchronously readable, backed by an async IndexedDB write. This pattern (sync read, async persist) is critical for local-first performance.

5. **What is a "shape" in ElectricSQL, and why is partial sync important?**
   A shape is a subscription to a subset of the database — e.g., all todos belonging to the current user. Full sync would require replicating the entire Postgres database to every client, which is impractical. Partial sync (shapes) allows each client to subscribe only to the data they need, reducing storage and bandwidth. Shapes can be dynamic — change as the user navigates.

6. **How do you handle schema migrations in an IndexedDB-backed local-first app?**
   IndexedDB uses a versioned schema. Increment the version number and provide an `onupgradeneeded` handler that transforms the old schema to the new one. The tricky part is migrating existing data: you must iterate old records, transform them, and write them to new stores. Libraries like `idb` make this manageable. For offline-first apps, you must handle all historical schema versions since users may open the app after months without updating.

7. **What security concerns arise from storing user data in IndexedDB?**
   IndexedDB is accessible to any JavaScript on the same origin — XSS attacks can exfiltrate all stored data. Mitigations: strong CSP, store only non-sensitive data or encrypted blobs, use Web Crypto API to encrypt before storage, never store auth tokens in IndexedDB (use `httpOnly` cookies instead). Data also persists across sessions, so ensure logout clears sensitive data with `indexedDB.deleteDatabase()`.

8. **Describe the "poke + pull" model used by Replicache and why it's more scalable than polling.**
   In poke + pull, the server sends a lightweight "poke" notification (WebSocket message or Server-Sent Event) when there's new data. The client then pulls the full changeset. This avoids: (1) polling overhead (constant requests even when no changes), (2) pushing large payloads over persistent connections. The pull can be served from a CDN or read replica, making it horizontally scalable. The poke is cheap — just an empty notification.
