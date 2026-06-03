# CRDTs and Real-Time Collaboration

## Overview

Real-time collaborative editing — like Google Docs, Figma, or Notion — requires multiple users to edit the same data simultaneously without conflicts. Two algorithmic approaches dominate: **Operational Transformation (OT)** and **Conflict-free Replicated Data Types (CRDTs)**. CRDTs are the modern preferred approach because they don't require a central server to serialize operations, making them suitable for peer-to-peer and local-first architectures.

This guide covers CRDT theory, the Yjs and Automerge libraries, building presence and awareness features, and the engineering trade-offs at scale.

```
Without conflict resolution:
User A: "Hello World"  ──► insert "!" at pos 11  ──► "Hello World!"
User B: "Hello World"  ──► delete "World"         ──► "Hello !"
                                                          ↑ Lost "World"

With CRDT:
User A: insert "!" ──► converges ──► "Hello World!"  (same on all peers)
User B: delete "World" ──►              "Hello !"
After merge: "Hello !"  or "Hello World!" depending on semantics — always consistent
```

## Operational Transforms vs CRDTs

### Operational Transformation (OT)

OT transforms operations against concurrent operations to preserve intent. Used by Google Docs.

```
State: "AB"
User A: insert "X" at index 1  → "AXB"
User B: insert "Y" at index 1  → "AYB"

Server receives A's op first, then must transform B's op:
B_transformed: insert "Y" at index 2  → "AXYB"
```

**OT Problems:**
- Requires a central server to totally-order operations
- Transformation functions are notoriously hard to get right (especially for rich text)
- O(n²) complexity for convergence in some cases
- Peer-to-peer is extremely difficult

### CRDTs — Core Idea

A CRDT is a data structure where all concurrent updates can be merged without conflicts because:
1. Every update is **commutative** (A ∘ B = B ∘ A)
2. Every update is **associative** ((A ∘ B) ∘ C = A ∘ (B ∘ C))
3. Every update is **idempotent** (A ∘ A = A) — replaying the same op is safe

```typescript
// Conceptual: G-Counter CRDT (grow-only counter)
class GCounter {
  private counts: Map<string, number>;

  constructor(private nodeId: string) {
    this.counts = new Map([[nodeId, 0]]);
  }

  increment(): void {
    this.counts.set(this.nodeId, (this.counts.get(this.nodeId) ?? 0) + 1);
  }

  value(): number {
    let total = 0;
    for (const v of this.counts.values()) total += v;
    return total;
  }

  // Merge: take max of each node's count
  merge(other: GCounter): GCounter {
    const merged = new GCounter(this.nodeId);
    const allNodes = new Set([...this.counts.keys(), ...other.counts.keys()]);

    for (const node of allNodes) {
      merged.counts.set(
        node,
        Math.max(this.counts.get(node) ?? 0, other.counts.get(node) ?? 0)
      );
    }
    return merged;
  }
}

// This is commutative — merge(A, B) === merge(B, A)
// This is idempotent — merge(A, A) === A
```

### CRDT Types

| Type | Description | Use Case |
|------|-------------|----------|
| G-Counter | Grow-only counter | View counts, likes |
| PN-Counter | Increment/decrement | Shopping cart quantities |
| G-Set | Grow-only set | Tags, labels |
| 2P-Set | Add/remove set | Presence tracking |
| LWW-Register | Last-write-wins register | Config values |
| OR-Set | Observed-remove set | Concurrent delete/re-add |
| LSEQ / RGA | Sequence CRDT | Collaborative text editing |
| Map CRDT | Nested key-value | Document state |

## Yjs — Production CRDT Library

Yjs is the most widely-used CRDT library for collaborative applications. It implements the YATA algorithm for sequences and provides shared types that behave like native JS data structures.

### Installation and Basic Setup

```typescript
import * as Y from 'yjs';
import { WebsocketProvider } from 'y-websocket';

// Create a shared document
const ydoc = new Y.Doc();

// Connect to a sync server (y-websocket, PartyKit, Liveblocks, etc.)
const provider = new WebsocketProvider(
  'wss://your-server.com',
  'room-name',
  ydoc
);

provider.on('status', (event: { status: string }) => {
  console.log('Connection status:', event.status);
});
```

### Shared Types

```typescript
// Y.Map — shared key-value store
const config = ydoc.getMap<string>('config');
config.set('theme', 'dark');
config.set('language', 'en');

// Y.Array — shared ordered list
const todos = ydoc.getArray<{ text: string; done: boolean }>('todos');
todos.push([{ text: 'Build CRDT demo', done: false }]);

// Y.Text — collaborative text (the main one for editors)
const content = ydoc.getText('content');
content.insert(0, 'Hello, World!');

// Observe changes
content.observe((event: Y.YTextEvent) => {
  console.log('Text changed:', event.changes);
  // event.changes.delta — array of retain/insert/delete ops
});

// Y.Map of Y.Maps — nested shared state
const users = ydoc.getMap<Y.Map<string>>('users');
const userState = new Y.Map<string>();
userState.set('name', 'Alice');
userState.set('cursor', '42');
users.set('user-alice', userState);
```

### Transactions — Atomic Updates

Group multiple operations so observers fire once:

```typescript
// Without transaction: observers fire for each operation
config.set('theme', 'dark');     // observer fires
config.set('fontSize', '16px'); // observer fires again

// With transaction: single observer notification
ydoc.transact(() => {
  config.set('theme', 'dark');
  config.set('fontSize', '16px');
  todos.push([{ text: 'New task', done: false }]);
  // All changes dispatched atomically
}, 'optional-origin-tag');
```

### Integrating with CodeMirror 6

```typescript
import { EditorView } from '@codemirror/view';
import { basicSetup } from 'codemirror';
import { yCollab } from 'y-codemirror.next';

const ydoc = new Y.Doc();
const ytext = ydoc.getText('codemirror');
const provider = new WebsocketProvider('wss://...', 'room', ydoc);

const userColor = '#' + Math.floor(Math.random() * 0xFFFFFF).toString(16);
provider.awareness.setLocalStateField('user', {
  name: 'Alice',
  color: userColor,
  colorLight: userColor + '33',
});

const view = new EditorView({
  doc: ytext.toString(),
  extensions: [
    basicSetup,
    yCollab(ytext, provider.awareness),
  ],
  parent: document.getElementById('editor')!,
});
```

### Integrating with TipTap / ProseMirror

```typescript
import { useEditor } from '@tiptap/react';
import StarterKit from '@tiptap/starter-kit';
import Collaboration from '@tiptap/extension-collaboration';
import CollaborationCursor from '@tiptap/extension-collaboration-cursor';

const ydoc = new Y.Doc();
const provider = new WebsocketProvider('wss://...', 'room', ydoc);

const editor = useEditor({
  extensions: [
    StarterKit.configure({ history: false }), // Yjs handles history
    Collaboration.configure({ document: ydoc }),
    CollaborationCursor.configure({
      provider,
      user: { name: 'Alice', color: '#f783ac' },
    }),
  ],
});
```

## Awareness Protocol

Awareness is ephemeral state (cursor positions, user presence, selections) that does NOT need to persist. It's stored separately from the CRDT document with a different TTL.

```typescript
// Set local awareness state
provider.awareness.setLocalState({
  user: { id: 'alice', name: 'Alice', color: '#e74c3c' },
  cursor: { anchor: 42, head: 55 },
  isTyping: true,
});

// Clear on disconnect
window.addEventListener('beforeunload', () => {
  provider.awareness.setLocalState(null);
});

// Listen for awareness changes
provider.awareness.on('change', ({
  added,
  updated,
  removed,
}: {
  added: number[];
  updated: number[];
  removed: number[];
}) => {
  const states = provider.awareness.getStates();
  
  // Render all online users' cursors
  states.forEach((state, clientId) => {
    if (clientId === ydoc.clientID) return; // Skip self
    renderRemoteCursor(clientId, state);
  });

  // Remove cursors for disconnected users
  removed.forEach((clientId) => {
    removeRemoteCursor(clientId);
  });
});

// Get all online users
function getOnlineUsers() {
  const states = provider.awareness.getStates();
  return Array.from(states.values())
    .filter((s) => s?.user)
    .map((s) => s.user);
}
```

### Presence Indicator Component

```typescript
import { useState, useEffect } from 'react';

function PresenceIndicator({ awareness }: { awareness: any }) {
  const [users, setUsers] = useState<Array<{ name: string; color: string }>>([]);

  useEffect(() => {
    function updateUsers() {
      const states = awareness.getStates();
      const online = Array.from(states.values())
        .filter((s: any) => s?.user)
        .map((s: any) => s.user);
      setUsers(online);
    }

    awareness.on('change', updateUsers);
    updateUsers();

    return () => awareness.off('change', updateUsers);
  }, [awareness]);

  return (
    <div className="presence-stack">
      {users.slice(0, 5).map((user, i) => (
        <div
          key={i}
          className="avatar"
          style={{
            backgroundColor: user.color,
            marginLeft: i > 0 ? '-8px' : 0,
            zIndex: users.length - i,
          }}
          title={user.name}
        >
          {user.name[0].toUpperCase()}
        </div>
      ))}
      {users.length > 5 && (
        <div className="avatar-overflow">+{users.length - 5}</div>
      )}
    </div>
  );
}
```

## Automerge — Alternative CRDT Library

Automerge uses a document model closer to Immutable.js and is excellent for structured documents (not optimized for large text).

```typescript
import * as Automerge from '@automerge/automerge';

// Create initial document
interface Todo {
  items: Array<{ text: string; done: boolean }>;
  title: string;
}

let doc = Automerge.init<Todo>();

// Apply changes
doc = Automerge.change(doc, 'Add todo', (d) => {
  d.title = 'My Todo List';
  d.items = [];
  d.items.push({ text: 'Learn CRDTs', done: false });
});

// Serialize to binary
const binary = Automerge.save(doc);

// Deserialize
const loadedDoc = Automerge.load<Todo>(binary);

// Merge concurrent changes
let docA = Automerge.change(doc, (d) => {
  d.items[0].done = true;
});

let docB = Automerge.change(doc, (d) => {
  d.items.push({ text: 'Ship to production', done: false });
});

// Merge diverged docs — commutative!
const mergedAB = Automerge.merge(docA, docB);
const mergedBA = Automerge.merge(docB, docA);
// mergedAB deep-equals mergedBA ✓
```

### Automerge Repo (v2 — Production Pattern)

```typescript
import { Repo } from '@automerge/automerge-repo';
import { BrowserWebSocketClientAdapter } from '@automerge/automerge-repo-network-websocket';
import { IndexedDBStorageAdapter } from '@automerge/automerge-repo-storage-indexeddb';

const repo = new Repo({
  network: [new BrowserWebSocketClientAdapter('wss://sync.example.com')],
  storage: new IndexedDBStorageAdapter(),
});

// Create a document handle
const handle = repo.create<{ text: string }>({ text: '' });

// Or find existing
// const handle = repo.find<{ text: string }>(documentId);

// Subscribe to changes
handle.on('change', ({ doc }) => {
  console.log('Document updated:', doc);
});

// Make a change
handle.change((doc) => {
  doc.text = 'Hello, Automerge!';
});

// Get doc URL for sharing
console.log('Share this:', handle.url);
```

## Offline Support and Sync

```typescript
// Yjs with IndexedDB persistence for offline
import { IndexeddbPersistence } from 'y-indexeddb';

const ydoc = new Y.Doc();

// Persist locally — survives page refresh
const persistence = new IndexeddbPersistence('my-doc-id', ydoc);

persistence.on('synced', () => {
  console.log('Loaded from IndexedDB');
  // Now safe to connect to network provider
  const wsProvider = new WebsocketProvider('wss://...', 'room', ydoc);
});

// Track sync state
let isOnline = false;
const wsProvider = new WebsocketProvider('wss://...', 'room', ydoc, {
  connect: false, // Don't auto-connect
});

window.addEventListener('online', () => {
  isOnline = true;
  wsProvider.connect();
});

window.addEventListener('offline', () => {
  isOnline = false;
  wsProvider.disconnect();
  // Edits continue locally via IndexedDB — will sync when reconnected
});
```

## Undo/Redo in Collaborative Contexts

Collaborative undo is non-trivial: you should only undo YOUR OWN changes, not others'.

```typescript
import { UndoManager } from 'yjs';

const ytext = ydoc.getText('content');

// Scope undo to specific tracked types
const undoManager = new UndoManager(ytext, {
  captureTimeout: 500, // Group operations within 500ms into one undo step
});

// Undo only local user's changes
function undo() {
  undoManager.undo();
}
function redo() {
  undoManager.redo();
}

// Key bindings
document.addEventListener('keydown', (e) => {
  if (e.ctrlKey || e.metaKey) {
    if (e.key === 'z' && !e.shiftKey) { e.preventDefault(); undo(); }
    if (e.key === 'z' && e.shiftKey)  { e.preventDefault(); redo(); }
    if (e.key === 'y')                { e.preventDefault(); redo(); }
  }
});
```

## Scaling Considerations

```
Single-server WebSocket:
  All peers connect to one server
  Server relays all CRDT updates
  ✓ Simple   ✗ SPOF   ✗ Doesn't scale

Peer-to-peer via WebRTC:
  Peers sync directly with y-webrtc
  Signaling server required for discovery
  ✓ No server cost   ✗ NAT traversal   ✗ Not all clients reachable

Distributed sync (e.g., PartyKit, Liveblocks):
  Edge workers hold document state
  Clients sync to nearest edge node
  ✓ Low latency   ✓ Scalable   ✗ Vendor lock-in
```

### y-webrtc for P2P

```typescript
import { WebrtcProvider } from 'y-webrtc';

const ydoc = new Y.Doc();
const rtcProvider = new WebrtcProvider('room-name', ydoc, {
  signaling: ['wss://signaling.yjs.dev'],
  password: 'optional-room-password',
  // ICE servers for NAT traversal
  peerOpts: {
    iceServers: [
      { urls: 'stun:stun.l.google.com:19302' },
      { urls: 'turn:turn.example.com', credential: 'pass', username: 'user' },
    ],
  },
});
```

## Performance Benchmarks

| Library | 1k ops | 10k ops | Memory (1M chars) | Bundle Size |
|---------|--------|---------|-------------------|-------------|
| Yjs | ~1ms | ~10ms | ~2MB | 78KB gzipped |
| Automerge v2 | ~2ms | ~30ms | ~4MB | 230KB gzipped |
| ShareDB (OT) | N/A (server) | N/A | — | — |

Yjs is typically 2–10x faster than Automerge for text operations because it uses a more compact internal representation.

## Interview Questions

1. **What makes a CRDT "conflict-free"? What mathematical properties must it satisfy?**
   CRDTs must form a join-semilattice: a partial order where every pair of states has a unique least upper bound (join). The merge operation (⊔) must be commutative (A ⊔ B = B ⊔ A), associative ((A ⊔ B) ⊔ C = A ⊔ (B ⊔ C)), and idempotent (A ⊔ A = A). These properties guarantee convergence regardless of message ordering or retransmission.

2. **Why does Google Docs use OT instead of CRDTs? What are the trade-offs?**
   Google Docs was built before mature CRDT libraries existed. OT with a central server is simpler to implement for a client-server model and has well-understood semantics. CRDTs have higher per-operation metadata overhead (tombstones, vector clocks) and were historically seen as too heavy. Modern CRDTs (Yjs) have addressed most performance concerns.

3. **What is a tombstone in a CRDT sequence, and why can't you simply delete items?**
   In sequence CRDTs, deletion is implemented by marking an item as deleted (tombstone) rather than removing it. This is necessary because if two peers both have the item and one deletes it, the other must be able to receive the delete operation and apply it even if they've made subsequent insertions relative to the deleted item. Tombstones can be garbage-collected once all peers have acknowledged the deletion.

4. **Explain the difference between a state-based CRDT and an operation-based CRDT.**
   State-based (CvRDT): merge entire state objects; requires sending full state but simpler to reason about. Operation-based (CmRDT): send only the operation delta; more efficient over the wire but requires causal delivery (operations must arrive in causal order). Yjs uses operation-based internally.

5. **What is awareness in Yjs, and why is it stored separately from the main document?**
   Awareness is ephemeral state (cursors, presence, selections) that doesn't need persistence or full CRDT guarantees. It uses a simpler last-write-wins map per client ID and expires automatically when a client disconnects. Mixing it into the CRDT document would create unbounded growth from all the transient state changes.

6. **How would you implement collaborative undo without undoing other users' changes?**
   Use a per-user undo stack that tracks only that user's operations (using origin tags in Yjs transactions). When undoing, invert only the local user's operations. Yjs's `UndoManager` implements this by filtering operations by the document's `clientID`. The tricky case: if user A undoes a change that user B has since built upon — in this case, the undo may have unexpected effects on B's work.

7. **What happens in Yjs when two users simultaneously insert text at the same position?**
   Yjs uses the YATA algorithm which assigns each character a unique ID (clientID + clock). When two inserts collide at the same position, they're ordered deterministically by their unique IDs (e.g., lexicographic order of clientIDs). This is arbitrary but consistent — both users see the same final order.

8. **How do you handle CRDT document size growth over time (tombstone accumulation)?**
   Options: (1) Garbage collect tombstones once all known peers have acknowledged them — requires tracking each peer's vector clock. (2) Create document snapshots and start a new document from the snapshot, retiring the old one. (3) Use Automerge's built-in compaction. In practice, long-lived documents in Yjs grow at ~bytes per character per edit, which is manageable for typical editing sessions but problematic for months of edits.
