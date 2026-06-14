# Split Pane Editor (VS Code-style)

## The Idea

**In plain English:** Two panels sit side by side. A thin drag handle lives between them. You grab it, drag left or right, and both panels resize in real time — the left one grows as the right one shrinks, and vice versa. There are hard limits on how small either panel can get, and the last split position is remembered across page reloads.

**Real-world analogy:** Think of a glass sliding door between two rooms in a house.

- The **two rooms** are your panels — perhaps a file tree on the left and a code editor on the right.
- The **door frame** is the outer container — it sets the total width that both rooms must fill together.
- The **sliding door itself** is the drag handle — it physically sits at the boundary, and you grab its edge to move it.
- **Door stops on each side** are your min/max constraints — the door cannot slide so far that one room disappears entirely.
- **The door position when you lock up and leave** is localStorage — the next morning the door is exactly where you left it.
- **Keyboard resize** is like using a dial on the wall to inch the door left or right in precise 10px steps, without touching it physically.

The key insight: flex-basis drives the visual split; the drag handle only mutates that one number. Everything else — clamping, persisting, keyboard control — is logic layered on top of that single numeric value.

---

## Learning Objectives

- Understand why `flex-basis` (not `width`) is the right tool for a resizable split layout
- Implement a drag handle using `pointermove` / `pointerup` events on the document (not the element)
- Apply min/max clamping so neither pane can collapse to zero
- Wire up keyboard resize with `Alt+Arrow` keys and announce the new size to screen readers
- Persist and restore the split ratio using `localStorage` across sessions
- Know the production edge cases: touch input, iframe interference, RTL layouts, and resize observer debouncing

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Two panels filling 100% width together | ✅ flexbox | — |
| Static split (e.g., always 30%/70%) | ✅ `flex-basis` | — |
| User-draggable boundary | ❌ | CSS has no drag-to-resize primitive for flex children |
| Clamp pane size to min/max during drag | ❌ | Requires JS arithmetic on every `pointermove` |
| Persist last position across reloads | ❌ | CSS cannot read or write `localStorage` |
| Keyboard resize (Alt+Arrow keys) | ❌ | Requires `keydown` listener and numeric state |
| Announce current pane size to screen readers | ❌ | `aria-valuenow` must be updated by JS |
| Prevent text selection during drag | ⚠️ | `user-select: none` helps but only if JS adds it at drag start |
| Correct behavior when container resizes (window resize) | ❌ | Requires `ResizeObserver` to re-clamp the stored value |

**Conclusion:** CSS owns the flex container geometry and handle appearance. JS owns the drag arithmetic, clamping, persistence, keyboard control, and ARIA state.

---

## HTML & CSS Foundation

### The Structure

```html
<!-- The outer shell — a flex row -->
<div class="split-container" id="editor-split">

  <!-- Left pane -->
  <div
    class="split-pane split-pane--left"
    id="split-pane-left"
    role="group"
    aria-label="File explorer"
  >
    <!-- content here -->
  </div>

  <!-- The drag handle -->
  <div
    class="split-handle"
    role="separator"
    aria-label="Resize panes"
    aria-orientation="vertical"
    aria-valuemin="15"
    aria-valuemax="80"
    aria-valuenow="30"
    aria-valuetext="30 percent"
    tabindex="0"
  ></div>

  <!-- Right pane -->
  <div
    class="split-pane split-pane--right"
    id="split-pane-right"
    role="group"
    aria-label="Code editor"
  >
    <!-- content here -->
  </div>

</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --handle-width: 5px;
  --handle-hover-width: 5px;
  --handle-color: #2d2d2d;
  --handle-hover-color: #0078d4;   /* VS Code blue */
  --handle-active-color: #005a9e;
  --pane-min: 15%;                 /* neither pane can be smaller than this */
  --pane-max: 80%;                 /* neither pane can be larger than this */
  --transition-duration: 0.12s;
}

/* ─── Container ─── */
.split-container {
  display: flex;
  flex-direction: row;
  width: 100%;
  height: 100%;           /* fills whatever parent gives it */
  overflow: hidden;       /* pane content scrolls; container does not */
  position: relative;
}

/* ─── Panes ─── */
.split-pane {
  /*
    flex-grow: 0 so panes only take what flex-basis gives them.
    flex-shrink: 0 because JS controls sizing — we don't want flex
    to auto-squeeze panes when the container gets narrow.
    flex-basis is set inline by JS (e.g., style="flex-basis: 30%").
  */
  flex: 0 0 auto;
  min-width: 0;           /* prevents flex overflow from inner content */
  overflow: auto;
  position: relative;
}

.split-pane--right {
  /*
    The right pane fills whatever space the left pane does not use.
    Using flex-grow:1 here instead of JS-managed flex-basis means we
    only need to track ONE value (left pane's width) in state.
  */
  flex: 1 1 auto;
  min-width: 0;
  overflow: auto;
}

/* ─── Handle ─── */
.split-handle {
  flex: 0 0 var(--handle-width);
  background: var(--handle-color);
  cursor: col-resize;
  position: relative;
  z-index: 1;
  transition: background var(--transition-duration);
  /* Expand the clickable/draggable area without changing visual width */
  touch-action: none;   /* required for Pointer Events on touch */
}

/* Wider invisible hit area via pseudo-element */
.split-handle::after {
  content: '';
  position: absolute;
  inset: 0 -4px;          /* extend 4px on each side */
}

.split-handle:hover,
.split-handle:focus-visible {
  background: var(--handle-hover-color);
  outline: none;          /* visual feedback comes from color, not outline */
}

.split-handle:active,
.split-handle.is-dragging {
  background: var(--handle-active-color);
}

/* Focus ring via box-shadow so it stays within the element */
.split-handle:focus-visible {
  box-shadow: 0 0 0 2px #fff, 0 0 0 4px var(--handle-hover-color);
}

/* ─── Drag state: disable text selection container-wide ─── */
.split-container.is-dragging {
  user-select: none;
  cursor: col-resize;     /* keep col-resize even when mouse leaves handle */
}

/* ─── Drag state: disable pointer events on iframes inside panes ─── */
.split-container.is-dragging .split-pane iframe {
  pointer-events: none;
}

/* ─── Reduced motion: skip the handle color transition ─── */
@media (prefers-reduced-motion: reduce) {
  .split-handle {
    transition: none;
  }
}
```

**What CSS owns:** The flex geometry, handle appearance and hover/focus state, hit-area expansion via `::after`, iframe pointer-event suppression during drag, and cursor locking.

**What CSS cannot own:** The numeric split value, drag arithmetic, clamping, localStorage persistence, keyboard stepping, and ARIA value updates.

---

## React Implementation

### The Hook

```tsx
// useSplitPane.ts
import { useState, useCallback, useEffect, useRef } from 'react';

interface SplitPaneOptions {
  storageKey?: string;
  defaultRatio?: number;   // 0–1, fraction for the left pane
  minPercent?: number;     // minimum left-pane width as % of container
  maxPercent?: number;     // maximum left-pane width as % of container
  step?: number;           // keyboard step in percentage points
}

interface SplitPaneState {
  ratio: number;                              // current left pane fraction (0–1)
  isDragging: boolean;
  containerRef: React.RefObject<HTMLDivElement>;
  handleRef: React.RefObject<HTMLDivElement>;
  handlePointerDown: (e: React.PointerEvent) => void;
  handleKeyDown: (e: React.KeyboardEvent) => void;
  leftStyle: React.CSSProperties;
}

export function useSplitPane({
  storageKey = 'split-pane-ratio',
  defaultRatio = 0.3,
  minPercent = 15,
  maxPercent = 80,
  step = 2,
}: SplitPaneOptions = {}): SplitPaneState {
  const containerRef = useRef<HTMLDivElement>(null);
  const handleRef    = useRef<HTMLDivElement>(null);

  // Restore from localStorage, fall back to defaultRatio
  const [ratio, setRatio] = useState<number>(() => {
    try {
      const stored = localStorage.getItem(storageKey);
      if (stored !== null) {
        const parsed = parseFloat(stored);
        if (!isNaN(parsed)) return parsed;
      }
    } catch {
      // localStorage unavailable (SSR, private mode quota exceeded)
    }
    return defaultRatio;
  });

  const [isDragging, setIsDragging] = useState(false);

  // Clamp helper: keeps ratio within [min%, max%] expressed as fractions
  const clamp = useCallback(
    (value: number) => Math.min(maxPercent / 100, Math.max(minPercent / 100, value)),
    [minPercent, maxPercent]
  );

  // Persist whenever ratio changes
  useEffect(() => {
    try {
      localStorage.setItem(storageKey, String(ratio));
    } catch {
      // quota exceeded or SSR — silently skip
    }
  }, [ratio, storageKey]);

  // ── Drag logic ──────────────────────────────────────────────────────
  const handlePointerDown = useCallback(
    (e: React.PointerEvent) => {
      e.preventDefault();
      // Capture pointer on the handle so we receive move events
      // even when the pointer leaves the element
      (e.target as Element).setPointerCapture(e.pointerId);
      setIsDragging(true);
    },
    []
  );

  useEffect(() => {
    if (!isDragging) return;

    const container = containerRef.current;
    if (!container) return;

    const onPointerMove = (e: PointerEvent) => {
      const rect = container.getBoundingClientRect();
      // clientX relative to container left edge, divided by container width
      const newRatio = (e.clientX - rect.left) / rect.width;
      setRatio(clamp(newRatio));
    };

    const onPointerUp = () => {
      setIsDragging(false);
    };

    // Listen on document so drag works even if pointer moves fast
    // outside the handle or pane boundaries
    document.addEventListener('pointermove', onPointerMove);
    document.addEventListener('pointerup',   onPointerUp);
    document.addEventListener('pointercancel', onPointerUp);

    return () => {
      document.removeEventListener('pointermove', onPointerMove);
      document.removeEventListener('pointerup',   onPointerUp);
      document.removeEventListener('pointercancel', onPointerUp);
    };
  }, [isDragging, clamp]);

  // ── Keyboard resize (Alt + Arrow keys) ───────────────────────────────
  const handleKeyDown = useCallback(
    (e: React.KeyboardEvent) => {
      if (!e.altKey) return;

      let delta = 0;
      if (e.key === 'ArrowLeft')  delta = -(step / 100);
      if (e.key === 'ArrowRight') delta =  (step / 100);
      if (delta === 0) return;

      e.preventDefault();
      setRatio(prev => clamp(prev + delta));
    },
    [step, clamp]
  );

  // ── Re-clamp on container resize (e.g. window resize) ────────────────
  useEffect(() => {
    const container = containerRef.current;
    if (!container || typeof ResizeObserver === 'undefined') return;

    const observer = new ResizeObserver(() => {
      // Re-apply clamp without changing state if already in range
      setRatio(prev => clamp(prev));
    });

    observer.observe(container);
    return () => observer.disconnect();
  }, [clamp]);

  // ── Update ARIA value on handle ───────────────────────────────────────
  useEffect(() => {
    const handle = handleRef.current;
    if (!handle) return;
    const pct = Math.round(ratio * 100);
    handle.setAttribute('aria-valuenow',  String(pct));
    handle.setAttribute('aria-valuetext', `${pct} percent`);
  }, [ratio]);

  return {
    ratio,
    isDragging,
    containerRef,
    handleRef,
    handlePointerDown,
    handleKeyDown,
    leftStyle: { flexBasis: `${ratio * 100}%` },
  };
}
```

### The Component

```tsx
// SplitPaneEditor.tsx
import { useRef } from 'react';
import { useSplitPane } from './useSplitPane';

interface SplitPaneEditorProps {
  left: React.ReactNode;
  right: React.ReactNode;
  storageKey?: string;
  defaultRatio?: number;
  minPercent?: number;
  maxPercent?: number;
}

export function SplitPaneEditor({
  left,
  right,
  storageKey = 'split-pane-ratio',
  defaultRatio = 0.3,
  minPercent = 15,
  maxPercent = 80,
}: SplitPaneEditorProps) {
  const {
    isDragging,
    containerRef,
    handleRef,
    handlePointerDown,
    handleKeyDown,
    leftStyle,
  } = useSplitPane({ storageKey, defaultRatio, minPercent, maxPercent });

  return (
    <div
      ref={containerRef}
      className={`split-container${isDragging ? ' is-dragging' : ''}`}
    >
      {/* Left pane — flex-basis controlled by JS */}
      <div
        className="split-pane split-pane--left"
        style={leftStyle}
        role="group"
        aria-label="Left pane"
      >
        {left}
      </div>

      {/* Drag handle */}
      <div
        ref={handleRef}
        className={`split-handle${isDragging ? ' is-dragging' : ''}`}
        role="separator"
        aria-label="Resize panes. Use Alt+Left and Alt+Right arrow keys."
        aria-orientation="vertical"
        aria-valuemin={minPercent}
        aria-valuemax={maxPercent}
        aria-valuenow={Math.round(defaultRatio * 100)}
        aria-valuetext={`${Math.round(defaultRatio * 100)} percent`}
        tabIndex={0}
        onPointerDown={handlePointerDown}
        onKeyDown={handleKeyDown}
      />

      {/* Right pane — fills remaining space via flex-grow:1 in CSS */}
      <div
        className="split-pane split-pane--right"
        role="group"
        aria-label="Right pane"
      >
        {right}
      </div>
    </div>
  );
}
```

### Usage

```tsx
// App.tsx
import { SplitPaneEditor } from './SplitPaneEditor';

export function App() {
  return (
    <div style={{ width: '100vw', height: '100vh' }}>
      <SplitPaneEditor
        storageKey="vscode-split"
        defaultRatio={0.25}
        minPercent={15}
        maxPercent={75}
        left={<FileExplorer />}
        right={<CodeEditor />}
      />
    </div>
  );
}
```

---

## Angular Implementation

### Service

```typescript
// split-pane.service.ts
import { Injectable, signal, computed, effect } from '@angular/core';

@Injectable()
export class SplitPaneService {
  private readonly storageKey: string;
  private readonly minFraction: number;
  private readonly maxFraction: number;
  readonly stepFraction: number;

  private _ratio = signal<number>(0.3);

  readonly ratio      = this._ratio.asReadonly();
  readonly leftBasis  = computed(() => `${this._ratio() * 100}%`);
  readonly ariaValue  = computed(() => Math.round(this._ratio() * 100));

  constructor() {
    this.storageKey  = 'split-pane-ratio';
    this.minFraction = 0.15;
    this.maxFraction = 0.80;
    this.stepFraction = 0.02;
    this._restore();

    // Persist on every change
    effect(() => {
      try {
        localStorage.setItem(this.storageKey, String(this._ratio()));
      } catch { /* quota / SSR */ }
    });
  }

  configure(opts: {
    storageKey?: string;
    defaultRatio?: number;
    minPercent?: number;
    maxPercent?: number;
    step?: number;
  }) {
    Object.assign(this, {
      storageKey:   opts.storageKey  ?? this.storageKey,
      minFraction:  (opts.minPercent ?? 15) / 100,
      maxFraction:  (opts.maxPercent ?? 80) / 100,
      stepFraction: (opts.step       ?? 2)  / 100,
    });
    this._restore();
  }

  setRatio(value: number) {
    this._ratio.set(this._clamp(value));
  }

  step(direction: 'left' | 'right') {
    const delta = direction === 'right' ? this.stepFraction : -this.stepFraction;
    this._ratio.update(r => this._clamp(r + delta));
  }

  private _clamp(value: number): number {
    return Math.min(this.maxFraction, Math.max(this.minFraction, value));
  }

  private _restore() {
    try {
      const stored = localStorage.getItem(this.storageKey);
      if (stored !== null) {
        const parsed = parseFloat(stored);
        if (!isNaN(parsed)) {
          this._ratio.set(this._clamp(parsed));
          return;
        }
      }
    } catch { /* SSR / private mode */ }
  }
}
```

### Component

```typescript
// split-pane.component.ts
import {
  Component, Input, ContentChild, TemplateRef,
  ElementRef, ViewChild, OnDestroy, inject, effect
} from '@angular/core';
import { NgTemplateOutlet, NgClass } from '@angular/common';
import { SplitPaneService } from './split-pane.service';

@Component({
  selector: 'app-split-pane',
  standalone: true,
  imports: [NgTemplateOutlet, NgClass],
  providers: [SplitPaneService],   // scoped per instance
  template: `
    <div
      #containerEl
      class="split-container"
      [class.is-dragging]="isDragging"
    >
      <!-- Left pane -->
      <div
        class="split-pane split-pane--left"
        [style.flex-basis]="svc.leftBasis()"
        role="group"
        [attr.aria-label]="leftLabel"
      >
        <ng-container *ngTemplateOutlet="leftTemplate" />
      </div>

      <!-- Handle -->
      <div
        #handleEl
        class="split-handle"
        [class.is-dragging]="isDragging"
        role="separator"
        aria-orientation="vertical"
        [attr.aria-label]="'Resize panes. Use Alt+Left and Alt+Right.'"
        [attr.aria-valuemin]="minPercent"
        [attr.aria-valuemax]="maxPercent"
        [attr.aria-valuenow]="svc.ariaValue()"
        [attr.aria-valuetext]="svc.ariaValue() + ' percent'"
        tabindex="0"
        (pointerdown)="onPointerDown($event)"
        (keydown)="onKeyDown($event)"
      ></div>

      <!-- Right pane -->
      <div
        class="split-pane split-pane--right"
        role="group"
        [attr.aria-label]="rightLabel"
      >
        <ng-container *ngTemplateOutlet="rightTemplate" />
      </div>
    </div>
  `,
})
export class SplitPaneComponent implements OnDestroy {
  @Input() leftLabel  = 'Left pane';
  @Input() rightLabel = 'Right pane';
  @Input() minPercent = 15;
  @Input() maxPercent = 80;
  @Input() storageKey = 'split-pane-ratio';
  @Input() defaultRatio = 0.3;

  @ContentChild('leftPane',  { read: TemplateRef }) leftTemplate!:  TemplateRef<unknown>;
  @ContentChild('rightPane', { read: TemplateRef }) rightTemplate!: TemplateRef<unknown>;

  @ViewChild('containerEl') containerEl!: ElementRef<HTMLDivElement>;
  @ViewChild('handleEl')    handleEl!:    ElementRef<HTMLDivElement>;

  svc = inject(SplitPaneService);
  isDragging = false;

  private moveListener?: (e: PointerEvent) => void;
  private upListener?:   (e: PointerEvent) => void;
  private resizeObserver?: ResizeObserver;

  ngAfterViewInit() {
    this.svc.configure({
      storageKey:   this.storageKey,
      defaultRatio: this.defaultRatio,
      minPercent:   this.minPercent,
      maxPercent:   this.maxPercent,
    });

    // Re-clamp on container resize
    this.resizeObserver = new ResizeObserver(() => {
      this.svc.setRatio(this.svc.ratio());   // triggers internal clamp
    });
    this.resizeObserver.observe(this.containerEl.nativeElement);
  }

  onPointerDown(e: PointerEvent) {
    e.preventDefault();
    (e.target as Element).setPointerCapture(e.pointerId);
    this.isDragging = true;

    this.moveListener = (ev: PointerEvent) => {
      const rect = this.containerEl.nativeElement.getBoundingClientRect();
      this.svc.setRatio((ev.clientX - rect.left) / rect.width);
    };

    this.upListener = () => {
      this.isDragging = false;
      document.removeEventListener('pointermove', this.moveListener!);
      document.removeEventListener('pointerup',   this.upListener!);
      document.removeEventListener('pointercancel', this.upListener!);
    };

    document.addEventListener('pointermove',   this.moveListener);
    document.addEventListener('pointerup',     this.upListener);
    document.addEventListener('pointercancel', this.upListener);
  }

  onKeyDown(e: KeyboardEvent) {
    if (!e.altKey) return;
    if (e.key === 'ArrowLeft')  { e.preventDefault(); this.svc.step('left');  }
    if (e.key === 'ArrowRight') { e.preventDefault(); this.svc.step('right'); }
  }

  ngOnDestroy() {
    this.resizeObserver?.disconnect();
    if (this.moveListener) document.removeEventListener('pointermove', this.moveListener);
    if (this.upListener)   document.removeEventListener('pointerup',   this.upListener);
  }
}
```

### Usage

```html
<!-- app.component.html -->
<app-split-pane
  storageKey="vscode-split"
  [defaultRatio]="0.25"
  [minPercent]="15"
  [maxPercent]="75"
  leftLabel="File explorer"
  rightLabel="Code editor"
>
  <ng-template #leftPane>
    <app-file-explorer />
  </ng-template>

  <ng-template #rightPane>
    <app-code-editor />
  </ng-template>
</app-split-pane>
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Handle is keyboard focusable | `tabindex="0"` on the handle element |
| Handle role identifies it as a separator | `role="separator"` |
| Current split value announced | `aria-valuenow` and `aria-valuetext` updated on every change |
| Min/max range communicated | `aria-valuemin` / `aria-valuemax` set to percentage values |
| Keyboard resize gesture documented | `aria-label` on handle describes Alt+Arrow keys |
| Arrow key resize works without mouse | `keydown` listener fires `step()` on `Alt+ArrowLeft/Right` |
| No text selection during drag | `user-select: none` on container via `.is-dragging` class |
| Iframes do not intercept pointer events | `.split-container.is-dragging iframe { pointer-events: none }` |
| Focus ring visible on handle | `box-shadow` outline on `:focus-visible` |
| Pane content regions labelled | `role="group"` with `aria-label` on each pane |
| Animation respects reduced motion | `transition: none` in `@media (prefers-reduced-motion: reduce)` |

---

## Production Pitfalls

**1. Pointer events stolen by iframes inside panes**
If either pane contains an `<iframe>` (a sandboxed preview, a Storybook frame, a PDF viewer), the iframe captures pointer events during a drag as soon as the cursor enters it. The drag freezes. Fix: add `.split-container.is-dragging iframe { pointer-events: none }` to CSS. Remove the class the moment `pointerup` fires so the iframe regains interactivity.

**2. `setPointerCapture` must be called on the correct target**
Calling `element.setPointerCapture(id)` on the wrong element (e.g. the container instead of the handle) causes `pointermove` events to stop firing once the pointer leaves the handle. Always call `(e.target as Element).setPointerCapture(e.pointerId)` in `pointerdown`, using the actual event target.

**3. Stored ratio becomes invalid after container width changes**
If a user resizes their browser window to 400px with a stored 75% left-pane ratio, the left pane becomes wider than the viewport allows. Fix: re-run the clamp whenever a `ResizeObserver` fires on the container. Since the ratio is a fraction, the clamp only needs to ensure the resulting pixel widths are above the minimum — a percentage-based min/max handles this without pixel arithmetic.

**4. `localStorage` throws in private/incognito mode in some browsers**
`localStorage.setItem` throws a `QuotaExceededError` in Safari's private mode even when no data is stored. Wrap every `localStorage` call in a `try/catch`. The split still works; it just won't persist across sessions.

**5. RTL layouts reverse the drag direction**
In a right-to-left document (`dir="rtl"`), the left pane is visually on the right. Moving the handle right in screen coordinates should increase the right-pane (logical left pane) size, which means the ratio calculation inverts. Fix: check `getComputedStyle(container).direction === 'rtl'` and subtract from `rect.right` instead of `rect.left` when computing the fraction.

**6. Drag starts on text inside panes when clicking the narrow handle**
If the handle's hit area is only 5px wide, users frequently miss it and land on pane text instead, causing a browser text-selection drag instead of a pane resize. Fix: expand the invisible hit area with a `::after` pseudo-element (`inset: 0 -4px`) so the effective target is 13px wide while the visible bar stays 5px.

**7. Keyboard step size feels wrong at different container widths**
A 2% step in a 1400px container is 28px — comfortable. The same 2% step in a 400px container is 8px — acceptable but noticeable. Switching to a fixed pixel step (e.g., 10px) and dividing by `containerWidth` to get a fraction produces more consistent feel across viewport sizes.

---

## Interview Angle

**Q: "How would you implement a VS Code-style resizable split pane that persists its position?"**

A strong answer follows the shape of the actual implementation:

Start with **data model**: one number (the left pane's fraction of total container width, 0–1). Everything derives from this: the left pane's `flex-basis`, the right pane's implicit fill via `flex-grow: 1`, the ARIA `aria-valuenow`. Never store two widths — they must always sum to 100% minus the handle's fixed width, so tracking one is sufficient and keeps the invariant trivially.

Then cover **drag mechanics**: listen for `pointerdown` on the handle, call `setPointerCapture` so `pointermove` follows the pointer even outside the element, compute `(clientX - containerLeft) / containerWidth` on every move, clamp to `[min, max]`, and update state. Detach listeners on `pointerup` and `pointercancel`. Emphasise that listeners go on `document` during drag, not on the handle — this is what prevents the pointer moving faster than React/Angular can re-render from breaking the drag.

Then address **persistence**: `localStorage` read synchronously on initialization (avoids a flash of default layout), write on every state change, wrap in `try/catch` for private-mode Safari. On SSR, gate the read behind `typeof window !== 'undefined'`.

Mention **edge cases**: iframe pointer-event interception, RTL layout inversion, and the `ResizeObserver` re-clamp for window resize.

**Follow-up: "Why use `flex-basis` instead of `width` on the left pane?"**

`flex-basis` participates in the flex algorithm, so the right pane's `flex-grow: 1` correctly fills the remainder after accounting for both the left pane and the fixed-width handle — no arithmetic needed. Setting `width` instead can produce unexpected results when the flex container itself changes size, because `width` does not interact with the flex sizing model the same way. `flex-basis` is the explicit declaration of the pane's "starting size before flex growth/shrinkage is applied" — exactly what a manually dragged pane represents.
