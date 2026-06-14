# List Reorder Animation (FLIP)

## The Idea

**In plain English:** You have a list of items — a kanban card, a to-do list, a playlist. The user drags item 3 above item 1. Without animation, all items teleport to their new positions instantly. With FLIP animation, each item slides smoothly from where it was to where it now belongs — the user can track every item and understand what changed.

**Real-world analogy:** Think of a teacher rearranging students in a class photo.

- Without FLIP: the photographer yells "swap" and everyone teleports. You look at the new photo and have no idea what changed.
- With FLIP: the photographer says "everyone swap" and you watch each student walk from their old spot to their new spot. You can follow Alice walking to the front row and Bob moving back — you mentally track the changes.
- The **FLIP technique** is the choreography. The teacher (JS) notes where everyone was standing (First), waits for the new arrangement to be applied (Last), then makes each student walk backward to their old position (Invert), then walks forward to their actual new position (Play).
- The students do not know they are doing FLIP — they just walk. The clever part is the teacher's bookkeeping.

---

## Learning Objectives

- Understand the FLIP technique in full detail (First, Last, Invert, Play)
- Implement FLIP from scratch with vanilla JS and `getBoundingClientRect`
- Use GSAP's `Flip` plugin for production-grade list reorder animations
- Drop in AutoAnimate for zero-configuration list animations
- Build Angular CDK drag-and-drop with animated reorder previews
- Measure DOM positions before and after reorder correctly (avoiding common pitfalls)

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Animate a single item moving within the same container | ⚠️ with `order` changes + `transition` | CSS Grid/Flexbox `order` changes are not animatable |
| Know the old position of an item before DOM reorder | ❌ | CSS has no access to computed layout history |
| Apply an inverted transform to each item | ❌ | Requires JS to compute per-element delta |
| Animate from old DOM position to new DOM position | ❌ | Old position is gone once DOM mutates |
| Animate items that move out of the visible viewport | ❌ | Requires JS to track positions outside the viewport |
| Handle variable-height items correctly | ❌ | CSS transitions on `height` are expensive and imprecise |

**Conclusion:** FLIP is fundamentally a JS technique. CSS only executes the final `transition` once JS has applied the inverted `transform`. The smarts are entirely in the geometry bookkeeping.

---

## HTML & CSS Foundation

### List Structure

```html
<ul class="sortable-list" id="task-list">
  <li class="sortable-item" data-id="1">
    <span class="item-handle" aria-hidden="true">⠿</span>
    Task One
  </li>
  <li class="sortable-item" data-id="2">
    <span class="item-handle" aria-hidden="true">⠿</span>
    Task Two
  </li>
  <li class="sortable-item" data-id="3">
    <span class="item-handle" aria-hidden="true">⠿</span>
    Task Three
  </li>
</ul>
```

```css
.sortable-list {
  list-style: none;
  padding: 0;
  margin: 0;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.sortable-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 16px;
  background: #ffffff;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  cursor: grab;
  user-select: none;
  /* FLIP: will-change tells the browser to composite this layer */
  will-change: transform;
}

.sortable-item.is-dragging {
  opacity: 0.5;
  cursor: grabbing;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.15);
}

/* The FLIP transition — applied by JS after the Invert step */
.sortable-item.is-animating {
  transition: transform 300ms cubic-bezier(0.4, 0, 0.2, 1);
}

.item-handle {
  color: #9ca3af;
  font-size: 1.2rem;
  flex-shrink: 0;
}

@media (prefers-reduced-motion: reduce) {
  .sortable-item.is-animating {
    transition: none;
  }
}
```

---

## React Implementation

### FLIP from Scratch (No Library)

```tsx
// useFLIPList.ts
import { useRef, useCallback } from 'react';

interface FLIPSnapshot {
  [id: string]: DOMRect;
}

export function useFLIPList() {
  const snapshotRef = useRef<FLIPSnapshot>({});
  const containerRef = useRef<HTMLUListElement>(null);

  // FIRST — capture positions before reorder
  const captureSnapshot = useCallback(() => {
    if (!containerRef.current) return;
    const items = containerRef.current.querySelectorAll<HTMLElement>('[data-id]');
    snapshotRef.current = {};
    items.forEach(item => {
      const id = item.dataset.id!;
      snapshotRef.current[id] = item.getBoundingClientRect();
    });
  }, []);

  // PLAY — called after React re-renders with new order
  const animateToNewPositions = useCallback(() => {
    if (!containerRef.current) return;
    const prefersReduced = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
    if (prefersReduced) return;

    const items = containerRef.current.querySelectorAll<HTMLElement>('[data-id]');

    items.forEach(item => {
      const id = item.dataset.id!;
      const first = snapshotRef.current[id];
      if (!first) return;

      // LAST — current (post-reorder) position
      const last = item.getBoundingClientRect();

      // INVERT — compute the delta between old and new positions
      const deltaY = first.top - last.top;
      const deltaX = first.left - last.left;

      if (Math.abs(deltaY) < 1 && Math.abs(deltaX) < 1) return; // no movement

      // Apply inverted transform (makes element appear at old position)
      item.style.transform = `translate(${deltaX}px, ${deltaY}px)`;
      item.style.transition = 'none';

      // Force reflow to register the inverted position
      item.getBoundingClientRect(); // eslint-disable-line @typescript-eslint/no-unused-expressions

      // PLAY — remove the inverted transform; CSS transition handles the animation
      item.classList.add('is-animating');
      item.style.transform = '';

      item.addEventListener('transitionend', () => {
        item.classList.remove('is-animating');
      }, { once: true });
    });
  }, []);

  return { containerRef, captureSnapshot, animateToNewPositions };
}
```

```tsx
// SortableList.tsx
import { useState, useLayoutEffect, useRef } from 'react';
import { useFLIPList } from './useFLIPList';

interface Task {
  id: string;
  title: string;
}

function moveItem<T>(arr: T[], from: number, to: number): T[] {
  const result = [...arr];
  const [item] = result.splice(from, 1);
  result.splice(to, 0, item);
  return result;
}

export function SortableList({ initialTasks }: { initialTasks: Task[] }) {
  const [tasks, setTasks] = useState(initialTasks);
  const { containerRef, captureSnapshot, animateToNewPositions } = useFLIPList();

  const pendingReorder = useRef(false);

  const reorderItems = (fromIndex: number, toIndex: number) => {
    // FIRST: capture before state update triggers re-render
    captureSnapshot();
    pendingReorder.current = true;
    setTasks(prev => moveItem(prev, fromIndex, toIndex));
  };

  // useLayoutEffect fires synchronously after DOM mutation but before paint
  // This is the correct place to run FLIP's "PLAY" step
  useLayoutEffect(() => {
    if (!pendingReorder.current) return;
    pendingReorder.current = false;
    animateToNewPositions();
  });

  return (
    <ul ref={containerRef} className="sortable-list">
      {tasks.map((task, index) => (
        <li
          key={task.id}
          data-id={task.id}
          className="sortable-item"
          draggable
          onDragStart={(e) => {
            e.dataTransfer.setData('text/plain', String(index));
          }}
          onDragOver={(e) => {
            e.preventDefault();
          }}
          onDrop={(e) => {
            e.preventDefault();
            const fromIndex = Number(e.dataTransfer.getData('text/plain'));
            if (fromIndex !== index) {
              reorderItems(fromIndex, index);
            }
          }}
        >
          <span className="item-handle" aria-hidden="true">⠿</span>
          {task.title}
        </li>
      ))}
    </ul>
  );
}
```

### AutoAnimate (Zero-Config Drop-In)

```tsx
// AutoAnimateList.tsx
import { useAutoAnimate } from '@formkit/auto-animate/react';
import { useState } from 'react';

interface Task {
  id: string;
  title: string;
}

export function AutoAnimateList({ initialTasks }: { initialTasks: Task[] }) {
  const [tasks, setTasks] = useState(initialTasks);
  // autoAnimate automatically detects DOM mutations and applies FLIP
  const [listRef] = useAutoAnimate<HTMLUListElement>({
    duration: 300,
    easing: 'ease-in-out',
  });

  const moveUp = (index: number) => {
    if (index === 0) return;
    setTasks(prev => {
      const next = [...prev];
      [next[index - 1], next[index]] = [next[index], next[index - 1]];
      return next;
    });
  };

  return (
    <ul ref={listRef} className="sortable-list">
      {tasks.map((task, index) => (
        <li key={task.id} data-id={task.id} className="sortable-item">
          <button onClick={() => moveUp(index)} aria-label={`Move ${task.title} up`}>
            ↑
          </button>
          {task.title}
        </li>
      ))}
    </ul>
  );
}
```

---

## Angular Implementation

### Angular CDK Drag and Drop with FLIP

```typescript
// task-list.component.ts
import {
  Component, signal, inject, PLATFORM_ID
} from '@angular/core';
import { isPlatformBrowser, NgFor } from '@angular/common';
import {
  CdkDragDrop, DragDropModule, moveItemInArray
} from '@angular/cdk/drag-drop';

interface Task {
  id: string;
  title: string;
}

@Component({
  selector: 'app-task-list',
  standalone: true,
  imports: [NgFor, DragDropModule],
  styles: [`
    .cdk-drag-preview {
      box-shadow: 0 8px 24px rgba(0,0,0,0.15);
      border-radius: 8px;
      opacity: 0.9;
    }
    .cdk-drag-placeholder {
      opacity: 0.3;
      border: 2px dashed #9ca3af;
      background: transparent;
      border-radius: 8px;
    }
    .cdk-drag-animating {
      transition: transform 250ms cubic-bezier(0.4, 0, 0.2, 1);
    }
    .sortable-list.cdk-drop-list-dragging .sortable-item:not(.cdk-drag-placeholder) {
      transition: transform 250ms cubic-bezier(0.4, 0, 0.2, 1);
    }
  `],
  template: `
    <ul
      cdkDropList
      class="sortable-list"
      [cdkDropListData]="tasks()"
      (cdkDropListDropped)="onDrop($event)"
    >
      <li
        *ngFor="let task of tasks(); trackBy: trackById"
        cdkDrag
        [cdkDragData]="task"
        class="sortable-item"
      >
        <span cdkDragHandle class="item-handle" aria-hidden="true">⠿</span>
        {{ task.title }}

        <!-- Custom drag preview: shows a clean floating card -->
        <div *cdkDragPreview class="sortable-item">
          {{ task.title }}
        </div>
      </li>
    </ul>
  `,
})
export class TaskListComponent {
  private platformId = inject(PLATFORM_ID);

  tasks = signal<Task[]>([
    { id: '1', title: 'Task One' },
    { id: '2', title: 'Task Two' },
    { id: '3', title: 'Task Three' },
    { id: '4', title: 'Task Four' },
  ]);

  onDrop(event: CdkDragDrop<Task[]>): void {
    if (event.previousIndex === event.currentIndex) return;

    this.tasks.update(items => {
      const next = [...items];
      moveItemInArray(next, event.previousIndex, event.currentIndex);
      return next;
    });
  }

  trackById(_: number, task: Task): string {
    return task.id;
  }
}
```

### Manual FLIP Service for Angular

```typescript
// flip-animation.service.ts
import { Injectable, PLATFORM_ID, inject } from '@angular/core';
import { isPlatformBrowser } from '@angular/common';

@Injectable({ providedIn: 'root' })
export class FlipAnimationService {
  private platformId = inject(PLATFORM_ID);
  private snapshots = new Map<string, DOMRect>();

  /** Call BEFORE state change. Pass the container element. */
  capture(container: HTMLElement): void {
    if (!isPlatformBrowser(this.platformId)) return;
    this.snapshots.clear();

    container.querySelectorAll<HTMLElement>('[data-id]').forEach(el => {
      this.snapshots.set(el.dataset['id']!, el.getBoundingClientRect());
    });
  }

  /** Call AFTER state change (in ngAfterViewChecked or effect). */
  play(container: HTMLElement, duration = 300): void {
    if (!isPlatformBrowser(this.platformId)) return;
    if (window.matchMedia('(prefers-reduced-motion: reduce)').matches) return;

    container.querySelectorAll<HTMLElement>('[data-id]').forEach(el => {
      const id = el.dataset['id']!;
      const first = this.snapshots.get(id);
      if (!first) return;

      const last = el.getBoundingClientRect();
      const dy = first.top  - last.top;
      const dx = first.left - last.left;

      if (Math.abs(dy) < 1 && Math.abs(dx) < 1) return;

      el.style.transform = `translate(${dx}px, ${dy}px)`;
      el.style.transition = 'none';

      // Force reflow
      void el.offsetHeight;

      el.style.transition = `transform ${duration}ms cubic-bezier(0.4, 0, 0.2, 1)`;
      el.style.transform = '';

      el.addEventListener('transitionend', () => {
        el.style.transition = '';
        el.style.transform = '';
      }, { once: true });
    });
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Drag-and-drop has keyboard alternative | Provide "Move up / Move down" buttons alongside the drag handle |
| `prefers-reduced-motion` disables all animations | CSS `transition: none` on `.is-animating`; JS guard before applying FLIP |
| Dragged item's new position announced | `aria-live="polite"` region updated: "Task One moved to position 3 of 5" |
| Drag handle is keyboard-focusable | `tabindex="0"`, Space/Enter activates drag mode, arrow keys move |
| Visual drag placeholder is clear | Angular CDK `.cdk-drag-placeholder` styles communicate the drop target |
| `will-change: transform` scoped correctly | Apply only during animation; remove after `transitionend` to free GPU layer |
| Animation duration under 400ms | Keep FLIP animations at 250–300ms; faster feels more responsive |

---

## Production Pitfalls

**1. Calling `getBoundingClientRect` inside React's render cycle returns stale values**
`getBoundingClientRect` must be called *before* state update triggers a re-render. Fix: capture the snapshot in the event handler (before `setState`), then run FLIP in `useLayoutEffect` after the render.

**2. `useLayoutEffect` vs `useEffect` for FLIP's PLAY step**
`useEffect` fires after paint, meaning the items have already jumped to their new positions before you can apply the inverted transform — the user sees a flash. Fix: always use `useLayoutEffect` for the PLAY step. It fires synchronously after DOM mutation but before paint.

**3. GSAP FLIP plugin vs AutoAnimate**
GSAP FLIP is powerful but requires the commercial license for revenue-generating products. AutoAnimate is MIT-licensed, zero-config, and good for simple lists. For complex scenarios (nested lists, spring physics, custom easing), implement FLIP manually or use GSAP.

**4. Items with `position: sticky` or inside a `transform` ancestor report wrong rects**
`getBoundingClientRect` returns viewport-relative coordinates, which can be misleading if the container is scrolled or transformed. Fix: subtract the container's own rect from each item's rect to get container-relative positions.

**5. Angular CDK animation conflicts with FLIP**
Angular CDK already applies its own transition via `.cdk-drag-animating`. Adding your own FLIP transform on top creates a conflict. Fix: either use CDK exclusively (leveraging `.cdk-drop-list-dragging .cdk-drag:not(.cdk-drag-placeholder)`) or disable CDK's built-in animation and run FLIP manually in `ngAfterViewChecked`.

**6. Items added or removed during an ongoing animation**
If a new item is added while a FLIP animation is in progress, its `getBoundingClientRect` at "First" step will be `{top: 0, left: 0}` (not yet rendered). Fix: debounce rapid state changes, or check for zero-rect snapshots before applying the inverted transform.

---

## Interview Angle

**Q: "Explain the FLIP animation technique. Why is `useLayoutEffect` (not `useEffect`) required?"**

Strong answer covers four points:
1. **FLIP is a 4-step bookkeeping trick** — First (capture old rects before state change), Last (measure new rects after DOM update), Invert (apply a transform to make the element appear at its old position), Play (CSS transition removes the transform, animating to the natural new position).
2. **`useLayoutEffect` runs synchronously after DOM mutation but before the browser paints** — this is the only window where you can apply the inverted transform without the user seeing the item at its new position first. `useEffect` fires after paint, causing a visual snap.
3. **Forced reflow is mandatory** — between applying `transform: translateY(Xpx)` and starting the transition, you must flush styles by reading a layout property. Without the flush, the browser batches the two style changes and the animation never plays from the inverted position.
4. **When to use a library** — AutoAnimate for low-config needs; GSAP Flip for complex scenarios; Angular CDK for drag-and-drop with accessibility built-in. FLIP from scratch is still worth knowing for interviews.

**Follow-up: "AutoAnimate says it handles FLIP automatically. What does it actually do?"**
AutoAnimate uses a `MutationObserver` to watch the list container. When it detects a child being added, removed, or reordered, it automatically captures the Before positions (via a `beforeMutation` snapshot queued using `queueMicrotask`), then after the mutation runs the standard FLIP calculation and applies CSS transitions to each affected child.
