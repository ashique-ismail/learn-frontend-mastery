# Swipe-to-Delete List Item

## The Idea

**In plain English:** Each row in a list can be swiped horizontally to the left, revealing a red "Delete" button underneath. If the user swipes far enough and fast enough, the item snaps into the revealed state and waits for confirmation. If the swipe is weak or released mid-way, the row snaps back to its original position. Once the user taps "Delete", the row animates out and an undo snackbar appears at the bottom of the screen with a countdown timer. Tapping "Undo" before the timer expires restores the item.

**Real-world analogy:** Think of a physical folder in a filing cabinet.

- **Pulling the folder tab out a little and letting go** = a weak swipe; the folder slides back into place.
- **Pulling the folder all the way out** = a committed swipe; the delete button is now fully visible and waiting.
- **Physically removing the folder from the drawer** = confirming the delete; the item leaves the list.
- **The "Recently Deleted" tray on the cabinet** = the undo snackbar; you have a short window to drop the folder back before it's gone for good.
- **The speed at which you pulled** = velocity threshold; a fast decisive yank at the end of a partial swipe still counts as a committed swipe even if the distance was modest.

The key insight: the gesture lives on a spectrum — distance and velocity together determine intent. CSS handles the visual reveal; JS tracks the physics.

---

## Learning Objectives

- Capture `touchstart` / `touchmove` / `touchend` events and compute horizontal displacement and velocity
- Gate pointer events with `touch-action: pan-y` so vertical scrolling still works
- Use `translateX` to reveal an underlying action button without moving the item out of the document flow
- Apply snap-back and snap-forward animations with CSS transitions turned on/off conditionally
- Implement pointer events (`pointerdown` / `pointermove` / `pointerup`) as a cross-device alternative
- Build an undo snackbar with a timeout, clearable ref, and graceful removal
- Understand why you cannot rely on CSS alone for gesture-driven interactions

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Slide the row visually during drag | ❌ | `translateX` must update in real time to match finger position — CSS cannot read touch coordinates |
| Detect horizontal vs. vertical intent | ❌ | Requires comparing `deltaX` vs. `deltaY` on `touchmove` |
| Compute swipe velocity | ❌ | Velocity = distance ÷ time; CSS has no access to either |
| Snap forward vs. snap back based on velocity | ❌ | Decision requires JS logic combining distance and velocity |
| Disable CSS transition during drag, re-enable on release | ❌ | Toggling `transition` must respond to pointer state |
| Remove item from the DOM after delete animation | ❌ | Requires state mutation after `transitionend` |
| Show an undo snackbar with a countdown timer | ❌ | Requires `setTimeout` and dynamic DOM insertion |
| Restore the item on undo | ❌ | Requires state reconciliation |
| Support mouse drag on desktop (pointer events) | ❌ | Touch events do not fire for mouse; requires unified pointer event API |

**Conclusion:** CSS owns the transition curves, the delete button reveal layer, and the snap animations. JS owns all gesture tracking, state decisions, and the undo lifecycle.

---

## HTML & CSS Foundation

### The Structure

```html
<!--
  The item wrapper holds two layers:
    1. .swipe-action  — the delete button, revealed behind the row
    2. .swipe-row     — the visible row content, translated by JS
-->
<ul class="swipe-list" aria-label="Task list">
  <li class="swipe-item">
    <!-- Revealed layer: sits behind the row -->
    <div class="swipe-action" aria-hidden="true">
      <button class="delete-btn" tabindex="-1">Delete</button>
    </div>

    <!-- The draggable row -->
    <div
      class="swipe-row"
      role="listitem"
      aria-label="Buy groceries"
      touch-action="pan-y"
    >
      <span class="row-content">Buy groceries</span>

      <!-- Keyboard-accessible delete button (not touch-dependent) -->
      <button
        class="row-delete-btn"
        aria-label="Delete Buy groceries"
      >
        <svg aria-hidden="true" width="16" height="16" viewBox="0 0 16 16">
          <path d="M2 4h12M5 4V2h6v2M6 7v5M10 7v5M3 4l1 9h8l1-9H3z"
                stroke="currentColor" stroke-width="1.5" fill="none" stroke-linecap="round"/>
        </svg>
      </button>
    </div>
  </li>
</ul>

<!-- Undo snackbar (injected by JS, lives outside the list) -->
<div class="snackbar" role="status" aria-live="polite" aria-atomic="true" hidden>
  <span class="snackbar-message">Item deleted</span>
  <button class="snackbar-undo">Undo</button>
</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --row-height: 60px;
  --delete-width: 80px;
  --snap-transition: 0.25s cubic-bezier(0.4, 0, 0.2, 1);
  --collapse-transition: 0.3s ease-out;
  --delete-bg: #d32f2f;
  --delete-color: #ffffff;
  --snackbar-bg: #323232;
  --snackbar-color: #ffffff;
}

/* ─── List container ─── */
.swipe-list {
  list-style: none;
  margin: 0;
  padding: 0;
  overflow: hidden;         /* clamp children; no horizontal scroll on the list */
}

/* ─── Item wrapper ─── */
/*
  position: relative creates the stacking context.
  overflow: hidden clips the action layer so it's invisible until revealed.
*/
.swipe-item {
  position: relative;
  height: var(--row-height);
  overflow: hidden;
  background: #fff;
  border-bottom: 1px solid #e0e0e0;
}

/* Item exit animation — triggered by JS adding .is-deleting */
.swipe-item.is-deleting {
  height: 0;
  border-bottom-width: 0;
  transition: height var(--collapse-transition),
              border-bottom-width var(--collapse-transition);
}

/* ─── Action layer (delete button behind the row) ─── */
.swipe-action {
  position: absolute;
  top: 0;
  right: 0;
  width: var(--delete-width);
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--delete-bg);
}

.delete-btn {
  color: var(--delete-color);
  background: none;
  border: none;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  width: 100%;
  height: 100%;
}

/* ─── Row (the draggable surface) ─── */
.swipe-row {
  position: relative;       /* sits above .swipe-action due to stacking order */
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 100%;
  padding: 0 1rem;
  background: #fff;
  /* transition is OFF during drag; JS adds .is-snapping to re-enable it */
  touch-action: pan-y;      /* allow vertical scroll, suppress horizontal browser handling */
  user-select: none;        /* prevent text selection mid-swipe */
  will-change: transform;
}

.swipe-row.is-snapping {
  transition: transform var(--snap-transition);
}

/* When row is fully swiped open (JS adds .is-open) */
.swipe-row.is-open {
  transform: translateX(calc(-1 * var(--delete-width)));
}

/* ─── Inline keyboard-accessible delete button ─── */
.row-delete-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 44px;
  height: 44px;
  border: none;
  background: none;
  cursor: pointer;
  color: #757575;
  border-radius: 50%;
  transition: background 0.15s, color 0.15s;
}

.row-delete-btn:hover,
.row-delete-btn:focus-visible {
  background: #ffebee;
  color: var(--delete-bg);
  outline: 2px solid var(--delete-bg);
  outline-offset: 2px;
}

/* ─── Snackbar ─── */
.snackbar {
  position: fixed;
  bottom: 1rem;
  left: 50%;
  transform: translateX(-50%) translateY(0);
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 0.75rem 1.25rem;
  background: var(--snackbar-bg);
  color: var(--snackbar-color);
  border-radius: 4px;
  font-size: 0.875rem;
  box-shadow: 0 3px 8px rgba(0,0,0,0.3);
  z-index: 1000;
  animation: snackbar-in 0.2s ease-out;
}

.snackbar[hidden] {
  display: none;
}

.snackbar-undo {
  background: none;
  border: none;
  color: #ffb74d;
  font-weight: 700;
  cursor: pointer;
  font-size: 0.875rem;
  padding: 0;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

@keyframes snackbar-in {
  from { transform: translateX(-50%) translateY(1rem); opacity: 0; }
  to   { transform: translateX(-50%) translateY(0);    opacity: 1; }
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .swipe-row.is-snapping,
  .swipe-item.is-deleting,
  .snackbar {
    transition: none;
    animation: none;
  }
}
```

**What CSS owns:** the delete button reveal layer, row slide position via `translateX`, snap transition curves, item collapse animation, snackbar appearance, and reduced-motion fallback.

**What CSS cannot own:** reading touch coordinates, computing velocity, deciding snap direction, toggling the transition class at the right moment, and orchestrating the undo timeout.

---

## React Implementation

### Types

```tsx
// types.ts
export interface ListItem {
  id: string;
  label: string;
}

export interface SwipeState {
  x: number;           // current translateX in pixels (always <= 0)
  dragging: boolean;
  committed: boolean;  // true = snapped to reveal state
}
```

### The Swipe Hook

```tsx
// useSwipeToDelete.ts
import { useRef, useCallback, useState } from 'react';

const DELETE_WIDTH     = 80;   // matches --delete-width
const VELOCITY_THRESH  = 0.4;  // px per ms — fast flick still triggers even if distance is short
const DISTANCE_THRESH  = DELETE_WIDTH * 0.5; // must swipe > 50% of delete zone to commit

interface UseSwipeOptions {
  onDelete: () => void;
}

export function useSwipeToDelete({ onDelete }: UseSwipeOptions) {
  const [committed, setCommitted] = useState(false);

  // Refs hold in-flight gesture data — no re-renders during drag
  const startXRef    = useRef(0);
  const startYRef    = useRef(0);
  const startTimeRef = useRef(0);
  const currentXRef  = useRef(0);
  const rowRef       = useRef<HTMLDivElement>(null);
  const lockAxisRef  = useRef<'h' | 'v' | null>(null);

  // Apply transform directly to DOM — bypasses React render cycle for smoothness
  const setTranslate = useCallback((x: number, withTransition: boolean) => {
    const el = rowRef.current;
    if (!el) return;
    el.style.transition = withTransition ? '' : 'none';  // '' = let CSS class decide
    el.style.transform  = `translateX(${Math.min(0, x)}px)`;
  }, []);

  const snapTo = useCallback((x: number) => {
    const el = rowRef.current;
    if (!el) return;
    el.classList.add('is-snapping');
    el.style.transform = `translateX(${x}px)`;
    if (x === 0) {
      el.classList.remove('is-open');
    } else {
      el.classList.add('is-open');
    }
  }, []);

  // ── Pointer Events (mouse + touch via unified API) ──
  const onPointerDown = useCallback((e: React.PointerEvent<HTMLDivElement>) => {
    if (committed) return;
    const el = rowRef.current;
    if (!el) return;

    el.setPointerCapture(e.pointerId);   // keep receiving events even if pointer leaves element

    startXRef.current    = e.clientX;
    startYRef.current    = e.clientY;
    startTimeRef.current = e.timeStamp;
    currentXRef.current  = 0;
    lockAxisRef.current  = null;

    el.classList.remove('is-snapping');  // disable transition during live drag
  }, [committed]);

  const onPointerMove = useCallback((e: React.PointerEvent<HTMLDivElement>) => {
    if (!rowRef.current?.hasPointerCapture(e.pointerId)) return;

    const dx = e.clientX - startXRef.current;
    const dy = e.clientY - startYRef.current;

    // Determine axis lock on first significant move
    if (!lockAxisRef.current) {
      if (Math.abs(dx) > 4 || Math.abs(dy) > 4) {
        lockAxisRef.current = Math.abs(dx) >= Math.abs(dy) ? 'h' : 'v';
      }
      return; // not enough movement yet to decide
    }

    if (lockAxisRef.current === 'v') return; // vertical scroll — ignore

    // Horizontal swipe only; clamp to [−DELETE_WIDTH, 0]
    e.preventDefault();
    const clamped = Math.max(-DELETE_WIDTH, Math.min(0, dx));
    currentXRef.current = clamped;
    setTranslate(clamped, false);
  }, [setTranslate]);

  const onPointerUp = useCallback((e: React.PointerEvent<HTMLDivElement>) => {
    const el = rowRef.current;
    if (!el?.hasPointerCapture(e.pointerId)) return;
    el.releasePointerCapture(e.pointerId);

    if (lockAxisRef.current !== 'h') return; // was a vertical scroll, ignore

    const dx       = currentXRef.current;   // negative number
    const elapsed  = e.timeStamp - startTimeRef.current;
    const velocity = Math.abs(dx) / elapsed; // px/ms

    const shouldCommit =
      Math.abs(dx) >= DISTANCE_THRESH || velocity >= VELOCITY_THRESH;

    if (shouldCommit) {
      setCommitted(true);
      snapTo(-DELETE_WIDTH);
    } else {
      snapTo(0);
    }
  }, [snapTo]);

  const onPointerCancel = useCallback(() => {
    snapTo(0);
  }, [snapTo]);

  // Called when the revealed Delete button is tapped
  const handleDeleteConfirm = useCallback(() => {
    onDelete();
  }, [onDelete]);

  // Called from the snap-back button (e.g., clicking elsewhere to dismiss)
  const handleClose = useCallback(() => {
    setCommitted(false);
    snapTo(0);
  }, [snapTo]);

  return {
    rowRef,
    committed,
    handlers: { onPointerDown, onPointerMove, onPointerUp, onPointerCancel },
    handleDeleteConfirm,
    handleClose,
  };
}
```

### Undo Snackbar Hook

```tsx
// useUndoSnackbar.ts
import { useRef, useState, useCallback } from 'react';

const UNDO_TIMEOUT_MS = 5000;

interface PendingDeletion<T> {
  item: T;
  index: number;
}

export function useUndoSnackbar<T>() {
  const [items, setItems]                       = useState<T[]>([]);
  const [pending, setPending]                   = useState<PendingDeletion<T> | null>(null);
  const timerRef                                = useRef<ReturnType<typeof setTimeout> | null>(null);

  const clearTimer = useCallback(() => {
    if (timerRef.current !== null) {
      clearTimeout(timerRef.current);
      timerRef.current = null;
    }
  }, []);

  const scheduleDelete = useCallback((item: T, index: number) => {
    setPending({ item, index });

    clearTimer();
    timerRef.current = setTimeout(() => {
      // Timer expired — commit the deletion by simply clearing pending
      setPending(null);
    }, UNDO_TIMEOUT_MS);
  }, [clearTimer]);

  const deleteItem = useCallback((id: string, getItemId: (item: T) => string) => {
    setItems(prev => {
      const index = prev.findIndex(i => getItemId(i) === id);
      if (index === -1) return prev;
      const item = prev[index];
      scheduleDelete(item, index);
      return prev.filter((_, i) => i !== index);
    });
  }, [scheduleDelete]);

  const undoDelete = useCallback(() => {
    clearTimer();
    if (!pending) return;
    // Re-insert at original index
    setItems(prev => {
      const next = [...prev];
      next.splice(pending.index, 0, pending.item);
      return next;
    });
    setPending(null);
  }, [pending, clearTimer]);

  return { items, setItems, pending, deleteItem, undoDelete };
}
```

### Full Component

```tsx
// SwipeList.tsx
import { useCallback }         from 'react';
import { useSwipeToDelete }    from './useSwipeToDelete';
import { useUndoSnackbar }     from './useUndoSnackbar';
import type { ListItem }       from './types';

interface SwipeRowProps {
  item: ListItem;
  onDelete: () => void;
}

function SwipeRow({ item, onDelete }: SwipeRowProps) {
  const { rowRef, committed, handlers, handleDeleteConfirm, handleClose } =
    useSwipeToDelete({ onDelete });

  return (
    <li className="swipe-item">
      {/* Revealed action layer */}
      <div className="swipe-action" aria-hidden="true">
        <button
          className="delete-btn"
          tabIndex={committed ? 0 : -1}  // only reachable by keyboard when revealed
          onClick={handleDeleteConfirm}
        >
          Delete
        </button>
      </div>

      {/* Draggable row */}
      <div
        ref={rowRef}
        className="swipe-row"
        aria-label={item.label}
        {...handlers}
        onClick={committed ? handleClose : undefined}
      >
        <span className="row-content">{item.label}</span>

        {/* Always-accessible keyboard delete button */}
        <button
          className="row-delete-btn"
          aria-label={`Delete ${item.label}`}
          onClick={(e) => {
            e.stopPropagation();
            onDelete();
          }}
        >
          <svg aria-hidden="true" width="16" height="16" viewBox="0 0 16 16">
            <path
              d="M2 4h12M5 4V2h6v2M6 7v5M10 7v5M3 4l1 9h8l1-9H3z"
              stroke="currentColor" strokeWidth="1.5" fill="none" strokeLinecap="round"
            />
          </svg>
        </button>
      </div>
    </li>
  );
}

export function SwipeList({ initialItems }: { initialItems: ListItem[] }) {
  const { items, setItems, pending, deleteItem, undoDelete } =
    useUndoSnackbar<ListItem>();

  // Initialise on mount
  const [initialised, setInitialised] = (
    // minimal one-time init via lazy useState
    useState as typeof useState
  )(() => {
    setItems(initialItems);
    return true;
  });

  const handleDelete = useCallback((id: string) => {
    deleteItem(id, item => item.id);
  }, [deleteItem]);

  return (
    <>
      <ul className="swipe-list" aria-label="Task list">
        {items.map(item => (
          <SwipeRow
            key={item.id}
            item={item}
            onDelete={() => handleDelete(item.id)}
          />
        ))}
      </ul>

      {/* Undo Snackbar */}
      {pending && (
        <div className="snackbar" role="status" aria-live="polite" aria-atomic="true">
          <span className="snackbar-message">"{pending.item.label}" deleted</span>
          <button className="snackbar-undo" onClick={undoDelete}>Undo</button>
        </div>
      )}
    </>
  );
}
```

---

## Angular Implementation

### Service

```typescript
// swipe-list.service.ts
import { Injectable, signal, computed } from '@angular/core';

export interface ListItem {
  id: string;
  label: string;
}

interface PendingDeletion {
  item: ListItem;
  index: number;
}

const UNDO_TIMEOUT_MS = 5000;

@Injectable({ providedIn: 'root' })
export class SwipeListService {
  private _items   = signal<ListItem[]>([]);
  private _pending = signal<PendingDeletion | null>(null);
  private _timer: ReturnType<typeof setTimeout> | null = null;

  readonly items        = this._items.asReadonly();
  readonly pending      = this._pending.asReadonly();
  readonly hasPending   = computed(() => this._pending() !== null);

  init(items: ListItem[]) {
    this._items.set(items);
  }

  delete(id: string) {
    const current = this._items();
    const index   = current.findIndex(i => i.id === id);
    if (index === -1) return;

    const item = current[index];
    this._items.set(current.filter((_, i) => i !== index));
    this._pending.set({ item, index });

    this.clearTimer();
    this._timer = setTimeout(() => this._pending.set(null), UNDO_TIMEOUT_MS);
  }

  undo() {
    this.clearTimer();
    const p = this._pending();
    if (!p) return;
    this._items.update(prev => {
      const next = [...prev];
      next.splice(p.index, 0, p.item);
      return next;
    });
    this._pending.set(null);
  }

  private clearTimer() {
    if (this._timer !== null) {
      clearTimeout(this._timer);
      this._timer = null;
    }
  }
}
```

### Swipe Row Directive

```typescript
// swipe-row.directive.ts
import {
  Directive, ElementRef, Output, EventEmitter,
  OnDestroy, inject, input
} from '@angular/core';

const DELETE_WIDTH    = 80;
const VELOCITY_THRESH = 0.4;
const DISTANCE_THRESH = DELETE_WIDTH * 0.5;

@Directive({
  selector: '[appSwipeRow]',
  standalone: true,
  host: {
    '(pointerdown)':   'onPointerDown($event)',
    '(pointermove)':   'onPointerMove($event)',
    '(pointerup)':     'onPointerUp($event)',
    '(pointercancel)': 'onPointerCancel()',
  },
})
export class SwipeRowDirective implements OnDestroy {
  @Output() committed   = new EventEmitter<void>();
  @Output() snappedBack = new EventEmitter<void>();

  private el          = inject(ElementRef<HTMLElement>);
  private startX      = 0;
  private startY      = 0;
  private startTime   = 0;
  private currentX    = 0;
  private axisLock: 'h' | 'v' | null = null;
  private isCommitted = false;

  private get host(): HTMLElement { return this.el.nativeElement; }

  onPointerDown(e: PointerEvent) {
    if (this.isCommitted) return;
    this.host.setPointerCapture(e.pointerId);
    this.startX    = e.clientX;
    this.startY    = e.clientY;
    this.startTime = e.timeStamp;
    this.currentX  = 0;
    this.axisLock  = null;
    this.host.classList.remove('is-snapping');
  }

  onPointerMove(e: PointerEvent) {
    if (!this.host.hasPointerCapture(e.pointerId)) return;

    const dx = e.clientX - this.startX;
    const dy = e.clientY - this.startY;

    if (!this.axisLock) {
      if (Math.abs(dx) > 4 || Math.abs(dy) > 4) {
        this.axisLock = Math.abs(dx) >= Math.abs(dy) ? 'h' : 'v';
      }
      return;
    }

    if (this.axisLock === 'v') return;

    e.preventDefault();
    const clamped   = Math.max(-DELETE_WIDTH, Math.min(0, dx));
    this.currentX   = clamped;
    this.host.style.transition = 'none';
    this.host.style.transform  = `translateX(${clamped}px)`;
  }

  onPointerUp(e: PointerEvent) {
    if (!this.host.hasPointerCapture(e.pointerId)) return;
    this.host.releasePointerCapture(e.pointerId);

    if (this.axisLock !== 'h') return;

    const elapsed  = e.timeStamp - this.startTime;
    const velocity = Math.abs(this.currentX) / elapsed;
    const should   = Math.abs(this.currentX) >= DISTANCE_THRESH || velocity >= VELOCITY_THRESH;

    this.host.classList.add('is-snapping');

    if (should) {
      this.isCommitted = true;
      this.host.classList.add('is-open');
      this.host.style.transform = `translateX(${-DELETE_WIDTH}px)`;
      this.committed.emit();
    } else {
      this.snapBack();
    }
  }

  onPointerCancel() {
    this.snapBack();
  }

  snapBack() {
    this.isCommitted = false;
    this.host.classList.add('is-snapping');
    this.host.classList.remove('is-open');
    this.host.style.transform = 'translateX(0)';
    this.snappedBack.emit();
  }

  ngOnDestroy() {
    // Nothing to tear down — pointer capture is auto-released
  }
}
```

### Swipe List Component

```typescript
// swipe-list.component.ts
import { Component, OnInit, inject } from '@angular/core';
import { NgFor, NgIf }               from '@angular/common';
import { SwipeListService }          from './swipe-list.service';
import { SwipeRowDirective }         from './swipe-row.directive';

@Component({
  selector: 'app-swipe-list',
  standalone: true,
  imports: [NgFor, NgIf, SwipeRowDirective],
  template: `
    <ul class="swipe-list" aria-label="Task list">
      <li
        *ngFor="let item of service.items(); trackBy: trackById"
        class="swipe-item"
      >
        <!-- Revealed delete layer -->
        <div class="swipe-action" aria-hidden="true">
          <button
            class="delete-btn"
            [attr.tabindex]="committedId === item.id ? 0 : -1"
            (click)="confirmDelete(item.id)"
          >
            Delete
          </button>
        </div>

        <!-- Draggable row -->
        <div
          class="swipe-row"
          appSwipeRow
          [attr.aria-label]="item.label"
          (committed)="onCommitted(item.id)"
          (snappedBack)="onSnappedBack()"
          (click)="committedId === item.id ? onSnappedBack() : null"
        >
          <span class="row-content">{{ item.label }}</span>

          <button
            class="row-delete-btn"
            [attr.aria-label]="'Delete ' + item.label"
            (click)="$event.stopPropagation(); confirmDelete(item.id)"
          >
            <!-- inline SVG trash icon -->
            <svg aria-hidden="true" width="16" height="16" viewBox="0 0 16 16">
              <path d="M2 4h12M5 4V2h6v2M6 7v5M10 7v5M3 4l1 9h8l1-9H3z"
                    stroke="currentColor" stroke-width="1.5" fill="none" stroke-linecap="round"/>
            </svg>
          </button>
        </div>
      </li>
    </ul>

    <!-- Undo Snackbar -->
    <div
      *ngIf="service.hasPending()"
      class="snackbar"
      role="status"
      aria-live="polite"
      aria-atomic="true"
    >
      <span class="snackbar-message">
        "{{ service.pending()?.item?.label }}" deleted
      </span>
      <button class="snackbar-undo" (click)="service.undo()">Undo</button>
    </div>
  `,
})
export class SwipeListComponent implements OnInit {
  service     = inject(SwipeListService);
  committedId: string | null = null;

  ngOnInit() {
    this.service.init([
      { id: '1', label: 'Buy groceries' },
      { id: '2', label: 'Book dentist appointment' },
      { id: '3', label: 'Reply to emails' },
    ]);
  }

  trackById(_: number, item: { id: string }) { return item.id; }

  onCommitted(id: string) { this.committedId = id; }

  onSnappedBack() { this.committedId = null; }

  confirmDelete(id: string) {
    this.committedId = null;
    this.service.delete(id);
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Delete action reachable without touch | Keyboard-accessible trash button on every row with descriptive `aria-label` |
| Hidden delete button not in tab order until revealed | `tabindex="-1"` on `.delete-btn`; switched to `0` only when row is committed |
| Swipe action announced to screen readers | `aria-live="polite"` snackbar announces deletion; no silent removals |
| Undo button is keyboard-operable | Standard `<button>` element in snackbar; receives focus by default in visible state |
| Row label available to assistive tech | `aria-label` on `.swipe-row` mirrors visible text |
| Touch target minimum size | `min-height: 44px` on all interactive buttons; trash icon button is 44x44px |
| Swipe layer is purely presentational | `.swipe-action` uses `aria-hidden="true"`; action confirmed via separate accessible path |
| `role="status"` on snackbar | Polite live region; does not interrupt current screen reader speech |
| `aria-atomic="true"` on snackbar | Entire message re-read on update; prevents partial reads |
| Reduced-motion support | `@media (prefers-reduced-motion: reduce)` removes all transitions and animations |

---

## Production Pitfalls

**1. Transition fires during drag, making the row lag behind the finger**
The CSS transition must be disabled while the pointer is down. Achieve this by removing the `is-snapping` class on `pointerdown` and re-adding it only on `pointerup`. Never toggle it via `style.transition = 'none'` permanently — that breaks the snap-back.

**2. Horizontal swipe triggers page scroll on Android Chrome**
`touch-action: pan-y` on the row element tells the browser to handle vertical scrolling natively and defer horizontal events to your JS. Without it, the browser competes with your `pointermove` handler and the row either refuses to move or causes erratic scroll. This CSS property must be on the element that captures the pointer, not a parent.

**3. Velocity calculation is wrong because `timeStamp` resets on each event**
The correct approach is to store `startTime` on `pointerdown` and compute `elapsed = pointerup.timeStamp - startTime`. Using `Date.now()` is acceptable but less precise. Never compute velocity per `pointermove` event — latency between events is too variable.

**4. Multiple rows can be in the committed (open) state simultaneously**
If a user swipes row A open, then swipes row B, row A stays open visually. Fix: track a single `committedId` at the list level. When a new row commits, programmatically call `snapBack()` on the previously committed row's directive/ref before marking the new one.

**5. Undo timer persists after component unmount**
If the user navigates away while the snackbar timer is running, the `setTimeout` callback fires against a garbage-collected signal or unmounted component. In React, clean up with `useEffect` returning `clearTimeout`. In Angular, call `clearTimer()` inside the service's own destroy hook or when the component calls `ngOnDestroy`.

**6. Deleting while the row is mid-swipe causes a layout flash**
If the undo logic re-inserts a row that is in the middle of its collapse animation, the list jumps. Fix: always complete the collapse (`transitionend`) before removing the DOM node. React: listen for `transitionend` before calling `setItems`. Angular: use `(transitionend)` binding or an `AnimationBuilder` sequence, not immediate state mutation.

**7. Pointer capture is not released on programmatic `snapBack`**
If you call `snapBack()` from outside the directive (e.g., to close a sibling row), the element may still hold pointer capture from a previous gesture. Call `host.releasePointerCapture(...)` if any capture IDs are outstanding. The `hasPointerCapture` guard in `onPointerMove` prevents stale events from driving movement, but releasing capture explicitly is cleaner.

---

## Interview Angle

**Q: "How would you implement swipe-to-delete on a long list of items, making sure vertical scroll still works and the interaction feels native?"**

A strong answer covers four areas: First, the gesture layer — use the Pointer Events API (`pointerdown` / `pointermove` / `pointerup`) instead of touch events, because it handles mouse, stylus, and touch with one code path. Set `touch-action: pan-y` on each row so the browser keeps vertical scroll while yielding horizontal deltas to JavaScript. On `pointerdown`, call `setPointerCapture` so the element keeps receiving `pointermove` even when the finger slides off the element. Second, the decision logic — compute velocity as `Math.abs(deltaX) / elapsed` on `pointerup`, and commit the delete reveal if velocity exceeds 0.4 px/ms OR if horizontal distance exceeds 50% of the delete zone width. This mirrors the "fast flick or deliberate slide" feel of native iOS mail. Third, the animation — disable CSS `transition` during live drag by removing the snapping class on `pointerdown`; re-add it on `pointerup` before snapping to the final position. This prevents the row from lagging. Fourth, the undo pattern — do not delete from state immediately; instead remove the item from the list and store it in a `pending` slot. Start a `setTimeout`. If the user taps Undo, clear the timer and splice the item back at its original index. If the timer expires, discard the pending slot. The live region snackbar announces the deletion to screen readers regardless of input modality.

**Follow-up: "What breaks at scale — say, a list of 500 items?"**

Two problems emerge. First, attaching pointer listeners to 500 individual `div` elements is wasteful. Switch to event delegation: attach a single `pointerdown` listener to the `ul`, use `closest('.swipe-row')` to find the target row, then dynamically track that row through `pointermove` and `pointerup`. Second, if you store swipe state in React per-item state or Angular per-row signals, 500 rows will each re-render when any single item changes. Lift deletion state to a list-level service and ensure each row renders in isolation — in React via `memo`, in Angular via `OnPush` change detection.

**Follow-up: "How do you handle the accessibility story for users who cannot swipe?"**

Never rely on the swipe gesture as the only deletion path. The trash icon button on each row is the keyboard and screen-reader path. It has a descriptive `aria-label` (`"Delete Buy groceries"`), a minimum 44x44px touch target, and fires the same delete-with-undo flow as the swipe. The undo snackbar uses `role="status"` with `aria-live="polite"` and `aria-atomic="true"` so the confirmation is read aloud without interrupting the user. The hidden swipe-reveal delete button uses `tabindex="-1"` until the row is committed, ensuring it never appears in the natural tab order.
