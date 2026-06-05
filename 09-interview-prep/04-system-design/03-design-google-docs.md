# Design Google Docs (Collaborative Editor)

## The Idea

**In plain English:** Google Docs is a text editor that lives on the internet and lets many people type in the same document at the same time, seeing each other's changes instantly — like a shared notebook where everyone's pen writes simultaneously without erasing each other's words.

**Real-world analogy:** Imagine a group of friends editing a shared whiteboard in a classroom — each person has a marker and can write wherever they want at the same time. When two people write in the same spot, a teacher (the server) steps in to decide whose word goes first and adjusts the spacing so both contributions fit without conflict.

- The whiteboard = the shared document stored on a server
- Each person's marker = a user's editor sending typed characters as operations
- The teacher adjusting positions = the conflict-resolution algorithm (OT or CRDT) that merges simultaneous edits

---

## Requirements

**Functional:**
- Multiple users edit the same document simultaneously
- Changes appear in real-time for all users
- Persistent storage — refresh and your changes are there
- Presence indicators (who else is editing)
- Cursor/selection sharing
- Undo/redo that works for each user independently

**Non-functional:**
- Sub-100ms latency for local edits
- Eventual consistency — all users converge to the same state
- Offline support — edit without internet, sync on reconnect

---

## The Core Problem: Conflict Resolution

User A types "Hello" at position 0.  
User B simultaneously types "World" at position 0.  
What's the final document?

Two approaches: **Operational Transformation (OT)** and **CRDTs**.

---

## Operational Transformation (OT)

Google Docs uses OT. Each operation (insert/delete at position) is **transformed** against concurrent operations before applying.

```
Initial: "Hello"
User A: Insert " World" at position 5  → "Hello World"
User B: Delete char at position 4     → "Hell"

If B's delete arrives after A's insert:
B's operation must be transformed: delete at position 4 is still valid → "Hell World"
If A's insert arrives after B's delete:
A's operation must be transformed: insert at position 4 (adjusted) → "Hell World"
```

**How it works:**
1. Client sends operation + current document version number to server
2. Server serializes all operations
3. If operation is behind, transform it against all intermediate operations
4. Broadcast transformed operation to all clients

**Complexity:** OT is notoriously hard to implement correctly, especially for rich text operations. Google's original OT code had many bugs before being replaced by a new implementation.

---

## CRDTs (Conflict-free Replicated Data Types)

Used by: Figma, Notion, Linear, VS Code Live Share (via Yjs).

CRDTs define data structures where concurrent operations **always commute** — the order you apply them doesn't matter, they always converge to the same result.

**For text:** Each character gets a unique, immutable ID. Operations reference character IDs rather than positions:

```
"Hello"
Each character: h[id:1], e[id:2], l[id:3], l[id:4], o[id:5]

User A: Insert " "[id:6] after id:5
User B: Insert "!"][id:7] after id:5

Both arrive in any order → "Hello !" (insert[id:6] and insert[id:7] after id:5 — tie-broken by id)
```

**Libraries:** [Yjs](https://yjs.dev) and [Automerge](https://automerge.org) are the production-ready options.

---

## Frontend Architecture with Yjs

```ts
import * as Y from 'yjs'
import { WebsocketProvider } from 'y-websocket'
import { QuillBinding } from 'y-quill'

// Shared document
const ydoc = new Y.Doc()

// WebSocket provider for real-time sync
const provider = new WebsocketProvider(
  'wss://collab-server.example.com',
  'document-id',
  ydoc
)

// Shared text type
const ytext = ydoc.getText('content')

// Bind to editor (Quill, Monaco, Prosemirror all have Yjs bindings)
const binding = new QuillBinding(ytext, quillEditor, provider.awareness)
```

---

## Presence and Cursors

```ts
// Awareness protocol (built into Yjs)
provider.awareness.setLocalStateField('user', {
  name: 'Alice',
  color: '#ff0000',
  cursor: { anchor: 45, head: 48 }, // selection range
})

// React to others' presence
provider.awareness.on('change', () => {
  const states = Array.from(provider.awareness.getStates().values())
  renderRemoteCursors(states)
})
```

---

## Offline Support

Yjs handles offline natively via its CRDT structure:
1. User edits offline — operations stored locally (IndexedDB)
2. On reconnect — Yjs exchanges state vectors with server
3. Server and client sync only the missing operations

```ts
import { IndexeddbPersistence } from 'y-indexeddb'

// Persist document locally
const persistence = new IndexeddbPersistence('document-id', ydoc)
persistence.on('synced', () => console.log('Loaded from IndexedDB'))
```

---

## Component Architecture

```
┌─────────────────────────────────────────┐
│  DocumentEditor                          │
│  ├── Toolbar (bold, italic, headings)    │
│  ├── EditorCanvas (Prosemirror/Quill)    │
│  │   └── Yjs binding                    │
│  ├── CursorLayer (remote cursors)        │
│  ├── PresenceBar (online users)          │
│  └── CommentsSidebar                     │
└─────────────────────────────────────────┘
```

---

## Undo/Redo in Collaborative Context

Standard undo stacks break in collaborative editing — undoing your own edit shouldn't undo someone else's concurrent edit.

**Solution:** Each user has their own undo manager that only tracks their operations:

```ts
const undoManager = new Y.UndoManager(ytext, {
  captureTimeout: 500, // group operations within 500ms
})

document.addEventListener('keydown', (e) => {
  if (e.ctrlKey && e.key === 'z') undoManager.undo()
  if (e.ctrlKey && e.shiftKey && e.key === 'z') undoManager.redo()
})
```

---

## Interview Discussion Points

**Q: Why CRDTs over OT for new implementations?**
CRDTs don't require a central server to serialize operations — they work peer-to-peer and offline natively. OT requires a central authority. CRDTs have more predictable correctness properties.

**Q: What's the bandwidth concern with CRDTs?**
CRDT metadata (character IDs) grows the document size over time. Tombstones (deleted characters) accumulate. Production systems compact the CRDT periodically to bound memory/bandwidth growth.

**Q: How do you handle network splits?**
Both OT and CRDTs are designed for this. Operations are queued offline and merged on reconnection. The CRDT guarantees eventual consistency regardless of how long the split lasted.
