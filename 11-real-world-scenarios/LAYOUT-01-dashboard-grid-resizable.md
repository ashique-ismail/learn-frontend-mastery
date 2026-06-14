# Dashboard Grid with Resizable Panels

## The Idea

**In plain English:** You have a dashboard made of panels — charts, tables, stats cards — arranged in a grid. Users can drag the dividers between panels to give more space to the data they care about most. The layout survives a page refresh because it is saved to `localStorage`. You need to decide whether to build the resize behavior yourself or reach for `react-grid-layout`.

**Real-world analogy:** Think of a newsroom's physical corkboard, where editors pin stories.

- The **corkboard itself** is the CSS Grid container — it defines the columns and rows.
- Each **story card pinned on the board** is a grid panel: it occupies a named area (`header`, `sidebar`, `main`, `footer`).
- The **pushpin** at the border between two cards is the drag handle — dragging it redistributes how much corkboard space each card occupies.
- The **editor's memory of where cards were left last night** is `localStorage` — the arrangement persists so the board looks the same the next morning.
- A **board management tool** (like a Trello-style system) = `react-grid-layout`: it handles all the dragging, snapping, and serialization for you, at the cost of owning the layout entirely.

The key insight: CSS Grid `template-areas` gives panels semantic names and handles visual layout. Resize behavior is pure JavaScript state management — CSS cannot track pointer deltas or update column sizes dynamically.

---

## Learning Objectives

- Build a CSS Grid layout using named template areas
- Implement a drag-to-resize handle with `pointermove` and pointer capture
- Model grid column and row sizes as numeric state arrays
- Persist layout state to `localStorage` and restore it on mount
- Understand the trade-off between custom implementation and `react-grid-layout`
- Apply ARIA attributes so resize handles are operable by keyboard

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Define named grid areas | ✅ `grid-template-areas` | — |
| Set initial column/row sizes | ✅ `grid-template-columns/rows` | — |
| Update column size on pointer drag | ❌ | CSS has no pointer event handling |
| Track cumulative delta across a drag gesture | ❌ | Requires JS state to accumulate pixel movement |
| Cap column size to a min/max range | ⚠️ `minmax()` in track definition | Only for initial layout; cannot enforce at runtime as JS resizes |
| Save and restore layout across sessions | ❌ | `localStorage` is JS-only |
| Announce resize progress to screen readers | ❌ | `aria-valuenow` toggling requires JS |
| Snap to a grid increment (e.g. 50px) | ❌ | Requires JS rounding logic |

**Conclusion:** CSS owns the initial grid structure and visual presentation. Every aspect of dynamic resizing — delta tracking, state updates, persistence, ARIA value management — belongs to JavaScript.

---

## HTML & CSS Foundation

### Grid Structure with Named Areas

```html
<div class="dashboard" style="--col-a: 280px; --col-b: 1fr;">

  <header class="panel panel--header">Header / Toolbar</header>

  <aside class="panel panel--sidebar">Sidebar Panel</aside>

  <!-- Vertical resize handle between sidebar and main -->
  <div
    class="resize-handle resize-handle--col"
    role="separator"
    aria-orientation="vertical"
    aria-label="Resize sidebar"
    aria-valuenow="280"
    aria-valuemin="160"
    aria-valuemax="600"
    tabindex="0"
  ></div>

  <main class="panel panel--main">Main Content Panel</main>

  <footer class="panel panel--footer">Footer</footer>

</div>
```

### The CSS

```css
/* ─── CSS custom properties hold live grid sizes ─── */
:root {
  --col-a: 280px;      /* sidebar width — updated by JS */
  --col-b: 1fr;        /* main column — fills remaining space */
  --row-header: 56px;
  --row-footer: 48px;
  --row-main: 1fr;
  --handle-size: 6px;
  --handle-color: transparent;
  --handle-hover-color: #4f8ef7;
  --panel-bg: #ffffff;
  --panel-border: #e2e8f0;
  --transition-snap: 0.1s ease;
}

/* ─── Grid container ─── */
.dashboard {
  display: grid;
  grid-template-columns: var(--col-a) var(--handle-size) var(--col-b);
  grid-template-rows:
    var(--row-header)
    var(--row-main)
    var(--row-footer);
  grid-template-areas:
    "header  header  header"
    "sidebar handle  main  "
    "footer  footer  footer";
  height: 100vh;
  height: 100dvh;
  overflow: hidden;
  background: #f1f5f9;
  gap: 0;
}

/* ─── Panels ─── */
.panel {
  background: var(--panel-bg);
  border: 1px solid var(--panel-border);
  overflow: auto;
  min-width: 0;     /* prevent blowout in grid */
  min-height: 0;
}

.panel--header  { grid-area: header;  border-bottom: 1px solid var(--panel-border); }
.panel--sidebar { grid-area: sidebar; border-right: none; overflow-y: auto; }
.panel--main    { grid-area: main; }
.panel--footer  { grid-area: footer;  border-top: 1px solid var(--panel-border); }

/* ─── Resize handle ─── */
.resize-handle {
  background: var(--handle-color);
  cursor: col-resize;
  transition: background var(--transition-snap);
  position: relative;
  z-index: 10;
  /* Wider invisible hit area via pseudo-element */
}

.resize-handle--col {
  grid-area: handle;
  cursor: col-resize;
  border-left: 1px solid var(--panel-border);
  border-right: 1px solid var(--panel-border);
}

.resize-handle--row {
  cursor: row-resize;
}

/* Expand hit area without affecting layout */
.resize-handle::before {
  content: '';
  position: absolute;
  inset: 0 -6px;    /* 6px extra on each side */
}

.resize-handle:hover,
.resize-handle:focus-visible,
.resize-handle.is-dragging {
  background: var(--handle-hover-color);
  outline: none;
}

/* ─── Dragging cursor on the whole page prevents cursor flicker ─── */
body.resizing-col { cursor: col-resize !important; user-select: none; }
body.resizing-row { cursor: row-resize !important; user-select: none; }

/* ─── Reduced motion: disable snap animation ─── */
@media (prefers-reduced-motion: reduce) {
  .resize-handle { transition: none; }
}
```

**What CSS owns:** grid template structure, named areas, initial column sizes via custom properties, handle appearance, cursor changes, reduced-motion fallback.

**What CSS cannot own:** reading pointer coordinates, computing pixel deltas, updating `--col-a` on the container, clamping values, writing to `localStorage`.

---

## React Implementation

### Types and Storage

```tsx
// types.ts
export interface GridLayout {
  colA: number;    // sidebar width in px
  rowMain: number; // main row height in px — if you also need vertical resize
}

// storage.ts
const STORAGE_KEY = 'dashboard-layout';

export function loadLayout(): GridLayout | null {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    return raw ? (JSON.parse(raw) as GridLayout) : null;
  } catch {
    return null;
  }
}

export function saveLayout(layout: GridLayout): void {
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(layout));
  } catch {
    // Quota exceeded or private browsing — fail silently
  }
}
```

### The Resize Hook

```tsx
// useResizeHandle.ts
import { useRef, useCallback, useEffect } from 'react';

interface Options {
  axis: 'x' | 'y';
  initialSize: number;
  min: number;
  max: number;
  onResize: (newSize: number) => void;
  onCommit?: (finalSize: number) => void; // called on pointerup — ideal for persistence
  step?: number; // keyboard increment in px
}

export function useResizeHandle(options: Options) {
  const { axis, initialSize, min, max, onResize, onCommit, step = 16 } = options;

  const sizeRef       = useRef(initialSize);
  const isDraggingRef = useRef(false);
  const startPosRef   = useRef(0);
  const startSizeRef  = useRef(0);

  const handleRef = useRef<HTMLDivElement>(null);

  const clamp = (v: number) => Math.min(max, Math.max(min, v));

  const onPointerDown = useCallback((e: React.PointerEvent<HTMLDivElement>) => {
    e.preventDefault();
    isDraggingRef.current = true;
    startPosRef.current  = axis === 'x' ? e.clientX : e.clientY;
    startSizeRef.current = sizeRef.current;

    // Pointer capture: keeps receiving events even if pointer leaves the element
    handleRef.current?.setPointerCapture(e.pointerId);
    handleRef.current?.classList.add('is-dragging');
    document.body.classList.add(axis === 'x' ? 'resizing-col' : 'resizing-row');
  }, [axis]);

  const onPointerMove = useCallback((e: React.PointerEvent<HTMLDivElement>) => {
    if (!isDraggingRef.current) return;

    const delta    = (axis === 'x' ? e.clientX : e.clientY) - startPosRef.current;
    const newSize  = clamp(startSizeRef.current + delta);
    sizeRef.current = newSize;
    onResize(newSize);
  }, [axis, onResize, min, max]);

  const onPointerUp = useCallback((e: React.PointerEvent<HTMLDivElement>) => {
    if (!isDraggingRef.current) return;
    isDraggingRef.current = false;

    handleRef.current?.releasePointerCapture(e.pointerId);
    handleRef.current?.classList.remove('is-dragging');
    document.body.classList.remove('resizing-col', 'resizing-row');
    onCommit?.(sizeRef.current);
  }, [onCommit]);

  // Keyboard support: arrow keys adjust size by `step`
  const onKeyDown = useCallback((e: React.KeyboardEvent<HTMLDivElement>) => {
    const isIncrease = axis === 'x'
      ? e.key === 'ArrowRight'
      : e.key === 'ArrowDown';
    const isDecrease = axis === 'x'
      ? e.key === 'ArrowLeft'
      : e.key === 'ArrowUp';

    if (!isIncrease && !isDecrease) return;
    e.preventDefault();

    const delta   = (isIncrease ? 1 : -1) * step;
    const newSize = clamp(sizeRef.current + delta);
    sizeRef.current = newSize;
    onResize(newSize);
    onCommit?.(newSize);
  }, [axis, step, onResize, onCommit, min, max]);

  return { handleRef, onPointerDown, onPointerMove, onPointerUp, onKeyDown };
}
```

### Dashboard Component

```tsx
// Dashboard.tsx
import { useState, useCallback, useRef, useLayoutEffect } from 'react';
import { useResizeHandle } from './useResizeHandle';
import { loadLayout, saveLayout } from './storage';
import type { GridLayout } from './types';

const DEFAULT_LAYOUT: GridLayout = { colA: 280, rowMain: 0 };
const MIN_COL = 160;
const MAX_COL = 600;

export function Dashboard() {
  const containerRef = useRef<HTMLDivElement>(null);

  const [layout, setLayout] = useState<GridLayout>(() => {
    return loadLayout() ?? DEFAULT_LAYOUT;
  });

  // Apply CSS custom properties directly on the container whenever layout changes
  // useLayoutEffect avoids a flash of the default value
  useLayoutEffect(() => {
    const el = containerRef.current;
    if (!el) return;
    el.style.setProperty('--col-a', `${layout.colA}px`);
  }, [layout.colA]);

  const updateColA = useCallback((newSize: number) => {
    setLayout(prev => ({ ...prev, colA: newSize }));
  }, []);

  const commitColA = useCallback((finalSize: number) => {
    setLayout(prev => {
      const next = { ...prev, colA: finalSize };
      saveLayout(next);
      return next;
    });
  }, []);

  const { handleRef, onPointerDown, onPointerMove, onPointerUp, onKeyDown } =
    useResizeHandle({
      axis: 'x',
      initialSize: layout.colA,
      min: MIN_COL,
      max: MAX_COL,
      onResize: updateColA,
      onCommit: commitColA,
    });

  return (
    <div
      ref={containerRef}
      className="dashboard"
      style={{ '--col-a': `${layout.colA}px` } as React.CSSProperties}
    >
      <header className="panel panel--header">
        <h1>Dashboard</h1>
      </header>

      <aside className="panel panel--sidebar" aria-label="Sidebar">
        <p>Sidebar content</p>
      </aside>

      {/* Resize handle between sidebar and main */}
      <div
        ref={handleRef}
        className="resize-handle resize-handle--col"
        role="separator"
        aria-orientation="vertical"
        aria-label="Resize sidebar"
        aria-valuenow={Math.round(layout.colA)}
        aria-valuemin={MIN_COL}
        aria-valuemax={MAX_COL}
        tabIndex={0}
        onPointerDown={onPointerDown}
        onPointerMove={onPointerMove}
        onPointerUp={onPointerUp}
        onKeyDown={onKeyDown}
      />

      <main className="panel panel--main" aria-label="Main content">
        <p>Main panel content</p>
      </main>

      <footer className="panel panel--footer">Footer</footer>
    </div>
  );
}
```

### react-grid-layout Overview vs Custom Implementation

```tsx
// With react-grid-layout — drastically less code, but you surrender control
import GridLayout from 'react-grid-layout';
import 'react-grid-layout/css/styles.css';
import 'react-resizable/css/styles.css';

const layout = [
  { i: 'sidebar', x: 0, y: 0, w: 3, h: 10 },
  { i: 'main',    x: 3, y: 0, w: 9, h: 10 },
];

export function RGLDashboard() {
  return (
    <GridLayout
      className="layout"
      layout={layout}
      cols={12}
      rowHeight={30}
      width={1200}
      onLayoutChange={(newLayout) => {
        localStorage.setItem('rgl-layout', JSON.stringify(newLayout));
      }}
      draggableHandle=".drag-handle"
      resizeHandles={['se', 'sw']}
    >
      <div key="sidebar"><span className="drag-handle">Sidebar</span></div>
      <div key="main"><span className="drag-handle">Main</span></div>
    </GridLayout>
  );
}

/*
 * Trade-off summary:
 *
 * react-grid-layout:
 *   + Drag, resize, snap-to-grid, serialize out of the box
 *   + Active community (80k+ weekly npm downloads)
 *   - 12-column grid model — awkward for non-uniform layouts
 *   - Adds ~80 kB to the bundle (gzipped ~25 kB)
 *   - Opinionated CSS: overrides are messy
 *   - Poor accessibility by default (no keyboard resize, no ARIA roles)
 *   - Column widths are grid-unit fractions, not arbitrary px values
 *
 * Custom implementation:
 *   + Full control over visual design and CSS Grid areas
 *   + Keyboard resize with proper ARIA (role="separator", aria-valuenow)
 *   + localStorage schema is your own — easy versioning/migration
 *   + Zero additional bundle cost
 *   - You own the pointer math and edge cases (touch, multi-handle, RTL)
 *
 * Rule of thumb: use react-grid-layout when panels are user-rearrangeable
 * free tiles (like a Jira board). Use a custom handle when panels have
 * fixed positions and users only adjust the proportions between them.
 */
```

---

## Angular Implementation

### Layout Service

```typescript
// grid-layout.service.ts
import { Injectable, signal, effect } from '@angular/core';

export interface GridLayout {
  colA: number;
}

const STORAGE_KEY = 'dashboard-layout';
const DEFAULTS: GridLayout = { colA: 280 };

@Injectable({ providedIn: 'root' })
export class GridLayoutService {
  private _layout = signal<GridLayout>(this.#load());

  readonly layout = this._layout.asReadonly();

  constructor() {
    // Persist every time layout changes — throttle in prod for performance
    effect(() => {
      const current = this._layout();
      try {
        localStorage.setItem(STORAGE_KEY, JSON.stringify(current));
      } catch { /* quota exceeded */ }
    });
  }

  setColA(px: number): void {
    this._layout.update(prev => ({ ...prev, colA: Math.max(160, Math.min(600, px)) }));
  }

  #load(): GridLayout {
    try {
      const raw = localStorage.getItem(STORAGE_KEY);
      return raw ? (JSON.parse(raw) as GridLayout) : DEFAULTS;
    } catch {
      return DEFAULTS;
    }
  }
}
```

### Resize Handle Directive

```typescript
// resize-handle.directive.ts
import {
  Directive, ElementRef, HostListener, Input, Output,
  EventEmitter, OnDestroy, inject
} from '@angular/core';

@Directive({
  selector: '[appResizeHandle]',
  standalone: true,
  host: {
    role: 'separator',
    tabindex: '0',
    '[attr.aria-valuenow]': 'currentSize',
    '[attr.aria-valuemin]': 'min',
    '[attr.aria-valuemax]': 'max',
  }
})
export class ResizeHandleDirective implements OnDestroy {
  @Input() axis: 'x' | 'y' = 'x';
  @Input() min = 160;
  @Input() max = 600;
  @Input() step = 16;
  @Input() currentSize = 280;

  @Output() sizeChange = new EventEmitter<number>();
  @Output() sizeCommit = new EventEmitter<number>();

  private el = inject(ElementRef<HTMLElement>);
  private isDragging = false;
  private startPos   = 0;
  private startSize  = 0;

  private get cursClass() {
    return this.axis === 'x' ? 'resizing-col' : 'resizing-row';
  }

  @HostListener('pointerdown', ['$event'])
  onPointerDown(e: PointerEvent): void {
    e.preventDefault();
    this.isDragging  = true;
    this.startPos    = this.axis === 'x' ? e.clientX : e.clientY;
    this.startSize   = this.currentSize;
    this.el.nativeElement.setPointerCapture(e.pointerId);
    this.el.nativeElement.classList.add('is-dragging');
    document.body.classList.add(this.cursClass);
  }

  @HostListener('pointermove', ['$event'])
  onPointerMove(e: PointerEvent): void {
    if (!this.isDragging) return;
    const pos      = this.axis === 'x' ? e.clientX : e.clientY;
    const delta    = pos - this.startPos;
    const newSize  = this.clamp(this.startSize + delta);
    this.sizeChange.emit(newSize);
  }

  @HostListener('pointerup', ['$event'])
  onPointerUp(e: PointerEvent): void {
    if (!this.isDragging) return;
    this.isDragging = false;
    this.el.nativeElement.releasePointerCapture(e.pointerId);
    this.el.nativeElement.classList.remove('is-dragging');
    document.body.classList.remove(this.cursClass);
    this.sizeCommit.emit(this.currentSize);
  }

  @HostListener('keydown', ['$event'])
  onKeyDown(e: KeyboardEvent): void {
    const inc = this.axis === 'x' ? 'ArrowRight' : 'ArrowDown';
    const dec = this.axis === 'x' ? 'ArrowLeft'  : 'ArrowUp';
    if (e.key !== inc && e.key !== dec) return;
    e.preventDefault();
    const delta   = e.key === inc ? this.step : -this.step;
    const newSize = this.clamp(this.currentSize + delta);
    this.sizeChange.emit(newSize);
    this.sizeCommit.emit(newSize);
  }

  ngOnDestroy(): void {
    document.body.classList.remove('resizing-col', 'resizing-row');
  }

  private clamp(v: number): number {
    return Math.min(this.max, Math.max(this.min, v));
  }
}
```

### Dashboard Component

```typescript
// dashboard.component.ts
import { Component, computed, inject, ElementRef, effect } from '@angular/core';
import { GridLayoutService } from './grid-layout.service';
import { ResizeHandleDirective } from './resize-handle.directive';

@Component({
  selector: 'app-dashboard',
  standalone: true,
  imports: [ResizeHandleDirective],
  template: `
    <div class="dashboard" #container>

      <header class="panel panel--header">
        <h1>Dashboard</h1>
      </header>

      <aside class="panel panel--sidebar" aria-label="Sidebar">
        Sidebar content
      </aside>

      <div
        appResizeHandle
        class="resize-handle resize-handle--col"
        axis="x"
        [min]="160"
        [max]="600"
        [step]="16"
        [currentSize]="layout().colA"
        aria-orientation="vertical"
        aria-label="Resize sidebar"
        (sizeChange)="onSizeChange($event)"
        (sizeCommit)="onSizeCommit($event)"
      ></div>

      <main class="panel panel--main" aria-label="Main content">
        Main panel content
      </main>

      <footer class="panel panel--footer">Footer</footer>
    </div>
  `,
})
export class DashboardComponent {
  private service = inject(GridLayoutService);
  private elRef   = inject(ElementRef<HTMLElement>);

  readonly layout = this.service.layout;

  constructor() {
    // Sync CSS custom property to the container whenever layout signal changes
    effect(() => {
      const el = this.elRef.nativeElement.querySelector('.dashboard') as HTMLElement;
      if (el) el.style.setProperty('--col-a', `${this.layout().colA}px`);
    });
  }

  onSizeChange(px: number): void {
    this.service.setColA(px);
  }

  onSizeCommit(_px: number): void {
    // GridLayoutService effect() already persists on every signal change.
    // Hook here for any additional commit-only side effects (e.g. analytics).
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Handle has `role="separator"` | Communicates to AT that this element divides two regions |
| `aria-orientation="vertical"` on column handle | Screen readers announce "vertical splitter" |
| `aria-valuenow` reflects current size in px | Updated in sync with every resize event |
| `aria-valuemin` and `aria-valuemax` set bounds | Screen readers can announce position as a percentage of range |
| `tabindex="0"` makes handle keyboard focusable | Tab key reaches the handle |
| Arrow keys resize by `step` px | ArrowRight/Left for columns, ArrowUp/Down for rows |
| Focus indicator visible on handle | `focus-visible` CSS rule adds color highlight |
| Panels have `aria-label` | Users know what they are resizing between |
| `user-select: none` on body during drag | Prevents accidental text selection being announced |
| Layout restoration does not cause focus loss | `localStorage` read happens before first render (lazy `useState` initializer) |

---

## Production Pitfalls

**1. Pointer capture is essential — do not use `mousemove` on `document`**
Without `element.setPointerCapture(e.pointerId)`, moving the pointer faster than the browser renders will leave the element behind. The pointermove events fire on the captured element regardless of where the pointer physically is, so the resize stays locked until pointerup.

**2. CSS custom properties update synchronously — `style.setProperty` in `useLayoutEffect`, not `useEffect`**
If you update `--col-a` inside `useEffect`, the browser will have already painted one frame with the old value, causing a visible flicker. `useLayoutEffect` (React) or `afterNextRender` / synchronous `effect()` (Angular) run before the paint.

**3. `localStorage` reads in the render initializer can throw in private browsing**
`localStorage.getItem` throws a `DOMException` in some browsers under private mode or when storage is full. Always wrap in try/catch and fall back to defaults — never let a storage error crash the dashboard.

**4. Persisting on every `pointermove` event hammers `localStorage`**
`localStorage.setItem` is synchronous and blocks the main thread. Persist only on `pointerup` (the commit event), not on every intermediate size change. In Angular, the service `effect()` fires on every signal write — debounce it or split `setColA` (live update, no persist) from `commitColA` (persist).

**5. Min/max are not enforced by CSS `minmax()` after JS resizes**
`grid-template-columns: minmax(160px, 600px) ...` applies only to the initial track sizing algorithm. Once JS writes `--col-a: 100px` into the inline style, the grid renders it at 100px regardless. Clamp in JS explicitly; do not rely on CSS to enforce runtime bounds.

**6. RTL layouts reverse drag direction**
In a right-to-left document, the sidebar is on the right and dragging the handle left increases its size. Check `document.dir === 'rtl'` and negate the delta. `react-grid-layout` does not handle RTL correctly out of the box either — this is one case where a custom handle is more reliable.

**7. Multi-handle layouts require shared size state**
If you have both a vertical and a horizontal handle, their sizes are independent but both affect the grid definition on the same container. Model them together in a single layout object (e.g. `{ colA, rowA }`) and write both properties atomically. Splitting them into separate `useState` calls risks a render where one value has updated and the other has not.

---

## Interview Angle

**Q: "Walk me through how you would implement a drag-to-resize panel divider in a React dashboard."**

Strong answer covers five points:
1. **CSS Grid with custom properties** — define the initial layout using `grid-template-areas` and store the sidebar width as a CSS custom property (`--col-a`). This keeps the layout declarative and allows the JS to update a single value rather than recalculating multiple properties.
2. **Pointer capture** — `onPointerDown` records the start position and calls `setPointerCapture` on the handle. `onPointerMove` computes `delta = currentX - startX` and adds it to the recorded start size. Without pointer capture, fast drags lose tracking.
3. **State and CSS in sync** — the numeric size lives in React state (for ARIA and rendering consistency), but the actual visual position is driven by writing to `el.style.setProperty('--col-a', newSize + 'px')` inside `useLayoutEffect` to avoid a paint-before-update flicker.
4. **Persistence** — write to `localStorage` only on `pointerup` (the commit event), not on every move. Read from `localStorage` in the lazy `useState` initializer so the correct value is available before the first render.
5. **Keyboard and ARIA** — the handle is `role="separator"` with `aria-valuenow`, `aria-valuemin`, `aria-valuemax`, and `tabIndex=0`. Arrow keys adjust the size by a fixed step. This makes the feature operable without a pointer device.

**Follow-up: "When would you choose `react-grid-layout` over a custom implementation?"**

Use `react-grid-layout` when panels must be freely draggable and rearrangeable by the user (the position of each panel is part of the state, not just its size). Its 12-column snapping model, serializable layout format, and out-of-the-box drag handles save substantial engineering time for that use case. Choose a custom handle when the grid structure is fixed — panels are always in the same areas — and users only adjust proportions. Custom handles give you full CSS Grid control, proper ARIA semantics, RTL support, and zero bundle overhead from the library.
