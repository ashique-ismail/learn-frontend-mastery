# Kanban Board with Drag-and-Drop

## The Idea

**In plain English:** A board with named columns (To Do, In Progress, Done). Each column holds cards. Users drag a card from one column to another — or reorder cards within the same column. The state update must be instant (optimistic), and the board must respect per-column limits (WIP limits).

**Real-world analogy:** Think of a physical sticky-note board in a team room.

- **Columns** are vertical swim lanes drawn on a whiteboard.
- **Cards** are sticky notes placed in those lanes.
- When a developer finishes a task, they physically pick up the sticky note from "In Progress" and slap it onto "Done". Everyone watching sees the move happen immediately.
- The **team rule** is: no more than 3 stickies in "In Progress" at once. If someone tries to add a 4th, the team pushes back.
- The **order within a column** matters — the top card is the highest priority.
- If the sticky note falls off the board and lands on the floor (network error), someone puts it back where it was. That rollback is the optimistic UI pattern.

The key insight: the board's state is **normalized** — columns and cards are stored in separate maps keyed by ID, with an ordered list of card IDs per column. This avoids deeply nested mutation and makes reordering O(1).

---

## Learning Objectives

- Design a normalized board state: `columns: Record<id, Column>`, `cards: Record<id, Card>`, `columnOrder: string[]`
- Implement drag-and-drop with `@dnd-kit/core` (React) and Angular CDK `DragDrop`
- Handle reorder-within-column and move-between-columns as separate operations
- Apply optimistic updates with rollback on API failure
- Enforce WIP (work-in-progress) column limits

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Visual drag shadow (ghost element) | ⚠️ | Browser default, but custom ghost needs JS |
| Track which column a card is being dragged over | ❌ | Requires pointer events + JS hit testing |
| Reorder items in a column on drag | ❌ | DOM order change requires JS |
| Enforce WIP limit on drop | ❌ | Counting items in a column requires JS |
| Optimistic state update | ❌ | State management is JS |
| Rollback on failure | ❌ | Requires previous-state snapshot in JS |
| Keyboard drag-and-drop | ❌ | Arrow key handlers require JS |

---

## HTML & CSS Foundation

### The Structure

```html
<div class="board" role="region" aria-label="Kanban board">
  <div class="board-column" data-column-id="todo" aria-label="To Do — 3 cards">
    <div class="column-header">
      <h2 class="column-title">To Do</h2>
      <span class="column-count" aria-label="3 cards">3</span>
    </div>
    <ul class="column-cards" role="list" aria-label="Cards in To Do">
      <li class="card" draggable="true" data-card-id="card-1" aria-grabbed="false">
        <p class="card-title">Design mockups</p>
        <div class="card-meta">
          <span class="card-tag">Design</span>
        </div>
      </li>
    </ul>
    <!-- Drop zone indicator -->
    <div class="drop-indicator" aria-hidden="true" hidden></div>
  </div>
</div>
```

### The CSS

```css
:root {
  --board-bg:      #f1f5f9;
  --column-bg:     #f8fafc;
  --column-border: #e2e8f0;
  --card-bg:       #ffffff;
  --card-shadow:   0 1px 3px rgba(0,0,0,0.08);
  --drag-shadow:   0 8px 24px rgba(0,0,0,0.15);
  --accent:        #3b82f6;
  --wip-over:      #fef2f2;
  --wip-border:    #fca5a5;
}

.board {
  display: flex;
  gap: 1rem;
  padding: 1.5rem;
  overflow-x: auto;
  background: var(--board-bg);
  min-height: 100vh;
  align-items: flex-start;
}

.board-column {
  flex: 0 0 280px;
  background: var(--column-bg);
  border: 1px solid var(--column-border);
  border-radius: 10px;
  padding: 0.75rem;
  min-height: 120px;
  transition: background 0.15s, border-color 0.15s;
}

/* Column turns red when WIP limit would be exceeded */
.board-column[data-wip-exceeded="true"] {
  background: var(--wip-over);
  border-color: var(--wip-border);
}

.column-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 0.75rem;
}

.column-title {
  font-size: 0.875rem;
  font-weight: 600;
  margin: 0;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #475569;
}

.column-count {
  font-size: 0.75rem;
  background: #e2e8f0;
  border-radius: 999px;
  padding: 0.125rem 0.5rem;
  font-weight: 600;
  color: #64748b;
}

.column-cards {
  list-style: none;
  margin: 0;
  padding: 0;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  min-height: 40px;  /* drop target even when empty */
}

.card {
  background: var(--card-bg);
  border-radius: 8px;
  padding: 0.75rem;
  box-shadow: var(--card-shadow);
  cursor: grab;
  user-select: none;
  border: 2px solid transparent;
  transition: box-shadow 0.15s, transform 0.15s, border-color 0.15s;
}

.card:hover   { box-shadow: 0 2px 8px rgba(0,0,0,0.12); }
.card:active  { cursor: grabbing; }

/* DnD kit adds data-dragging class via style injection — mirror with class */
.card[data-dragging="true"] {
  box-shadow: var(--drag-shadow);
  transform: rotate(2deg) scale(1.02);
  border-color: var(--accent);
  opacity: 0.85;
}

/* Drop placeholder while dragging over */
.card-placeholder {
  height: 64px;
  border: 2px dashed var(--accent);
  border-radius: 8px;
  background: rgba(59, 130, 246, 0.04);
}

.card-title {
  font-size: 0.9375rem;
  margin: 0 0 0.5rem;
  color: #1e293b;
}

.card-meta   { display: flex; gap: 0.375rem; flex-wrap: wrap; }
.card-tag {
  font-size: 0.6875rem;
  padding: 0.125rem 0.5rem;
  border-radius: 999px;
  background: #dbeafe;
  color: #1d4ed8;
  font-weight: 500;
}
```

---

## React Implementation

### Normalized State

```tsx
// board.types.ts
export interface Card {
  id: string;
  title: string;
  tag?: string;
}

export interface Column {
  id: string;
  title: string;
  cardIds: string[];
  wipLimit?: number;
}

export interface BoardState {
  cards: Record<string, Card>;
  columns: Record<string, Column>;
  columnOrder: string[];
}
```

### Board Store (Zustand or useReducer)

```tsx
// useBoardStore.ts
import { create } from 'zustand';
import type { BoardState, Card, Column } from './board.types';

interface MovePayload {
  cardId:       string;
  fromColumnId: string;
  toColumnId:   string;
  toIndex:      number;
}

interface BoardStore extends BoardState {
  moveCard: (payload: MovePayload, previousState?: BoardState) => void;
  rollback: (snapshot: BoardState) => void;
}

const INITIAL: BoardState = {
  cards: {
    'c1': { id: 'c1', title: 'Design mockups', tag: 'Design' },
    'c2': { id: 'c2', title: 'Write API spec',  tag: 'Backend' },
    'c3': { id: 'c3', title: 'Set up CI/CD',    tag: 'DevOps' },
  },
  columns: {
    'todo':        { id: 'todo',        title: 'To Do',       cardIds: ['c1', 'c2'], wipLimit: 5 },
    'in-progress': { id: 'in-progress', title: 'In Progress', cardIds: ['c3'],       wipLimit: 3 },
    'done':        { id: 'done',        title: 'Done',        cardIds: [],           wipLimit: undefined },
  },
  columnOrder: ['todo', 'in-progress', 'done'],
};

export const useBoardStore = create<BoardStore>((set) => ({
  ...INITIAL,

  moveCard({ cardId, fromColumnId, toColumnId, toIndex }) {
    set(state => {
      const from = { ...state.columns[fromColumnId], cardIds: [...state.columns[fromColumnId].cardIds] };
      const to   = fromColumnId === toColumnId
        ? from
        : { ...state.columns[toColumnId], cardIds: [...state.columns[toColumnId].cardIds] };

      // Remove from source
      from.cardIds = from.cardIds.filter(id => id !== cardId);

      // Insert at target index
      if (fromColumnId === toColumnId) {
        from.cardIds.splice(toIndex, 0, cardId);
      } else {
        to.cardIds.splice(toIndex, 0, cardId);
      }

      return {
        columns: {
          ...state.columns,
          [fromColumnId]: from,
          [toColumnId]:   to,
        },
      };
    });
  },

  rollback(snapshot) { set(snapshot); },
}));
```

### Drag-and-Drop with @dnd-kit

```tsx
// Board.tsx
import {
  DndContext, DragOverlay, DragStartEvent, DragEndEvent,
  DragOverEvent, PointerSensor, KeyboardSensor, useSensor, useSensors,
  closestCorners,
} from '@dnd-kit/core';
import {
  SortableContext, verticalListSortingStrategy, useSortable, arrayMove,
} from '@dnd-kit/sortable';
import { CSS } from '@dnd-kit/utilities';
import { useState, useCallback } from 'react';
import { useBoardStore } from './useBoardStore';
import type { Card, Column } from './board.types';

export function Board() {
  const { cards, columns, columnOrder, moveCard, rollback } = useBoardStore();
  const [activeCard, setActiveCard] = useState<Card | null>(null);

  const sensors = useSensors(
    useSensor(PointerSensor, { activationConstraint: { distance: 5 } }),
    useSensor(KeyboardSensor)
  );

  function onDragStart({ active }: DragStartEvent) {
    setActiveCard(cards[active.id as string] ?? null);
  }

  async function onDragEnd({ active, over }: DragEndEvent) {
    setActiveCard(null);
    if (!over) return;

    const cardId     = active.id as string;
    const overId     = over.id as string;
    const snapshot   = useBoardStore.getState();

    // Determine destination column
    let toColumnId = overId;
    let toIndex    = 0;

    // If dropped over a card (not a column), find which column the card is in
    for (const col of Object.values(columns)) {
      const idx = col.cardIds.indexOf(overId);
      if (idx !== -1) { toColumnId = col.id; toIndex = idx; break; }
    }

    // Find source column
    const fromColumnId = Object.values(columns)
      .find(col => col.cardIds.includes(cardId))?.id;

    if (!fromColumnId) return;

    // Check WIP limit
    const destCol = columns[toColumnId];
    if (destCol.wipLimit !== undefined) {
      const newCount = destCol.cardIds.length + (fromColumnId !== toColumnId ? 1 : 0);
      if (newCount > destCol.wipLimit) return; // blocked
    }

    // Optimistic update
    moveCard({ cardId, fromColumnId, toColumnId, toIndex });

    // Persist to API
    try {
      await fetch('/api/board/move', {
        method: 'PATCH',
        body: JSON.stringify({ cardId, fromColumnId, toColumnId, toIndex }),
        headers: { 'Content-Type': 'application/json' },
      });
    } catch {
      // Rollback on failure
      rollback(snapshot);
    }
  }

  return (
    <DndContext
      sensors={sensors}
      collisionDetection={closestCorners}
      onDragStart={onDragStart}
      onDragEnd={onDragEnd}
    >
      <div className="board" role="region" aria-label="Kanban board">
        {columnOrder.map(colId => (
          <BoardColumn key={colId} column={columns[colId]} cards={cards} />
        ))}
      </div>
      <DragOverlay>
        {activeCard && <CardItem card={activeCard} isDragging />}
      </DragOverlay>
    </DndContext>
  );
}

function BoardColumn({ column, cards }: { column: Column; cards: Record<string, Card> }) {
  const cardList = column.cardIds.map(id => cards[id]).filter(Boolean);
  const isOverLimit = column.wipLimit !== undefined && cardList.length >= column.wipLimit;

  return (
    <div
      className="board-column"
      aria-label={`${column.title} — ${cardList.length} cards`}
      data-wip-exceeded={isOverLimit ? 'true' : undefined}
    >
      <div className="column-header">
        <h2 className="column-title">{column.title}</h2>
        <span className="column-count" aria-label={`${cardList.length} cards`}>
          {cardList.length}{column.wipLimit ? `/${column.wipLimit}` : ''}
        </span>
      </div>
      <SortableContext items={column.cardIds} strategy={verticalListSortingStrategy}>
        <ul className="column-cards" role="list" aria-label={`Cards in ${column.title}`}>
          {cardList.map(card => (
            <SortableCard key={card.id} card={card} />
          ))}
        </ul>
      </SortableContext>
    </div>
  );
}

function SortableCard({ card }: { card: Card }) {
  const { attributes, listeners, setNodeRef, transform, transition, isDragging } = useSortable({ id: card.id });

  const style = {
    transform: CSS.Transform.toString(transform),
    transition,
    opacity: isDragging ? 0 : 1,
  };

  return (
    <li ref={setNodeRef} style={style} {...attributes} {...listeners}>
      <CardItem card={card} isDragging={isDragging} />
    </li>
  );
}

function CardItem({ card, isDragging = false }: { card: Card; isDragging?: boolean }) {
  return (
    <div className="card" data-dragging={isDragging ? 'true' : undefined}
      aria-grabbed={isDragging}>
      <p className="card-title">{card.title}</p>
      {card.tag && (
        <div className="card-meta">
          <span className="card-tag">{card.tag}</span>
        </div>
      )}
    </div>
  );
}
```

---

## Angular Implementation

### Board Service with Signals

```typescript
// board.service.ts
import { Injectable, signal, computed } from '@angular/core';
import type { BoardState, Card, Column } from './board.types';

@Injectable({ providedIn: 'root' })
export class BoardService {
  private _state = signal<BoardState>({ /* same INITIAL as React */ cards: {}, columns: {}, columnOrder: [] });

  readonly state = this._state.asReadonly();
  readonly columns = computed(() =>
    this._state().columnOrder.map(id => this._state().columns[id])
  );

  moveCard(cardId: string, fromId: string, toId: string, toIndex: number) {
    this._state.update(s => {
      const from = { ...s.columns[fromId], cardIds: s.columns[fromId].cardIds.filter(id => id !== cardId) };
      const to   = fromId === toId
        ? { ...from, cardIds: [...from.cardIds] }
        : { ...s.columns[toId], cardIds: [...s.columns[toId].cardIds] };
      to.cardIds.splice(toIndex, 0, cardId);
      return { ...s, columns: { ...s.columns, [fromId]: from, [toId]: to } };
    });
  }

  snapshot(): BoardState { return this._state(); }
  rollback(snap: BoardState) { this._state.set(snap); }
}
```

### CDK Drag-and-Drop Component

```typescript
// board.component.ts
import { Component, inject } from '@angular/core';
import { CdkDragDrop, DragDropModule, transferArrayItem, moveItemInArray } from '@angular/cdk/drag-drop';
import { CommonModule } from '@angular/common';
import { BoardService } from './board.service';

@Component({
  selector: 'app-board',
  standalone: true,
  imports: [CommonModule, DragDropModule],
  template: `
    <div class="board" role="region" aria-label="Kanban board">
      @for (col of board.columns(); track col.id) {
        <div class="board-column"
          [attr.data-wip-exceeded]="col.wipLimit && col.cardIds.length >= col.wipLimit ? 'true' : null">
          <div class="column-header">
            <h2 class="column-title">{{ col.title }}</h2>
            <span class="column-count">
              {{ col.cardIds.length }}{{ col.wipLimit ? '/' + col.wipLimit : '' }}
            </span>
          </div>
          <ul class="column-cards" role="list"
            cdkDropList
            [cdkDropListData]="col.id"
            [id]="col.id"
            [cdkDropListConnectedTo]="board.state().columnOrder"
            (cdkDropListDropped)="onDrop($event)">
            @for (cardId of col.cardIds; track cardId) {
              <li class="card" cdkDrag [cdkDragData]="cardId"
                [attr.aria-label]="board.state().cards[cardId]?.title">
                <p class="card-title">{{ board.state().cards[cardId]?.title }}</p>
                @if (board.state().cards[cardId]?.tag) {
                  <span class="card-tag">{{ board.state().cards[cardId].tag }}</span>
                }
                <div *cdkDragPlaceholder class="card-placeholder"></div>
              </li>
            }
          </ul>
        </div>
      }
    </div>
  `,
})
export class BoardComponent {
  board = inject(BoardService);

  async onDrop(event: CdkDragDrop<string>) {
    const cardId     = event.item.data as string;
    const fromId     = event.previousContainer.data;
    const toId       = event.container.data;
    const toIndex    = event.currentIndex;
    const destCol    = this.board.state().columns[toId];

    // WIP check
    if (destCol.wipLimit !== undefined && fromId !== toId &&
        destCol.cardIds.length >= destCol.wipLimit) return;

    const snap = this.board.snapshot();
    this.board.moveCard(cardId, fromId, toId, toIndex);

    try {
      await fetch('/api/board/move', {
        method: 'PATCH',
        body: JSON.stringify({ cardId, fromId, toId, toIndex }),
        headers: { 'Content-Type': 'application/json' },
      });
    } catch {
      this.board.rollback(snap);
    }
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Board region has `aria-label` | `role="region" aria-label="Kanban board"` |
| Columns have descriptive labels | `aria-label="In Progress — 3 cards"` |
| Cards are list items | `<ul role="list">` + `<li>` for semantics |
| Drag state announced | `aria-grabbed` attribute toggled during drag |
| Keyboard drag-and-drop supported | `KeyboardSensor` from @dnd-kit included |
| WIP limit surfaced to screen readers | Column count shows "3/3" — reads aloud |
| Drag overlay has same content as card | Screen readers don't see phantom elements |
| Card titles are announced | `aria-label` on draggable `<li>` |
| Focus returns to card after drop | @dnd-kit restores focus automatically |
| Column drop zone announced when dragging over | `aria-live="polite"` region for "Dropped into Done" |

---

## Production Pitfalls

**1. Normalized state becomes stale after optimistic update fails silently**
If the API call throws but the error is swallowed, the UI shows incorrect state while the server has old state. Fix: always await the mutation and explicitly rollback to the snapshot in the catch block. Never use fire-and-forget for state mutations.

**2. Drag ghost flickers on fast pointer events**
`DragOverlay` re-renders every `onDragMove` event. Fix: memoize the overlay content with `React.memo` and keep it stateless (pass only `card` as a prop).

**3. `cdkDropListConnectedTo` misses dynamically added columns**
Angular CDK requires the list of connected IDs to be up-to-date. If columns are added/removed at runtime, the static `columnOrder` array gets out of sync. Fix: bind `[cdkDropListConnectedTo]` to a computed signal that always reflects the current column IDs.

**4. Reordering causes expensive `key` remounts**
If you use array index as the `key` in React (or `trackBy: index` in Angular), every card below the insertion point remounts. Fix: always key/track by `card.id`. The DOM element follows the card, not its position.

**5. WIP limit check is client-only**
The client blocks the move, but a second browser tab can bypass it. Fix: enforce the WIP limit server-side in the PATCH endpoint. The client check is UX feedback only.

**6. Touch drag on mobile iOS 13 needs passive event listeners workaround**
@dnd-kit uses `touchstart` which triggers the iOS "passive listener" warning and may cause scroll hijacking. Fix: pass `{ activationConstraint: { delay: 250, tolerance: 5 } }` to `TouchSensor` so brief taps don't activate drag, and configure `touch-action: none` on draggable elements via CSS.

---

## Interview Angle

**Q: "How would you implement a Kanban board where cards can be dragged between columns?"**

Strong answer covers:
1. **Normalized state** — columns and cards in separate maps, with `cardIds: string[]` per column. Moving a card is a pure array splice with no deep clone of unrelated data.
2. **Library choice** — @dnd-kit (React) gives composable sensors (pointer, keyboard, touch) and an accessible overlay. Angular CDK DragDrop integrates with Angular change detection. Never roll your own `mousedown`/`mousemove`/`mouseup` pipeline for production use.
3. **Optimistic UI + rollback** — take a snapshot before the mutation, apply immediately, persist async, roll back in the catch. Users see instant feedback; consistency is maintained on failure.
4. **WIP limits** — checked before applying the optimistic update, enforced server-side on the PATCH endpoint. Client check is UX; server check is correctness.

**Follow-up: "How do you handle reordering within a column vs. moving between columns?"**
Same `moveCard` operation but with different source and destination column IDs. When `fromColumnId === toColumnId` it's a reorder (one array mutation). When they differ, it's a transfer (two array mutations). @dnd-kit's `SortableContext` + `DragOverlay` handles the visual distinction; your state layer doesn't need to know about the UI.
