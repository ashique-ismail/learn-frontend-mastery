# Production Failure: State Sync Conflicts and Partial Writes

## The Idea

**In plain English:** State sync conflicts happen when the same piece of data lives in more than one place at the same time — like a browser tab, a server, and another user's screen — and those copies get out of step with each other. A "conflict" is when two places try to change the same data at once, and you have to decide whose change wins.

**Real-world analogy:** Imagine two people editing the same shared Google Doc while offline. Person A changes the title to "Project Alpha" and Person B changes it to "Project Beta" — both while disconnected. When they both sync back to the cloud, the system has to pick one title (or show both and ask). This is a state sync conflict.

- The Google Doc = the shared data on the server
- Each person's offline copy = the client-side state in a browser tab
- Both people hitting "sync" at the same time = two concurrent writes
- The cloud picking a winner (or showing a conflict warning) = optimistic concurrency control

---

## Overview

Distributed state — any scenario where the same data lives in multiple places (client + server, multiple tabs, multiple users) — is where some of the hardest production bugs live. These aren't theoretical: they're the bugs that make users say "I saved my work and it disappeared."

---

## Failure 1: The Lost Update (Concurrent Edits)

### The Scenario

Two users edit the same document field simultaneously:

```
User A reads:  { title: "Hello World", version: 1 }
User B reads:  { title: "Hello World", version: 1 }

User A writes: { title: "Hello React",  version: 1 } → server saves → version becomes 2
User B writes: { title: "Hello Angular", version: 1 } → server saves → version becomes 2
```

User A's change is silently overwritten. Neither user sees an error. The document now says "Hello Angular" — User A has no idea their work was lost.

### Detection: Optimistic Concurrency Control

The server should reject writes where the client's version doesn't match the current version:

```typescript
// Server: check version before writing
async function updateDocument(id: string, update: Partial<Doc>, clientVersion: number) {
  const current = await db.docs.findOne(id);
  if (current.version !== clientVersion) {
    throw new ConflictError({
      message: 'Document was modified by another user',
      serverVersion: current.version,
      clientVersion,
      serverValue: current,
    });
  }
  return db.docs.update(id, { ...update, version: current.version + 1 });
}
```

### Client Handling

```typescript
async function saveDocument(localDoc: Doc) {
  try {
    const saved = await api.update(localDoc.id, localDoc, localDoc.version);
    dispatch(documentSaved(saved));
  } catch (err) {
    if (err instanceof ConflictError) {
      // Show a merge UI: "Someone else edited this document"
      dispatch(conflictDetected({
        local: localDoc,
        remote: err.serverValue,
      }));
    }
  }
}
```

---

## Failure 2: The Phantom Write (Request Succeeds, Response Lost)

### The Scenario

```
Client sends:  POST /orders  { items: [...] }
Server processes: order created, ID = "ord-42"
Server sends:  200 { id: "ord-42" }
Network drops: client never receives the response
Client sees:   Error — assumes order was NOT created
Client retries: POST /orders { items: [...] }
Server creates: ANOTHER order — "ord-43"
User is charged twice.
```

This isn't hypothetical — Stripe's payment API handles millions of dollars of transactions where this failure mode would be catastrophic.

### Idempotency Keys

```typescript
// Client: generate a unique key per logical operation, store it
const idempotencyKey = crypto.randomUUID(); // persist this in sessionStorage
sessionStorage.setItem('checkout-key', idempotencyKey);

const response = await fetch('/api/orders', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Idempotency-Key': idempotencyKey, // ← include on every attempt
  },
  body: JSON.stringify(orderPayload),
});

// If the response arrives: clear the key
if (response.ok) sessionStorage.removeItem('checkout-key');
// If timeout/network error: retry with the SAME key
```

```typescript
// Server: check if this key was already processed
async function createOrder(payload: OrderPayload, idempotencyKey: string) {
  // Check if we've already processed this key
  const existing = await db.idempotencyKeys.findOne(idempotencyKey);
  if (existing) {
    return existing.result; // return the original response, don't process again
  }

  // Process the order
  const order = await db.orders.create(payload);

  // Store the result keyed by the idempotency key (expire after 24h)
  await db.idempotencyKeys.set(idempotencyKey, {
    result: order,
    expiresAt: Date.now() + 24 * 3600 * 1000,
  });

  return order;
}
```

### Frontend Idempotency for Mutations

```tsx
function useIdempotentMutation<T>(mutationFn: (key: string) => Promise<T>) {
  const keyRef = useRef<string | null>(null);

  return {
    mutate: async () => {
      // Reuse the same key on retry
      if (!keyRef.current) keyRef.current = crypto.randomUUID();
      try {
        const result = await mutationFn(keyRef.current);
        keyRef.current = null; // success — reset for next operation
        return result;
      } catch (err) {
        // Key preserved — retry will use it
        throw err;
      }
    },
  };
}
```

---

## Failure 3: Multi-Tab State Divergence

### The Scenario

```
Tab 1: User updates their profile photo
Tab 2: Still shows the old photo — state is stale
Tab 3: Shows the loading skeleton — fetching old data
```

Tab 1 invalidated its React Query cache. Tabs 2 and 3 have no idea. Each tab maintains its own in-memory state, completely isolated.

### Cross-Tab Sync Patterns

**BroadcastChannel API:**
```typescript
// In your mutation layer
const channel = new BroadcastChannel('cache-invalidation');

async function updateProfile(data: ProfileUpdate) {
  const result = await api.patch('/profile', data);
  queryClient.invalidateQueries(['profile']);
  // Tell other tabs to do the same
  channel.postMessage({ type: 'INVALIDATE', queryKey: ['profile'] });
  return result;
}

// On app init: listen for invalidation messages from other tabs
channel.onmessage = (event) => {
  if (event.data.type === 'INVALIDATE') {
    queryClient.invalidateQueries(event.data.queryKey);
  }
};
```

**localStorage event (works across tabs, same origin):**
```typescript
window.addEventListener('storage', (event) => {
  if (event.key === 'profile-updated') {
    queryClient.invalidateQueries(['profile']);
  }
});

// In mutation
localStorage.setItem('profile-updated', Date.now().toString());
```

**visibilitychange refetch (simpler but coarser):**
```typescript
// TanStack Query built-in: refetchOnWindowFocus: true
// When user returns to a tab, stale data is refreshed automatically
// Trade-off: more network requests, but always eventually consistent
```

---

## Failure 4: WebSocket Reconnection State Gap

### The Scenario

```
t=0:   Client connects to WebSocket. Server streams live updates.
t=60:  Network drops. Client disconnects.
t=90:  Client reconnects.

Events missed during gap (t=60 to t=90):
  - Item A was updated
  - Item B was deleted
  - Item C was created

Client shows stale state. No error. No indication anything was missed.
```

### The Fix: Sequence Numbers + Gap Fill

```typescript
class ReliableWebSocket {
  private lastSeq = 0;
  private ws: WebSocket | null = null;

  connect() {
    this.ws = new WebSocket(`wss://api.example.com/stream?since=${this.lastSeq}`);

    this.ws.onmessage = (event) => {
      const message = JSON.parse(event.data);

      if (message.seq <= this.lastSeq) return; // duplicate — ignore

      if (message.seq > this.lastSeq + 1) {
        // Gap detected — fetch missed events
        this.fillGap(this.lastSeq + 1, message.seq - 1);
      }

      this.lastSeq = message.seq;
      this.applyEvent(message);
    };

    this.ws.onclose = () => {
      setTimeout(() => this.connect(), 1000); // reconnect with lastSeq
    };
  }

  private async fillGap(from: number, to: number) {
    const missed = await fetch(`/api/events?from=${from}&to=${to}`).then(r => r.json());
    missed.forEach(event => this.applyEvent(event));
  }
}
```

---

## Failure 5: LocalStorage Schema Migration

### The Scenario

You store user preferences in `localStorage`:

```typescript
// v1: { theme: 'dark', fontSize: 16 }
localStorage.setItem('prefs', JSON.stringify({ theme: 'dark', fontSize: 16 }));
```

Two months later you ship v2 which changes the schema:

```typescript
// v2: { theme: { mode: 'dark', accent: 'blue' }, fontSize: 16 }
const prefs = JSON.parse(localStorage.getItem('prefs'));
prefs.theme.mode; // ❌ TypeError: Cannot read property 'mode' of undefined
                  // Old users still have prefs.theme = 'dark' (a string)
```

### Migration Pattern

```typescript
const PREFS_VERSION = 2;

function loadPrefs(): UserPrefs {
  const raw = localStorage.getItem('prefs');
  if (!raw) return defaultPrefs();

  let stored = JSON.parse(raw);

  // Migrate forward through versions
  if (!stored.__version || stored.__version < 1) {
    stored = migrateV0toV1(stored);
  }
  if (stored.__version < 2) {
    stored = migrateV1toV2(stored);
  }

  return stored;
}

function migrateV1toV2(old: V1Prefs): V2Prefs {
  return {
    __version: 2,
    theme: {
      mode: old.theme,      // 'dark' → { mode: 'dark', accent: 'blue' }
      accent: 'blue',
    },
    fontSize: old.fontSize,
  };
}
```

---

## The Mental Model for Distributed State

When designing any feature that touches shared state, run through this checklist:

```
1. CONCURRENCY
   - What if two users edit this simultaneously?
   - What if the same user edits in two tabs?
   - Use: optimistic concurrency control (version numbers)

2. NETWORK FAILURE
   - What if the request succeeds but the response is lost?
   - What if the connection drops mid-stream?
   - Use: idempotency keys, sequence numbers, gap-fill

3. STALENESS
   - When this data changes elsewhere, who else has a stale copy?
   - Use: BroadcastChannel, cache invalidation broadcast

4. PERSISTENCE
   - If the user refreshes, what state is lost?
   - If the schema changes, what breaks?
   - Use: schema versioning, migration functions

5. ORDERING
   - Can events arrive out of order?
   - Use: sequence numbers, vector clocks (for multi-author)
```

---

## Interview Questions

**Q: A user clicks "Save". The request times out. The UI shows an error. Was the data saved?**
A: Unknown — the timeout means the response was lost, not necessarily that the request failed. The correct answer is: check with an idempotency key. If the server received the request, it stored the result keyed by the idempotency key. A retry with the same key returns the original response without re-processing. This is why idempotency keys are the correct solution to at-most-once vs at-least-once delivery.

**Q: How do you keep state consistent across multiple open tabs?**
A: Three approaches depending on complexity: (1) `BroadcastChannel` for explicit invalidation messages — when tab A mutates, it broadcasts the query key to invalidate; (2) `localStorage` `storage` event — a write in one tab fires the event in all others; (3) TanStack Query's `refetchOnWindowFocus` — when the user returns to a tab, stale data is automatically refetched. For real-time apps: a shared WebSocket connection via a SharedWorker that broadcasts events to all tabs.

**Q: Walk me through what happens when a WebSocket disconnects and reconnects.**
A: You've missed events during the gap. The fix is sequence numbers on every server event. On reconnect, send your last-seen sequence number. The server replays missed events. If the server doesn't support replay, you fetch the gap via a REST endpoint. Without this, your client state silently diverges from the server — users see stale data with no indication anything is wrong.
