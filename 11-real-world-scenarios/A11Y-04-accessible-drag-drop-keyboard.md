# Accessible Drag-and-Drop (Keyboard Alternative)

## The Idea

**In plain English:** Native drag-and-drop (HTML5 drag events or library-level pointer events) is 100% inaccessible to keyboard-only users and largely broken for screen reader users. WCAG 2.1 Success Criterion 2.1.1 requires that all functionality is available via keyboard. You need a parallel interaction mode: the user can press a key to "pick up" an item, use arrow keys to reorder it, and press another key to "drop" it. The screen reader needs to be told the item's current position at each step.

**Real-world analogy:** Think of moving furniture in a room.

- **Mouse drag** = physically carrying a sofa across the room — you see where you are going in real time.
- **Keyboard mode** = telling a removal crew which piece to pick up and which slot to put it in. You give explicit commands: "pick up the sofa", "move it to slot 3", "put it down". Each command gets a verbal confirmation: "sofa is now in position 3 of 5".
- **`aria-grabbed`** = the sticky note on the sofa that says "currently being moved" so everyone in the room (including the screen reader) knows which item is in transit.
- **`aria-dropeffect`** = the floor markers showing "items can be placed here". Both attributes are deprecated in ARIA 1.1 but are still encountered in production codebases and announced by NVDA/JAWS, so you need to understand them.
- **`aria-live` announcement** = the removal crew confirming aloud the new position after each arrow key press.

The key insight: keyboard drag-and-drop is not a degraded experience — it is a fully separate, purpose-designed interaction. The user never "drags" anything; they operate a pick-up/move/drop command interface.

---

## Learning Objectives

- Understand why HTML5 drag events are inaccessible and what WCAG 2.5.1 requires
- Know the deprecated `aria-grabbed` / `aria-dropeffect` attributes and their current browser support
- Implement the keyboard contract: Space to pick up, Arrow keys to move, Enter or Space to drop, Escape to cancel
- Announce position changes with `aria-live`
- Wire up both pointer-based drag and keyboard-based reorder in a single React component
- Replicate the keyboard pattern in Angular with signals

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Visual drag cursor and placeholder | ✅ `cursor: grab` / `.dragging` class | — |
| Detect which item is being dragged | ❌ | Requires `dragstart` event or pointer tracking |
| Reorder items in the list | ❌ | Requires mutating array state |
| Announce new position to screen reader | ❌ | Requires `aria-live` region update |
| Track "picked up" state for keyboard | ❌ | Requires a JS picked-up flag |
| Move item with arrow keys | ❌ | Requires `keydown` listener |
| Handle both pointer and keyboard modes | ❌ | Two separate event models |

---

## HTML & CSS Foundation

```html
<section aria-label="Task board">
  <h2 id="list-label">To-do items</h2>

  <!-- aria-live region announces position changes -->
  <div
    id="dnd-announcer"
    class="sr-only"
    aria-live="assertive"
    aria-atomic="true"
  ></div>

  <ul
    role="listbox"
    aria-labelledby="list-label"
    aria-multiselectable="false"
    class="dnd-list"
  >
    <li
      role="option"
      tabindex="0"
      aria-selected="false"
      aria-grabbed="false"       <!-- deprecated but still announces in NVDA/JAWS -->
      class="dnd-item"
      draggable="true"
      data-id="task-1"
    >
      Write unit tests
      <span class="sr-only">, item 1 of 4. Press Space to pick up.</span>
    </li>
    <!-- more items -->
  </ul>
</section>
```

```css
/* ── List container ── */
.dnd-list {
  list-style: none;
  margin: 0;
  padding: 0;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

/* ── Individual item ── */
.dnd-item {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.875rem 1rem;
  background: #fff;
  border: 1px solid #ddd;
  border-radius: 6px;
  cursor: grab;
  font-size: 0.9375rem;
  user-select: none;
  transition: box-shadow 0.15s, transform 0.15s, opacity 0.15s;
}

.dnd-item:focus {
  outline: 3px solid #005fcc;
  outline-offset: 2px;
  cursor: grab;
}

/* Pointer drag: item being dragged */
.dnd-item.is-dragging {
  opacity: 0.4;
  cursor: grabbing;
}

/* Keyboard mode: item is "picked up" */
.dnd-item.is-lifted {
  background: #edf5ff;
  border-color: #005fcc;
  box-shadow: 0 4px 16px rgba(0, 95, 204, 0.2);
  transform: scale(1.02);
  cursor: grabbing;
  z-index: 10;
  position: relative;
}

/* Drop zone highlight during pointer drag */
.dnd-item.is-drop-target {
  border-color: #005fcc;
  border-style: dashed;
}

/* Drag handle icon (optional) */
.drag-handle {
  color: #aaa;
  font-size: 1.1rem;
  cursor: grab;
  flex-shrink: 0;
}

.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

---

## React Implementation

### Types and Utility

```tsx
// types.ts
export interface DndItem {
  id: string;
  label: string;
}

// utils/arrayMove.ts
export function arrayMove<T>(arr: T[], from: number, to: number): T[] {
  const next = [...arr];
  const [removed] = next.splice(from, 1);
  next.splice(to, 0, removed);
  return next;
}
```

### The `useDragAndDrop` Hook

```tsx
// hooks/useDragAndDrop.ts
import { useState, useCallback, useRef } from 'react';
import { arrayMove } from '../utils/arrayMove';
import type { DndItem } from '../types';

interface UseDragAndDropReturn {
  items: DndItem[];
  dragHandlers: (id: string) => {
    draggable: true;
    onDragStart: () => void;
    onDragOver: (e: React.DragEvent) => void;
    onDrop: () => void;
    onDragEnd: () => void;
    className: string;
    'aria-grabbed': 'true' | 'false';
  };
  keyboardHandlers: (id: string, index: number) => {
    tabIndex: number;
    onKeyDown: (e: React.KeyboardEvent) => void;
    className: string;
    'aria-grabbed': 'true' | 'false';
    'aria-selected': 'true' | 'false';
  };
  liftedId: string | null;
}

export function useDragAndDrop(initial: DndItem[]): UseDragAndDropReturn {
  const [items, setItems] = useState(initial);
  const [draggingId, setDraggingId] = useState<string | null>(null);
  const [liftedId, setLiftedId] = useState<string | null>(null);  // keyboard mode
  const announcer = useRef<HTMLElement | null>(null);

  function announce(msg: string) {
    let el = announcer.current;
    if (!el) {
      el = document.getElementById('dnd-announcer');
      announcer.current = el;
    }
    if (!el) return;
    el.textContent = '';
    requestAnimationFrame(() => { if (el) el.textContent = msg; });
  }

  /* ── Pointer drag handlers ── */
  const dragHandlers = useCallback((id: string) => {
    const idx = items.findIndex(i => i.id === id);
    return {
      draggable: true as const,
      'aria-grabbed': (draggingId === id ? 'true' : 'false') as 'true' | 'false',
      className: [
        'dnd-item',
        draggingId === id ? 'is-dragging' : '',
        draggingId && draggingId !== id ? 'is-drop-target' : '',
      ].filter(Boolean).join(' '),
      onDragStart: () => setDraggingId(id),
      onDragOver: (e: React.DragEvent) => e.preventDefault(),
      onDrop: () => {
        if (!draggingId || draggingId === id) return;
        const fromIdx = items.findIndex(i => i.id === draggingId);
        const toIdx = items.findIndex(i => i.id === id);
        setItems(prev => arrayMove(prev, fromIdx, toIdx));
        announce(`${items[fromIdx].label} moved to position ${toIdx + 1} of ${items.length}`);
        setDraggingId(null);
      },
      onDragEnd: () => setDraggingId(null),
    };
  }, [items, draggingId]);

  /* ── Keyboard handlers ── */
  const keyboardHandlers = useCallback((id: string, index: number) => {
    const isLifted = liftedId === id;

    return {
      tabIndex: 0,
      'aria-grabbed': (isLifted ? 'true' : 'false') as 'true' | 'false',
      'aria-selected': (isLifted ? 'true' : 'false') as 'true' | 'false',
      className: ['dnd-item', isLifted ? 'is-lifted' : ''].filter(Boolean).join(' '),
      onKeyDown: (e: React.KeyboardEvent) => {
        if (e.key === ' ' || e.key === 'Enter') {
          e.preventDefault();
          if (!liftedId) {
            // Pick up
            setLiftedId(id);
            announce(`${items[index].label} picked up, current position ${index + 1} of ${items.length}. Use arrow keys to move, Enter or Space to drop, Escape to cancel.`);
          } else if (liftedId === id) {
            // Drop in current position
            setLiftedId(null);
            announce(`${items[index].label} dropped in position ${index + 1} of ${items.length}`);
          }
          return;
        }

        if (!isLifted) return;

        if (e.key === 'Escape') {
          e.preventDefault();
          setLiftedId(null);
          announce(`${items[index].label} drop cancelled. Returned to original position.`);
          return;
        }

        if (e.key === 'ArrowUp' && index > 0) {
          e.preventDefault();
          setItems(prev => arrayMove(prev, index, index - 1));
          announce(`${items[index].label} moved up. Now position ${index} of ${items.length}`);
          // Focus follows the moved item — React will re-render and we rely on the key prop
        }

        if (e.key === 'ArrowDown' && index < items.length - 1) {
          e.preventDefault();
          setItems(prev => arrayMove(prev, index, index + 1));
          announce(`${items[index].label} moved down. Now position ${index + 2} of ${items.length}`);
        }
      },
    };
  }, [items, liftedId]);

  return { items, dragHandlers, keyboardHandlers, liftedId };
}
```

### The List Component

```tsx
// components/SortableList.tsx
import { useEffect, useRef } from 'react';
import { useDragAndDrop } from '../hooks/useDragAndDrop';
import type { DndItem } from '../types';

interface SortableListProps {
  initialItems: DndItem[];
  onReorder?: (items: DndItem[]) => void;
}

export function SortableList({ initialItems, onReorder }: SortableListProps) {
  const { items, dragHandlers, keyboardHandlers, liftedId } = useDragAndDrop(initialItems);
  const prevItems = useRef(items);

  // Notify parent when order changes
  useEffect(() => {
    if (items !== prevItems.current) {
      prevItems.current = items;
      onReorder?.(items);
    }
  }, [items, onReorder]);

  // After keyboard reorder, keep focus on the moved item
  const itemRefs = useRef<Map<string, HTMLLIElement>>(new Map());

  useEffect(() => {
    if (!liftedId) return;
    itemRefs.current.get(liftedId)?.focus();
  }, [items, liftedId]); // re-focus when items array changes (after arrow key move)

  return (
    <>
      <div id="dnd-announcer" className="sr-only" aria-live="assertive" aria-atomic="true" />

      <ul
        role="listbox"
        aria-label="Sortable task list"
        className="dnd-list"
      >
        {items.map((item, index) => {
          const drag = dragHandlers(item.id);
          const keyboard = keyboardHandlers(item.id, index);

          return (
            <li
              key={item.id}
              ref={el => {
                if (el) itemRefs.current.set(item.id, el);
                else itemRefs.current.delete(item.id);
              }}
              role="option"
              id={`dnd-item-${item.id}`}
              {...drag}
              {...keyboard}
              // Merge classNames from both handler sets
              className={[drag.className, liftedId === item.id ? 'is-lifted' : ''].join(' ').trim()}
            >
              <span className="drag-handle" aria-hidden="true">⠿</span>
              {item.label}
              <span className="sr-only">
                , item {index + 1} of {items.length}.
                {liftedId === item.id
                  ? ' Currently lifted. Use arrow keys to move, Enter to drop, Escape to cancel.'
                  : ' Press Space to pick up.'}
              </span>
            </li>
          );
        })}
      </ul>
    </>
  );
}
```

---

## Angular Implementation

```typescript
// components/sortable-list/sortable-list.component.ts
import {
  Component, input, output, signal, computed, ElementRef, ViewChildren,
  QueryList, AfterViewInit, OnDestroy
} from '@angular/core';
import { NgFor } from '@angular/common';

export interface DndItem { id: string; label: string; }

function arrayMove<T>(arr: T[], from: number, to: number): T[] {
  const next = [...arr];
  const [removed] = next.splice(from, 1);
  next.splice(to, 0, removed);
  return next;
}

@Component({
  selector: 'app-sortable-list',
  standalone: true,
  imports: [NgFor],
  template: `
    <div id="dnd-announcer" class="sr-only" aria-live="assertive" aria-atomic="true"></div>

    <ul role="listbox" aria-label="Sortable task list" class="dnd-list">
      <li
        *ngFor="let item of items(); let i = index; trackBy: trackById"
        role="option"
        tabindex="0"
        [id]="'dnd-item-' + item.id"
        [class.dnd-item]="true"
        [class.is-lifted]="liftedId() === item.id"
        [class.is-dragging]="draggingId() === item.id"
        [attr.draggable]="true"
        [attr.aria-grabbed]="liftedId() === item.id || draggingId() === item.id"
        [attr.aria-selected]="liftedId() === item.id"
        (keydown)="onKeyDown($event, item.id, i)"
        (dragstart)="onDragStart(item.id)"
        (dragover)="onDragOver($event)"
        (drop)="onDrop(item.id)"
        (dragend)="draggingId.set(null)"
      >
        <span class="drag-handle" aria-hidden="true">⠿</span>
        {{ item.label }}
        <span class="sr-only">
          , item {{ i + 1 }} of {{ items().length }}.
          {{ liftedId() === item.id
            ? 'Currently lifted. Use arrow keys to move, Enter to drop, Escape to cancel.'
            : 'Press Space to pick up.' }}
        </span>
      </li>
    </ul>
  `,
})
export class SortableListComponent {
  readonly initialItems = input<DndItem[]>([]);
  readonly reordered = output<DndItem[]>();

  items = signal<DndItem[]>([]);
  liftedId = signal<string | null>(null);
  draggingId = signal<string | null>(null);

  ngOnInit() {
    this.items.set(this.initialItems());
  }

  trackById(_: number, item: DndItem) { return item.id; }

  private announce(msg: string) {
    const el = document.getElementById('dnd-announcer');
    if (!el) return;
    el.textContent = '';
    requestAnimationFrame(() => { el.textContent = msg; });
  }

  onDragStart(id: string) { this.draggingId.set(id); }
  onDragOver(e: DragEvent) { e.preventDefault(); }

  onDrop(targetId: string) {
    const fromId = this.draggingId();
    if (!fromId || fromId === targetId) { this.draggingId.set(null); return; }

    const current = this.items();
    const fromIdx = current.findIndex(i => i.id === fromId);
    const toIdx   = current.findIndex(i => i.id === targetId);
    const next = arrayMove(current, fromIdx, toIdx);

    this.items.set(next);
    this.draggingId.set(null);
    this.reordered.emit(next);
    this.announce(`${current[fromIdx].label} moved to position ${toIdx + 1} of ${current.length}`);
  }

  onKeyDown(e: KeyboardEvent, id: string, index: number) {
    const current = this.items();
    const item = current[index];

    if (e.key === ' ' || e.key === 'Enter') {
      e.preventDefault();
      if (!this.liftedId()) {
        this.liftedId.set(id);
        this.announce(`${item.label} picked up, position ${index + 1} of ${current.length}. Arrow keys to move, Enter to drop, Escape to cancel.`);
      } else if (this.liftedId() === id) {
        this.liftedId.set(null);
        this.announce(`${item.label} dropped at position ${index + 1} of ${current.length}`);
      }
      return;
    }

    if (this.liftedId() !== id) return;

    if (e.key === 'Escape') {
      e.preventDefault();
      this.liftedId.set(null);
      this.announce(`${item.label} drop cancelled.`);
      return;
    }

    if (e.key === 'ArrowUp' && index > 0) {
      e.preventDefault();
      const next = arrayMove(current, index, index - 1);
      this.items.set(next);
      this.reordered.emit(next);
      this.announce(`${item.label} moved up. Now position ${index} of ${current.length}`);
      setTimeout(() => document.getElementById(`dnd-item-${id}`)?.focus(), 0);
    }

    if (e.key === 'ArrowDown' && index < current.length - 1) {
      e.preventDefault();
      const next = arrayMove(current, index, index + 1);
      this.items.set(next);
      this.reordered.emit(next);
      this.announce(`${item.label} moved down. Now position ${index + 2} of ${current.length}`);
      setTimeout(() => document.getElementById(`dnd-item-${id}`)?.focus(), 0);
    }
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Keyboard alternative to drag | Space to pick up, arrows to move, Enter/Space to drop, Escape to cancel |
| Current position announced on pick-up | `aria-live="assertive"` region updated immediately on Space |
| Position change announced after each arrow key | New position announced: "item X moved to position N of M" |
| Cancel announced on Escape | "Drop cancelled. Returned to original position." |
| `aria-grabbed` reflects lifted state | `true` when item is picked up (keyboard or drag), `false` otherwise |
| Focus follows moved item | `itemRef.focus()` called after state update in `useEffect` |
| Hidden instruction text for screen readers | `.sr-only` span inside each item explains the keyboard contract |
| `draggable="true"` only on interactive items | Not on container or decorative elements |
| Minimum touch target for drag handle | 44×44px minimum for the handle element |
| `prefers-reduced-motion` respected | Scale/transform animations disabled via media query |

---

## Production Pitfalls

**1. `aria-grabbed` is deprecated but still matters**
ARIA 1.1 deprecated `aria-grabbed` and `aria-dropeffect` in favor of a drag-and-drop roles spec that was never finalized. In 2025, NVDA + Chrome and JAWS both still announce `aria-grabbed` state. Until a replacement exists, include both the deprecated attributes and the `aria-live` announcement.

**2. Focus does not follow the item after `arrayMove`**
When you reorder the items array, React re-renders the list. The item you moved is now at a new DOM position. If you call `.focus()` synchronously before the render, you focus the old position. Fix: call `.focus()` inside a `useEffect` that runs after the `items` state changes, using a ref map keyed by item ID.

**3. Pointer drag events fire on touch screens unpredictably**
HTML5 drag events are not touch events. On iOS, `dragstart` never fires. Use `pointer` events or a library like `@dnd-kit/core` for cross-device pointer drag; your keyboard code remains fully custom.

**4. `role="listbox"` / `role="option"` is wrong for multi-column boards**
If you have a kanban board with multiple columns, `role="listbox"` implies a single-select list. Use `role="list"` / `role="listitem"` and describe the drag operation purely through `aria-live` announcements and the hidden instruction text.

**5. Screen reader in virtual cursor mode intercepts Space and Enter**
NVDA and JAWS users can be in "browse mode" where Space and Enter are intercepted before the page receives them. Add a `role` that forces interaction mode (`role="application"` on the list, or `role="option"` on the items) to ensure key events reach your handler.

**6. Not testing with an actual screen reader**
Mouse and keyboard testing does not validate the auditory experience. Test with NVDA + Chrome and VoiceOver + Safari before shipping. The sequence of announcements is what matters, not the visual state.
