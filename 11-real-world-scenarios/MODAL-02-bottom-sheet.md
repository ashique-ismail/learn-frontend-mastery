# Bottom Sheet (Mobile Drawer)

## The Idea

**In plain English:** A bottom sheet is a panel that slides up from the bottom of the screen, partially or fully covering the page. It is the mobile-native alternative to a centered modal dialog. The user can swipe it down to dismiss it, tap the backdrop to close it, or pull it to snap it between height positions (half-height, full-height). It is used everywhere: share menus, filter panels, action sheets, quick-view product cards.

**Real-world analogy:** Think of a pull-out kitchen drawer under a countertop.

- The **countertop** is the page behind it — always visible above the drawer.
- **Pulling the drawer out halfway** = the sheet snapping to 50% height (a quick-glance peek state).
- **Pulling it all the way out** = full-screen sheet for rich content.
- **Pushing it back in** = swipe-to-dismiss, the drawer disappears below the counter.
- The **spring resistance** you feel when you pull a drawer is the velocity threshold — a slow, gentle push just repositions; a fast flick dismisses it entirely.
- **The rubber gasket at the bottom of the drawer** = `safe-area-inset-bottom`, making sure content never hides behind the iPhone notch or home indicator.

The key insight: a bottom sheet is not just a modal with `bottom: 0`. It requires its own gesture system, snap-point logic, and safe-area awareness that centered modals do not need.

---

## Learning Objectives

- Understand why `translateY` is the correct animation property (not `top` or `bottom`)
- Implement CSS snap points with `scroll-snap-type` for declarative multi-position sheets
- Build a swipe-to-dismiss gesture with pointer events and a velocity threshold
- Handle backdrop fade that tracks the sheet's drag position in real time
- Apply `safe-area-inset-bottom` so content is never hidden by notched phone chrome
- Lock body scroll when the sheet is open without breaking iOS rubber-band scrolling
- Wire all of this into typed React and Angular components

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Slide sheet in from `translateY(100%)` to `translateY(0)` | ✅ with a class toggle | — |
| Fade backdrop on open/close | ✅ with a class toggle | — |
| Snap sheet to 50% vs 100% height positions | ⚠️ CSS scroll-snap handles it inside a scroll container, but the sheet itself still needs JS to switch between snap points dynamically | CSS scroll-snap works only when the user scrolls — it cannot programmatically jump to a snap point without JS |
| Detect swipe-down gesture and track drag distance | ❌ | CSS has no pointer-event tracking |
| Apply velocity threshold to decide dismiss vs. snap back | ❌ | Requires `Date.now()` delta and pixel-distance math |
| Sync backdrop opacity to the live drag position (not just open/closed) | ❌ | Requires inline `style` updates per pointer-move frame |
| Lock body scroll while sheet is open (iOS-safe) | ❌ | `overflow: hidden` alone does not stop iOS momentum scrolling |
| Restore scroll position after close | ❌ | Requires storing `window.scrollY` before locking |
| Apply `env(safe-area-inset-bottom)` only when the sheet is at its lowest snap point | ❌ | Requires conditional class or inline style from JS |

**Conclusion:** CSS owns the enter/exit animation, the backdrop visibility, and the safe-area padding token. JS owns gesture tracking, snap-point switching, velocity math, scroll lock, and the real-time opacity binding during drag.

---

## HTML & CSS Foundation

### The Structure

```html
<!-- Backdrop -->
<div class="sheet-backdrop" aria-hidden="true"></div>

<!-- The sheet itself -->
<div
  class="bottom-sheet"
  role="dialog"
  aria-modal="true"
  aria-labelledby="sheet-title"
>
  <!-- Drag handle — visual affordance and tap target -->
  <div class="sheet-handle-bar" aria-hidden="true">
    <span class="sheet-handle-indicator"></span>
  </div>

  <!-- Content area — scrollable independently of the sheet position -->
  <div class="sheet-content" id="sheet-content">
    <h2 class="sheet-title" id="sheet-title">Sheet Title</h2>
    <p>Sheet body content goes here.</p>
  </div>
</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --sheet-radius:       16px;
  --sheet-bg:           #ffffff;
  --sheet-shadow:       0 -4px 24px rgba(0, 0, 0, 0.15);
  --sheet-transition:   0.35s cubic-bezier(0.32, 0.72, 0, 1);
  --sheet-backdrop-bg:  rgba(0, 0, 0, 0.4);
  --sheet-handle-color: #d1d5db;

  /* Snap heights as percentages of viewport height */
  --sheet-snap-peek:    45vh;
  --sheet-snap-full:    92vh;
}

/* ─── Backdrop ─── */
.sheet-backdrop {
  position: fixed;
  inset: 0;
  background: var(--sheet-backdrop-bg);
  opacity: 0;
  visibility: hidden;
  transition: opacity var(--sheet-transition), visibility var(--sheet-transition);
  z-index: 200;
  /* No pointer-events when hidden so underlying page is still interactive */
  pointer-events: none;
}

.sheet-backdrop.is-visible {
  opacity: 1;
  visibility: visible;
  pointer-events: auto;
}

/* ─── Sheet container ─── */
.bottom-sheet {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  height: var(--sheet-snap-full);  /* max height — JS clamps translate */
  background: var(--sheet-bg);
  border-radius: var(--sheet-radius) var(--sheet-radius) 0 0;
  box-shadow: var(--sheet-shadow);
  z-index: 201;
  display: flex;
  flex-direction: column;

  /* Closed state: fully below the viewport */
  transform: translateY(100%);
  transition: transform var(--sheet-transition);
  will-change: transform;

  /*
    Safe area inset: padding at the bottom so content never hides
    behind the iPhone home indicator or Android gesture bar.
    Only affects bottom of the sheet when fully open.
  */
  padding-bottom: env(safe-area-inset-bottom, 0px);
}

/* Open states — JS adds one of these classes */
.bottom-sheet.is-peek {
  transform: translateY(calc(var(--sheet-snap-full) - var(--sheet-snap-peek)));
}

.bottom-sheet.is-full {
  transform: translateY(0);
}

/*
  During a live drag, JS sets an inline style:
    transform: translateY(Xpx)
  The transition is disabled so it tracks the finger in real time.
*/
.bottom-sheet.is-dragging {
  transition: none;
}

/* ─── Drag handle bar ─── */
.sheet-handle-bar {
  flex-shrink: 0;
  display: flex;
  justify-content: center;
  align-items: center;
  height: 28px;
  cursor: grab;
  /* Wider tap target than the visual indicator */
  touch-action: none;   /* prevents the browser from stealing pointer events for scroll */
}

.sheet-handle-bar:active {
  cursor: grabbing;
}

.sheet-handle-indicator {
  display: block;
  width: 36px;
  height: 4px;
  border-radius: 2px;
  background: var(--sheet-handle-color);
}

/* ─── Scrollable content area ─── */
.sheet-content {
  flex: 1;
  overflow-y: auto;
  overflow-x: hidden;
  padding: 1rem 1.5rem;
  /* Momentum scrolling on iOS — this is intentional for content inside the sheet */
  -webkit-overflow-scrolling: touch;
  overscroll-behavior: contain;   /* stops scroll from propagating to the page */
}

/* ─── Body scroll lock ─── */
body.sheet-open {
  overflow: hidden;
  /* iOS Safari fix — position:fixed prevents rubber-band bleed */
  position: fixed;
  width: 100%;
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .bottom-sheet,
  .sheet-backdrop {
    transition: none;
  }
}
```

**What CSS owns:** enter/exit animation via `translateY`, backdrop fade, handle bar visuals, safe-area padding, reduced-motion fallback, the `is-dragging` state that strips the transition so drag tracks the finger.

**What CSS cannot own:** knowing which snap point is active, drag gesture math, velocity threshold, real-time opacity interpolation during drag, and the body scroll lock lifecycle.

---

## React Implementation

### Types

```tsx
// bottom-sheet.types.ts
export type SnapPoint = 'closed' | 'peek' | 'full';

export interface BottomSheetProps {
  isOpen: boolean;
  onClose: () => void;
  initialSnap?: Exclude<SnapPoint, 'closed'>;
  children: React.ReactNode;
  title: string;
}
```

### The Hook — Gesture + Snap Logic

```tsx
// useBottomSheet.ts
import { useRef, useState, useCallback, useEffect } from 'react';
import type { SnapPoint } from './bottom-sheet.types';

const SNAP_PEEK_VH = 0.45;
const SNAP_FULL_VH = 0.92;
const VELOCITY_DISMISS_THRESHOLD = 0.5;   // px/ms — faster than this = dismiss
const VELOCITY_EXPAND_THRESHOLD  = -0.4;  // negative = upward flick = expand

export function useBottomSheet(
  isOpen: boolean,
  onClose: () => void,
  initialSnap: Exclude<SnapPoint, 'closed'> = 'peek'
) {
  const [snap, setSnap] = useState<SnapPoint>('closed');
  const sheetRef        = useRef<HTMLDivElement>(null);
  const backdropRef     = useRef<HTMLDivElement>(null);

  // Drag state stored in refs — not state — to avoid re-renders on every pointer move
  const dragStart      = useRef<number>(0);
  const dragStartTime  = useRef<number>(0);
  const currentTranslate = useRef<number>(0);
  const isDragging     = useRef<boolean>(false);

  // Open/close driven by the isOpen prop
  useEffect(() => {
    if (isOpen) {
      setSnap(initialSnap);
      const scrollY = window.scrollY;
      document.body.style.top = `-${scrollY}px`;
      document.body.classList.add('sheet-open');
    } else {
      setSnap('closed');
      const scrollY = Math.abs(parseInt(document.body.style.top || '0', 10));
      document.body.classList.remove('sheet-open');
      document.body.style.top = '';
      window.scrollTo(0, scrollY);
    }
  }, [isOpen, initialSnap]);

  // Cleanup on unmount
  useEffect(() => () => {
    document.body.classList.remove('sheet-open');
    document.body.style.top = '';
  }, []);

  /*
    Compute the translateY value for a given snap point.
    The sheet height is --sheet-snap-full (92vh).
    translateY(0)   = fully open (full snap)
    translateY(X)   = peeking, where X = fullHeight - peekHeight
    translateY(100%) = closed (CSS default)
  */
  const getTranslateForSnap = useCallback((s: SnapPoint): number => {
    const fullH = window.innerHeight * SNAP_FULL_VH;
    const peekH = window.innerHeight * SNAP_PEEK_VH;
    if (s === 'full')  return 0;
    if (s === 'peek')  return fullH - peekH;
    return fullH;   // 'closed' — matches translateY(100%) visually
  }, []);

  // Real-time backdrop opacity during drag (0 = transparent, 1 = opaque)
  const setBackdropOpacity = useCallback((translateY: number) => {
    if (!backdropRef.current) return;
    const fullH   = window.innerHeight * SNAP_FULL_VH;
    const progress = Math.max(0, Math.min(1, 1 - translateY / fullH));
    backdropRef.current.style.opacity = String(progress);
  }, []);

  const onPointerDown = useCallback((e: React.PointerEvent<HTMLDivElement>) => {
    if (!sheetRef.current) return;
    isDragging.current   = true;
    dragStart.current    = e.clientY;
    dragStartTime.current = Date.now();
    currentTranslate.current = getTranslateForSnap(snap);
    sheetRef.current.classList.add('is-dragging');
    sheetRef.current.setPointerCapture(e.pointerId);
  }, [snap, getTranslateForSnap]);

  const onPointerMove = useCallback((e: React.PointerEvent<HTMLDivElement>) => {
    if (!isDragging.current || !sheetRef.current) return;
    const delta   = e.clientY - dragStart.current;
    const nextY   = Math.max(0, currentTranslate.current + delta);
    sheetRef.current.style.transform = `translateY(${nextY}px)`;
    setBackdropOpacity(nextY);
  }, [setBackdropOpacity]);

  const onPointerUp = useCallback((e: React.PointerEvent<HTMLDivElement>) => {
    if (!isDragging.current || !sheetRef.current) return;
    isDragging.current = false;
    sheetRef.current.classList.remove('is-dragging');
    sheetRef.current.style.transform = '';   // let CSS class take over

    const delta    = e.clientY - dragStart.current;
    const elapsed  = Date.now() - dragStartTime.current;
    const velocity = delta / elapsed;   // px/ms, positive = downward

    // Velocity threshold decision
    if (velocity > VELOCITY_DISMISS_THRESHOLD) {
      onClose();
      return;
    }
    if (velocity < VELOCITY_EXPAND_THRESHOLD) {
      setSnap('full');
      return;
    }

    // No strong flick — snap to nearest position based on drag position
    const fullH      = window.innerHeight * SNAP_FULL_VH;
    const peekH      = window.innerHeight * SNAP_PEEK_VH;
    const midpoint   = fullH - peekH / 2;
    const finalY     = currentTranslate.current + delta;

    if (finalY > fullH * 0.75) {
      onClose();
    } else if (finalY > midpoint) {
      setSnap('peek');
    } else {
      setSnap('full');
    }
  }, [onClose, getTranslateForSnap]);

  return {
    snap,
    setSnap,
    sheetRef,
    backdropRef,
    onPointerDown,
    onPointerMove,
    onPointerUp,
  };
}
```

### The Component

```tsx
// BottomSheet.tsx
import { useEffect, useRef } from 'react';
import { useBottomSheet } from './useBottomSheet';
import type { BottomSheetProps } from './bottom-sheet.types';

export function BottomSheet({
  isOpen,
  onClose,
  initialSnap = 'peek',
  title,
  children,
}: BottomSheetProps) {
  const {
    snap,
    sheetRef,
    backdropRef,
    onPointerDown,
    onPointerMove,
    onPointerUp,
  } = useBottomSheet(isOpen, onClose, initialSnap);

  const titleRef      = useRef<HTMLHeadingElement>(null);
  const firstFocusRef = useRef<HTMLButtonElement>(null);

  // Move focus into the sheet when it opens
  useEffect(() => {
    if (isOpen) {
      // Small delay lets the CSS transition start before focus moves
      requestAnimationFrame(() => titleRef.current?.focus());
    }
  }, [isOpen]);

  // Escape key dismisses the sheet
  useEffect(() => {
    if (!isOpen) return;
    const onKey = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onClose();
    };
    document.addEventListener('keydown', onKey);
    return () => document.removeEventListener('keydown', onKey);
  }, [isOpen, onClose]);

  // Derive CSS class from snap state
  const snapClass =
    snap === 'full' ? 'is-full'
    : snap === 'peek' ? 'is-peek'
    : '';

  return (
    <>
      {/* Backdrop */}
      <div
        ref={backdropRef}
        className={`sheet-backdrop ${isOpen ? 'is-visible' : ''}`}
        aria-hidden="true"
        onClick={onClose}
      />

      {/* Sheet */}
      <div
        ref={sheetRef}
        className={`bottom-sheet ${snapClass}`}
        role="dialog"
        aria-modal="true"
        aria-labelledby="sheet-title"
        aria-hidden={!isOpen}
      >
        {/* Drag handle — pointer events only on the handle bar */}
        <div
          className="sheet-handle-bar"
          onPointerDown={onPointerDown}
          onPointerMove={onPointerMove}
          onPointerUp={onPointerUp}
          aria-label="Drag to resize or dismiss"
          role="separator"
        >
          <span className="sheet-handle-indicator" />
        </div>

        {/* Hidden close button for keyboard users who cannot drag */}
        <button
          ref={firstFocusRef}
          className="sr-only"
          onClick={onClose}
          tabIndex={isOpen ? 0 : -1}
        >
          Close sheet
        </button>

        {/* Scrollable content */}
        <div className="sheet-content" id="sheet-content">
          <h2
            ref={titleRef}
            id="sheet-title"
            tabIndex={-1}   /* focusable programmatically, not in tab order */
          >
            {title}
          </h2>
          {children}
        </div>
      </div>
    </>
  );
}
```

### Usage Example

```tsx
// ProductQuickView.tsx
import { useState } from 'react';
import { BottomSheet } from './BottomSheet';

export function ProductQuickView() {
  const [open, setOpen] = useState(false);

  return (
    <>
      <button onClick={() => setOpen(true)}>Quick View</button>

      <BottomSheet
        isOpen={open}
        onClose={() => setOpen(false)}
        initialSnap="peek"
        title="Running Shoes — Nitro Elite"
      >
        <p>$189.99</p>
        <button>Add to cart</button>
      </BottomSheet>
    </>
  );
}
```

---

## Angular Implementation

### Model & Service

```typescript
// bottom-sheet.model.ts
export type SnapPoint = 'closed' | 'peek' | 'full';

export interface BottomSheetConfig {
  title: string;
  initialSnap?: Exclude<SnapPoint, 'closed'>;
}
```

```typescript
// bottom-sheet.service.ts
import { Injectable, signal, computed } from '@angular/core';
import type { SnapPoint, BottomSheetConfig } from './bottom-sheet.model';

@Injectable({ providedIn: 'root' })
export class BottomSheetService {
  private _isOpen  = signal(false);
  private _snap    = signal<SnapPoint>('closed');
  private _config  = signal<BottomSheetConfig>({ title: '' });
  private _scrollY = 0;

  readonly isOpen = this._isOpen.asReadonly();
  readonly snap   = this._snap.asReadonly();
  readonly config = this._config.asReadonly();

  readonly snapClass = computed(() => {
    const s = this._snap();
    if (s === 'full') return 'is-full';
    if (s === 'peek') return 'is-peek';
    return '';
  });

  open(config: BottomSheetConfig) {
    this._config.set(config);
    this._snap.set(config.initialSnap ?? 'peek');
    this._isOpen.set(true);
    this._scrollY = window.scrollY;
    document.body.style.top = `-${this._scrollY}px`;
    document.body.classList.add('sheet-open');
  }

  close() {
    this._isOpen.set(false);
    this._snap.set('closed');
    document.body.classList.remove('sheet-open');
    document.body.style.top = '';
    window.scrollTo(0, this._scrollY);
  }

  setSnap(point: SnapPoint) {
    if (point === 'closed') { this.close(); return; }
    this._snap.set(point);
  }
}
```

### Bottom Sheet Component

```typescript
// bottom-sheet.component.ts
import {
  Component, inject, ElementRef, ViewChild,
  AfterViewInit, OnDestroy, HostListener, effect
} from '@angular/core';
import { NgClass, NgIf } from '@angular/common';
import { BottomSheetService } from './bottom-sheet.service';

const SNAP_PEEK_VH = 0.45;
const SNAP_FULL_VH = 0.92;
const VELOCITY_DISMISS = 0.5;
const VELOCITY_EXPAND  = -0.4;

@Component({
  selector: 'app-bottom-sheet',
  standalone: true,
  imports: [NgClass, NgIf],
  template: `
    <!-- Backdrop -->
    <div
      #backdropEl
      class="sheet-backdrop"
      [class.is-visible]="service.isOpen()"
      aria-hidden="true"
      (click)="service.close()"
    ></div>

    <!-- Sheet -->
    <div
      #sheetEl
      class="bottom-sheet"
      [ngClass]="service.snapClass()"
      role="dialog"
      aria-modal="true"
      [attr.aria-hidden]="!service.isOpen()"
      aria-labelledby="sheet-title"
    >
      <!-- Drag handle -->
      <div
        class="sheet-handle-bar"
        (pointerdown)="onPointerDown($event)"
        (pointermove)="onPointerMove($event)"
        (pointerup)="onPointerUp($event)"
        role="separator"
        aria-label="Drag to resize or dismiss"
      >
        <span class="sheet-handle-indicator"></span>
      </div>

      <!-- Keyboard close button -->
      <button
        class="sr-only"
        (click)="service.close()"
        [attr.tabIndex]="service.isOpen() ? 0 : -1"
      >
        Close sheet
      </button>

      <!-- Content -->
      <div class="sheet-content">
        <h2 #titleEl id="sheet-title" tabindex="-1">
          {{ service.config().title }}
        </h2>
        <ng-content></ng-content>
      </div>
    </div>
  `,
})
export class BottomSheetComponent implements OnDestroy {
  service = inject(BottomSheetService);

  @ViewChild('sheetEl',   { static: true }) sheetEl!:   ElementRef<HTMLDivElement>;
  @ViewChild('backdropEl',{ static: true }) backdropEl!: ElementRef<HTMLDivElement>;
  @ViewChild('titleEl',   { static: true }) titleEl!:   ElementRef<HTMLHeadingElement>;

  private dragStart     = 0;
  private dragStartTime = 0;
  private currentTranslateY = 0;
  private dragging      = false;

  constructor() {
    // Focus the title whenever the sheet opens
    effect(() => {
      if (this.service.isOpen()) {
        requestAnimationFrame(() => this.titleEl?.nativeElement.focus());
      }
    });
  }

  @HostListener('document:keydown.escape')
  onEscape() {
    if (this.service.isOpen()) this.service.close();
  }

  private get fullH() { return window.innerHeight * SNAP_FULL_VH; }
  private get peekH() { return window.innerHeight * SNAP_PEEK_VH; }

  private translateForSnap(): number {
    const s = this.service.snap();
    if (s === 'full') return 0;
    if (s === 'peek') return this.fullH - this.peekH;
    return this.fullH;
  }

  onPointerDown(e: PointerEvent) {
    this.dragging          = true;
    this.dragStart         = e.clientY;
    this.dragStartTime     = Date.now();
    this.currentTranslateY = this.translateForSnap();
    const el = this.sheetEl.nativeElement;
    el.classList.add('is-dragging');
    el.setPointerCapture(e.pointerId);
  }

  onPointerMove(e: PointerEvent) {
    if (!this.dragging) return;
    const delta = e.clientY - this.dragStart;
    const nextY = Math.max(0, this.currentTranslateY + delta);
    this.sheetEl.nativeElement.style.transform = `translateY(${nextY}px)`;
    // Live backdrop opacity
    const progress = Math.max(0, Math.min(1, 1 - nextY / this.fullH));
    this.backdropEl.nativeElement.style.opacity = String(progress);
  }

  onPointerUp(e: PointerEvent) {
    if (!this.dragging) return;
    this.dragging = false;
    const el = this.sheetEl.nativeElement;
    el.classList.remove('is-dragging');
    el.style.transform = '';

    const delta    = e.clientY - this.dragStart;
    const elapsed  = Date.now() - this.dragStartTime;
    const velocity = delta / elapsed;

    if (velocity > VELOCITY_DISMISS) { this.service.close(); return; }
    if (velocity < VELOCITY_EXPAND)  { this.service.setSnap('full'); return; }

    const finalY   = this.currentTranslateY + delta;
    const midpoint = this.fullH - this.peekH / 2;

    if (finalY > this.fullH * 0.75)  this.service.close();
    else if (finalY > midpoint)       this.service.setSnap('peek');
    else                              this.service.setSnap('full');
  }

  ngOnDestroy() {
    this.service.close();
  }
}
```

### Usage from a Parent Component

```typescript
// product-card.component.ts
import { Component, inject } from '@angular/core';
import { BottomSheetService } from './bottom-sheet.service';
import { BottomSheetComponent } from './bottom-sheet.component';

@Component({
  selector: 'app-product-card',
  standalone: true,
  imports: [BottomSheetComponent],
  template: `
    <button (click)="openSheet()">Quick View</button>

    <app-bottom-sheet>
      <p>$189.99</p>
      <button>Add to cart</button>
    </app-bottom-sheet>
  `,
})
export class ProductCardComponent {
  private sheetService = inject(BottomSheetService);

  openSheet() {
    this.sheetService.open({ title: 'Running Shoes — Nitro Elite', initialSnap: 'peek' });
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Sheet has `role="dialog"` and `aria-modal="true"` | Prevents virtual cursor from leaving the sheet |
| Sheet has `aria-labelledby` pointing to the visible title | Screen reader announces the title when the sheet gains focus |
| `aria-hidden="true"` on the sheet when closed | Prevents screen reader from discovering off-screen content |
| Backdrop has `aria-hidden="true"` | It is a visual-only element with no semantic role |
| Focus moves into the sheet on open | `requestAnimationFrame` focuses the `<h2>` on open |
| Focus returns to the trigger button on close | The component consumer manages this — the service exposes `isOpen` for the trigger to watch |
| Keyboard-accessible close path | Hidden `Close sheet` button is in the focus order when open |
| Escape key dismisses the sheet | `HostListener` / `document.addEventListener` |
| Minimum drag handle tap target | `sheet-handle-bar` has `height: 28px` — taller than the 4px visual indicator |
| Sheet content is independently scrollable | `overflow-y: auto` on `.sheet-content`, `overscroll-behavior: contain` |
| Safe area inset at the bottom | `padding-bottom: env(safe-area-inset-bottom)` on `.bottom-sheet` |
| Reduced motion | `transition: none` via `@media (prefers-reduced-motion: reduce)` |

---

## Production Pitfalls

**1. Using `top` or `bottom` instead of `translateY` for the slide animation**
Animating `top` or `bottom` triggers layout recalculation on every frame, which causes jank on low-end Android devices. `translateY` runs entirely on the GPU compositor thread. Always animate transform, never position properties.

**2. `100vh` is wrong on iOS Safari**
iOS Safari's `100vh` includes the browser toolbar. The sheet appears taller than the visible screen, cutting off the top. Use `height: 100dvh` (dynamic viewport height) with a `100vh` fallback. For the snap-point calculation in JS, always use `window.innerHeight`, which reflects the actual visible area.

**3. Body scroll bleeds through on iOS**
`overflow: hidden` on `body` is not sufficient on iOS — the page still scrolls under the sheet. The correct fix is: store `window.scrollY`, then set `body { position: fixed; top: -Xpx; width: 100% }`. Restore both `position` and `scrollY` on close. Skipping the restore step causes the page to jump to the top when the sheet closes.

**4. Drag events stop during fast swipes**
If you only attach `pointermove` and `pointerup` to the handle element, the pointer can leave the handle during a fast swipe and the drag stops. Fix: call `element.setPointerCapture(e.pointerId)` on `pointerdown`. This routes all subsequent pointer events to that element regardless of where the pointer moves.

**5. Content inside the sheet scrolls and dismisses simultaneously**
When the user scrolls down through long sheet content, the sheet itself should not also start sliding down. Attach the drag handlers only to the handle bar element, not the entire sheet. Additionally, set `touch-action: none` on the handle bar only, so the browser still handles native scroll on `.sheet-content`.

**6. Backdrop opacity sticks during transitions**
During a drag, you set `backdropEl.style.opacity` inline. When the drag ends and a CSS transition kicks in, the inline opacity overrides the transition. Fix: immediately remove the inline `opacity` style on `pointerup` before snapping, and let the CSS class control it from that point forward.

**7. `safe-area-inset-bottom` only applies at the bottom snap position**
When the sheet is at its `peek` state, the bottom of the sheet is below the visible viewport — the inset padding is invisible and harmless. When fully open, the bottom of the sheet sits at the screen edge where the home indicator lives. The CSS `env()` declaration on `.bottom-sheet` handles this correctly for both states because the padding is always present but only visually matters when the sheet extends to the bottom edge.

**8. Velocity threshold not tuned to device pixel ratio**
Pointer events report coordinates in CSS pixels. On a high-DPI device, a "fast swipe" still produces the same CSS pixel velocity as on a 1x screen. This is correct — no DPR adjustment is needed. However, the threshold values (`0.5 px/ms`) need user testing on real devices, not just Chrome DevTools device simulation.

---

## Interview Angle

**Q: "How would you implement a bottom sheet with swipe-to-dismiss on mobile?"**

A strong answer addresses four layers:

1. **Animation layer** — `translateY` from `100%` to `0` is non-negotiable. Using `bottom`, `top`, or `height` for the transition animates layout, not compositing, and will drop frames on mid-range Android devices. The sheet height is fixed at `92vh`; only the Y offset changes.

2. **Gesture layer** — Pointer events (not touch events) give cross-device consistency. `setPointerCapture` on `pointerdown` is the key that prevents the drag from breaking when the finger moves faster than the element. Velocity is `deltaY / deltaTime` in px/ms; typical dismiss threshold is `0.4–0.6 px/ms` depending on your UX.

3. **Snap logic** — On `pointerup`, check velocity first (fast flick wins over position), then fall back to nearest snap point by comparing `finalTranslateY` against midpoints between each snap height.

4. **Platform edge cases** — `safe-area-inset-bottom` via `env()` for notched iPhones. `position: fixed; top: -Xpx` on `body` for iOS scroll-lock. `window.innerHeight` instead of `100vh` for JS calculations. `overscroll-behavior: contain` on the sheet content so inner scrolling does not propagate to the page.

**Follow-up: "What is the difference between a bottom sheet and a drawer? When would you use each?"**

A bottom sheet slides up from the bottom and is anchored to the viewport bottom — it is modal and temporary. It is appropriate for contextual actions, quick-view overlays, and filter panels where the user needs to return to the page. A side drawer slides in from the left or right and is typically used for persistent navigation — it usually has a persistent trigger (hamburger icon) and can be non-modal (the page is still visible and interactive beside it). The key technical difference: a drawer uses `translateX` and its snap logic is binary (open/closed); a bottom sheet uses `translateY` and typically has 2–3 snap positions with velocity-sensitive dismissal.
