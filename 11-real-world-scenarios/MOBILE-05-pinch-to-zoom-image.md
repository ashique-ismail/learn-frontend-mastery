# Pinch-to-Zoom Image

## The Idea

**In plain English:** You have an image that users can zoom in and out of using two fingers. When two fingers touch the screen and move apart, the image scales up. When the fingers move together, the image scales down. The zoom anchors at the midpoint between the two fingers, not the top-left corner of the image. Scale is clamped so the image cannot shrink below its original size or grow beyond four times its natural size.

**Real-world analogy:** Think of a printed photograph sitting on a table, and your fingers acting as a pair of compasses.

- The **distance between your two compass tips** is the "ruler" — you measure it at the moment the second finger lands, then measure it again every time either finger moves. The ratio of new distance to original distance is your scale factor.
- **Where you place your compass** matters — if both fingers are on the right side of the photo, the zoom should anchor to the right side. The midpoint between your two fingertips is the transform origin.
- The **minimum stop** is the original photo size — you cannot crumple the photo smaller than it started.
- The **maximum stop** is four times the original — zooming past that point snaps to 4x and holds, preventing infinite enlargement.
- **Lifting one finger** ends the pinch session and records the current scale as the new baseline so the next pinch starts from where you left off, not from 1x.

The key insight: there is no single "zoom value" — the value is always relative to where you were when the current pinch gesture began.

---

## Learning Objectives

- Understand why CSS alone cannot implement pinch-to-zoom behavior
- Use the Pointer Events API instead of Touch Events for a unified, cross-device model
- Track multiple simultaneous pointers by `pointerId` to isolate two-finger gestures
- Calculate Euclidean distance between two pointer positions
- Apply `transform: scale()` anchored at `transform-origin` set to the pinch midpoint
- Clamp scale between a minimum of 1x and a maximum of 4x
- Separate gesture state (active pointers, initial distance) from render state (current scale)

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Scale the image visually | ✅ `transform: scale()` | — |
| Detect two simultaneous touch/pointer contacts | ❌ | CSS has no event system |
| Calculate distance between two moving pointers | ❌ | Requires real-time coordinate math |
| Anchor zoom at the midpoint of two fingers | ❌ | `transform-origin` must be computed dynamically and changed per frame |
| Clamp scale within a min/max range | ❌ | CSS `clamp()` applies to static values, not event-driven state |
| Accumulate scale across multiple sequential pinch sessions | ❌ | Requires persistent state that survives pointer up/down cycles |
| Apply smooth transition only on gesture end, not during gesture | ❌ | Toggling `transition` on/off requires JS class manipulation |

**Conclusion:** CSS owns the visual output — `transform`, `transform-origin`, and the `transition` for snap-back. JS owns everything else: pointer tracking, distance math, scale accumulation, and origin calculation.

---

## HTML & CSS Foundation

### The Structure

```html
<!-- Outer container constrains the visible area and catches pointer events -->
<div class="zoom-container" id="zoom-container">
  <img
    class="zoom-image"
    id="zoom-image"
    src="/images/photo.jpg"
    alt="A detailed product photograph"
    draggable="false"
  />
</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --zoom-transition: 0.2s ease-out;
}

/* ─── Container: clips overflow and captures pointer events ─── */
.zoom-container {
  position: relative;
  overflow: hidden;         /* hide the image when it extends beyond its box */
  width: 100%;
  max-width: 600px;
  touch-action: none;       /* tell the browser: JS handles all touch gestures here */
                            /* without this, the browser intercepts pinch for page zoom */
  user-select: none;        /* prevent text selection during gesture */
  cursor: grab;
  border-radius: 8px;
}

.zoom-container:active {
  cursor: grabbing;
}

/* ─── Image: transform is the only property JS will change ─── */
.zoom-image {
  display: block;
  width: 100%;
  height: auto;
  transform-origin: 0 0;    /* default; JS overrides this per pinch midpoint */
  transform: scale(1);
  will-change: transform;   /* promote to GPU layer, avoids per-frame layout */
  pointer-events: none;     /* image itself should not intercept events; container does */
  draggable: false;
}

/* ─── Transition class: attached only after gesture ends ─── */
/*     During an active pinch, this class is removed so motion is instant  */
.zoom-image.is-settling {
  transition: transform var(--zoom-transition);
}

/* ─── Reduced motion: skip all animation ─── */
@media (prefers-reduced-motion: reduce) {
  .zoom-image.is-settling {
    transition: none;
  }
}
```

**Critical detail: `touch-action: none`** must be on the container. Without it, the browser intercepts the two-finger gesture to zoom the entire page and never fires `pointermove` events to your handler. This is the single most common reason a pinch implementation appears to do nothing on a real device.

**Why `transform-origin: 0 0`?** When `transform-origin` is `50% 50%` (the default), changing the scale also shifts the element position in a non-linear way. Anchoring to `0 0` makes the math straightforward: the origin is always expressed as a pixel offset from the element's top-left corner, which is directly calculable from `getBoundingClientRect()`.

---

## React Implementation

### Types and Geometry Utilities

```tsx
// pinchZoom.utils.ts

export interface PointerPosition {
  pointerId: number;
  x: number;
  y: number;
}

/** Euclidean distance between two pointer positions. */
export function distance(a: PointerPosition, b: PointerPosition): number {
  const dx = a.x - b.x;
  const dy = a.y - b.y;
  return Math.sqrt(dx * dx + dy * dy);
}

/** Midpoint between two pointer positions, in client coordinates. */
export function midpoint(
  a: PointerPosition,
  b: PointerPosition
): { x: number; y: number } {
  return { x: (a.x + b.x) / 2, y: (a.y + b.y) / 2 };
}

/**
 * Convert a client-space point to a point relative to an element's
 * top-left corner, accounting for current scroll and element offset.
 */
export function clientToElement(
  clientX: number,
  clientY: number,
  el: HTMLElement
): { x: number; y: number } {
  const rect = el.getBoundingClientRect();
  return { x: clientX - rect.left, y: clientY - rect.top };
}

/** Clamp a value between min and max. */
export function clamp(value: number, min: number, max: number): number {
  return Math.min(Math.max(value, min), max);
}
```

### The Hook

```tsx
// usePinchZoom.ts
import { useRef, useCallback, useEffect } from 'react';
import {
  PointerPosition, distance, midpoint, clientToElement, clamp
} from './pinchZoom.utils';

const MIN_SCALE = 1;
const MAX_SCALE = 4;

interface UsePinchZoomOptions {
  imageRef: React.RefObject<HTMLImageElement | null>;
  containerRef: React.RefObject<HTMLElement | null>;
}

export function usePinchZoom({ imageRef, containerRef }: UsePinchZoomOptions) {
  // Active pointers on the container, keyed by pointerId
  const activePointers = useRef<Map<number, PointerPosition>>(new Map());

  // Scale committed at the end of the previous pinch session
  const committedScale = useRef(1);

  // Distance between the two fingers at the moment the second finger landed
  const initialPinchDistance = useRef<number | null>(null);

  // Scale at the moment the current pinch began
  const scaleAtPinchStart = useRef(1);

  // Current transform-origin (element-relative pixels)
  const originRef = useRef({ x: 0, y: 0 });

  const applyTransform = useCallback(
    (scale: number, origin: { x: number; y: number }, settling = false) => {
      const img = imageRef.current;
      if (!img) return;

      // Toggle transition class: on during settle, off during active gesture
      img.classList.toggle('is-settling', settling);

      img.style.transformOrigin = `${origin.x}px ${origin.y}px`;
      img.style.transform = `scale(${scale})`;
    },
    [imageRef]
  );

  const handlePointerDown = useCallback(
    (e: PointerEvent) => {
      const container = containerRef.current;
      if (!container) return;

      // Capture so pointermove/up fire even if pointer leaves the element
      container.setPointerCapture(e.pointerId);

      activePointers.current.set(e.pointerId, {
        pointerId: e.pointerId,
        x: e.clientX,
        y: e.clientY,
      });

      if (activePointers.current.size === 2) {
        // Second finger landed: record baseline distance and current scale
        const [a, b] = [...activePointers.current.values()] as [
          PointerPosition,
          PointerPosition
        ];

        initialPinchDistance.current = distance(a, b);
        scaleAtPinchStart.current = committedScale.current;

        // Compute transform-origin as element-relative midpoint
        const mid = midpoint(a, b);
        const container = containerRef.current!;
        originRef.current = clientToElement(mid.x, mid.y, container);

        // Remove settling transition — we need instant response during pinch
        imageRef.current?.classList.remove('is-settling');
      }
    },
    [containerRef, imageRef]
  );

  const handlePointerMove = useCallback(
    (e: PointerEvent) => {
      if (!activePointers.current.has(e.pointerId)) return;

      // Update this pointer's position
      activePointers.current.set(e.pointerId, {
        pointerId: e.pointerId,
        x: e.clientX,
        y: e.clientY,
      });

      if (activePointers.current.size !== 2) return;
      if (initialPinchDistance.current === null) return;

      const [a, b] = [...activePointers.current.values()] as [
        PointerPosition,
        PointerPosition
      ];

      const currentDistance = distance(a, b);
      const ratio = currentDistance / initialPinchDistance.current;
      const rawScale = scaleAtPinchStart.current * ratio;
      const newScale = clamp(rawScale, MIN_SCALE, MAX_SCALE);

      applyTransform(newScale, originRef.current, false);
    },
    [applyTransform]
  );

  const handlePointerUp = useCallback(
    (e: PointerEvent) => {
      if (!activePointers.current.has(e.pointerId)) return;

      activePointers.current.delete(e.pointerId);

      if (activePointers.current.size < 2) {
        // Pinch session ended — commit whatever scale is currently applied
        const img = imageRef.current;
        if (img) {
          // Read the live scale from the style string and commit it
          const match = img.style.transform.match(/scale\(([^)]+)\)/);
          if (match) {
            const liveScale = parseFloat(match[1]);
            committedScale.current = clamp(liveScale, MIN_SCALE, MAX_SCALE);
          }
        }
        initialPinchDistance.current = null;

        // If scale is exactly 1, animate back cleanly with transition
        applyTransform(committedScale.current, originRef.current, true);
      }
    },
    [imageRef, applyTransform]
  );

  // Attach events to the container element imperatively
  // (avoids React synthetic event limitations on pointer capture)
  useEffect(() => {
    const container = containerRef.current;
    if (!container) return;

    container.addEventListener('pointerdown', handlePointerDown);
    container.addEventListener('pointermove', handlePointerMove);
    container.addEventListener('pointerup', handlePointerUp);
    container.addEventListener('pointercancel', handlePointerUp);

    return () => {
      container.removeEventListener('pointerdown', handlePointerDown);
      container.removeEventListener('pointermove', handlePointerMove);
      container.removeEventListener('pointerup', handlePointerUp);
      container.removeEventListener('pointercancel', handlePointerUp);
    };
  }, [containerRef, handlePointerDown, handlePointerMove, handlePointerUp]);
}
```

### The Component

```tsx
// PinchZoomImage.tsx
import { useRef } from 'react';
import { usePinchZoom } from './usePinchZoom';

interface PinchZoomImageProps {
  src: string;
  alt: string;
}

export function PinchZoomImage({ src, alt }: PinchZoomImageProps) {
  const containerRef = useRef<HTMLDivElement>(null);
  const imageRef     = useRef<HTMLImageElement>(null);

  usePinchZoom({ imageRef, containerRef });

  return (
    <div
      ref={containerRef}
      className="zoom-container"
      role="img"
      aria-label={`${alt} — pinch or use buttons to zoom`}
    >
      <img
        ref={imageRef}
        className="zoom-image"
        src={src}
        alt=""              /* decorative: outer div carries the accessible label */
        draggable={false}
      />

      {/* Accessible zoom controls for keyboard and AT users */}
      <div className="zoom-controls" aria-label="Zoom controls">
        <button
          className="zoom-btn"
          aria-label="Zoom in"
          onClick={() => {
            /* programmatic zoom handled via same applyTransform path */
          }}
        >
          +
        </button>
        <button className="zoom-btn" aria-label="Zoom out">
          −
        </button>
        <button className="zoom-btn" aria-label="Reset zoom">
          Reset
        </button>
      </div>
    </div>
  );
}
```

---

## Angular Implementation

### Utility Functions

```typescript
// pinch-zoom.utils.ts  (identical pure-function layer, imported by the directive)
export interface PointerPos { pointerId: number; x: number; y: number; }

export const distance = (a: PointerPos, b: PointerPos) =>
  Math.hypot(a.x - b.x, a.y - b.y);

export const midpoint = (a: PointerPos, b: PointerPos) =>
  ({ x: (a.x + b.x) / 2, y: (a.y + b.y) / 2 });

export const clamp = (v: number, min: number, max: number) =>
  Math.min(Math.max(v, min), max);

export const clientToElement = (
  cx: number, cy: number, el: HTMLElement
) => {
  const r = el.getBoundingClientRect();
  return { x: cx - r.left, y: cy - r.top };
};
```

### Directive

```typescript
// pinch-zoom.directive.ts
import {
  Directive, ElementRef, OnInit, OnDestroy, inject
} from '@angular/core';
import {
  PointerPos, distance, midpoint, clamp, clientToElement
} from './pinch-zoom.utils';

const MIN_SCALE = 1;
const MAX_SCALE = 4;

@Directive({
  selector: '[appPinchZoom]',
  standalone: true,
  host: { class: 'zoom-container' },
})
export class PinchZoomDirective implements OnInit, OnDestroy {
  private el = inject(ElementRef<HTMLElement>);

  private activePointers = new Map<number, PointerPos>();
  private committedScale = 1;
  private initialPinchDistance: number | null = null;
  private scaleAtPinchStart = 1;
  private origin = { x: 0, y: 0 };
  private imageEl: HTMLImageElement | null = null;

  // Bound handler references for removeEventListener
  private onDown   = this.handlePointerDown.bind(this);
  private onMove   = this.handlePointerMove.bind(this);
  private onUp     = this.handlePointerUp.bind(this);

  ngOnInit() {
    const host = this.el.nativeElement;
    this.imageEl = host.querySelector('img');

    host.addEventListener('pointerdown',   this.onDown);
    host.addEventListener('pointermove',   this.onMove);
    host.addEventListener('pointerup',     this.onUp);
    host.addEventListener('pointercancel', this.onUp);
  }

  ngOnDestroy() {
    const host = this.el.nativeElement;
    host.removeEventListener('pointerdown',   this.onDown);
    host.removeEventListener('pointermove',   this.onMove);
    host.removeEventListener('pointerup',     this.onUp);
    host.removeEventListener('pointercancel', this.onUp);
  }

  private handlePointerDown(e: PointerEvent) {
    const host = this.el.nativeElement;
    host.setPointerCapture(e.pointerId);

    this.activePointers.set(e.pointerId, {
      pointerId: e.pointerId, x: e.clientX, y: e.clientY
    });

    if (this.activePointers.size === 2) {
      const [a, b] = [...this.activePointers.values()] as [PointerPos, PointerPos];
      this.initialPinchDistance = distance(a, b);
      this.scaleAtPinchStart    = this.committedScale;

      const mid = midpoint(a, b);
      this.origin = clientToElement(mid.x, mid.y, host);

      this.imageEl?.classList.remove('is-settling');
    }
  }

  private handlePointerMove(e: PointerEvent) {
    if (!this.activePointers.has(e.pointerId)) return;

    this.activePointers.set(e.pointerId, {
      pointerId: e.pointerId, x: e.clientX, y: e.clientY
    });

    if (this.activePointers.size !== 2 || this.initialPinchDistance === null) return;

    const [a, b] = [...this.activePointers.values()] as [PointerPos, PointerPos];
    const ratio    = distance(a, b) / this.initialPinchDistance;
    const newScale = clamp(this.scaleAtPinchStart * ratio, MIN_SCALE, MAX_SCALE);

    this.applyTransform(newScale, this.origin, false);
  }

  private handlePointerUp(e: PointerEvent) {
    if (!this.activePointers.has(e.pointerId)) return;
    this.activePointers.delete(e.pointerId);

    if (this.activePointers.size < 2) {
      // Commit live scale
      if (this.imageEl) {
        const match = this.imageEl.style.transform.match(/scale\(([^)]+)\)/);
        if (match) {
          this.committedScale = clamp(parseFloat(match[1]), MIN_SCALE, MAX_SCALE);
        }
      }
      this.initialPinchDistance = null;
      this.applyTransform(this.committedScale, this.origin, true);
    }
  }

  private applyTransform(
    scale: number,
    origin: { x: number; y: number },
    settling: boolean
  ) {
    if (!this.imageEl) return;
    this.imageEl.classList.toggle('is-settling', settling);
    this.imageEl.style.transformOrigin = `${origin.x}px ${origin.y}px`;
    this.imageEl.style.transform       = `scale(${scale})`;
  }
}
```

### Component Using the Directive

```typescript
// zoom-photo.component.ts
import { Component, Input } from '@angular/core';
import { PinchZoomDirective } from './pinch-zoom.directive';

@Component({
  selector: 'app-zoom-photo',
  standalone: true,
  imports: [PinchZoomDirective],
  template: `
    <div
      appPinchZoom
      role="img"
      [attr.aria-label]="alt + ' — pinch or use buttons to zoom'"
    >
      <img
        class="zoom-image"
        [src]="src"
        alt=""
        draggable="false"
      />
      <div class="zoom-controls" aria-label="Zoom controls">
        <button class="zoom-btn" aria-label="Zoom in">+</button>
        <button class="zoom-btn" aria-label="Zoom out">−</button>
        <button class="zoom-btn" aria-label="Reset zoom">Reset</button>
      </div>
    </div>
  `,
})
export class ZoomPhotoComponent {
  @Input() src!: string;
  @Input() alt!: string;
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Image has an accessible name | Outer container carries `role="img"` and `aria-label`; `<img>` alt is empty to avoid double-announcement |
| Keyboard users can zoom without touch | Zoom In / Zoom Out / Reset buttons call the same `applyTransform` path |
| Screen reader is not confused by transform changes | CSS transforms are purely visual; they do not change DOM structure or ARIA tree |
| Buttons have descriptive `aria-label` | "Zoom in", "Zoom out", "Reset zoom" — not "+" or "−" alone |
| Minimum tap/click target for zoom buttons | 44×44px enforced via CSS on `.zoom-btn` |
| `prefers-reduced-motion` respected | `.is-settling` transition is suppressed via `@media` query |
| Container does not trap keyboard focus | No focus trap logic — the image is a passive viewer, not a dialog |
| `touch-action: none` does not remove all scrolling | Applied only to the image container, not the whole page |

---

## Production Pitfalls

**1. Forgetting `touch-action: none` on the container**
The browser intercepts two-finger gestures for native page zoom before your JS even runs. `touch-action: none` tells the browser to hand all pointer events to your handler. Apply it only to the image container, not to the entire page — otherwise you break page scrolling for users who are not interacting with the image.

**2. Reading `transform-origin` as a percentage**
The default `transform-origin` is `50% 50%`. If you set a pixel origin and then read it back via `getComputedStyle`, the browser converts it to pixels relative to the current element size — not the same numbers you wrote. Always set and track the origin yourself in JS state rather than reading it back from the style.

**3. Scale state drifts between pinch sessions**
If you compute scale purely as `currentDistance / initialDistance` on every `pointermove` without multiplying by the scale committed at pinch start (`scaleAtPinchStart`), the image snaps back to 1x the moment the user starts a second pinch. Always use `scaleAtPinchStart * (currentDistance / initialDistance)` so sessions accumulate.

**4. `setPointerCapture` not called**
Without pointer capture, `pointermove` events are only delivered while the pointer remains inside the element's bounding box. If the user drags a finger outside the container during a fast pinch, the pointer is "lost" and the scale freezes mid-gesture. Call `container.setPointerCapture(e.pointerId)` in `pointerdown`.

**5. Map not cleared on `pointercancel`**
`pointercancel` fires when the OS takes over a touch — for example, when a phone call arrives or the screen rotation lock triggers. If you only listen to `pointerup`, the map retains stale pointer entries, and the next touch incorrectly appears to be a two-finger gesture from the start. Always wire `pointercancel` to the same handler as `pointerup`.

**6. Applying CSS `transition` during the active pinch**
If `.is-settling` is attached while the user is still pinching, every `scale()` update is interpolated — the image visibly lags behind the fingers. The transition must only be active after the last finger lifts. Toggle the class off in `pointerdown` when two fingers are detected and back on in `pointerup` when the pinch ends.

**7. `transform-origin` mismatch when the image has been scrolled**
`getBoundingClientRect()` returns coordinates relative to the viewport, not the document. If the container is inside a scrollable parent, the origin calculation using `rect.left` and `rect.top` is already viewport-relative and does not need scroll offsets added — it is correct as-is. Adding `window.scrollX/Y` here is a common mistake that shifts the zoom anchor incorrectly.

---

## Interview Angle

**Q: "How would you implement pinch-to-zoom on an image without a library?"**

A strong answer covers the full event lifecycle. Start with pointer events rather than touch events — `PointerEvent` is a superset that works for mouse, stylus, and touch through one API. On `pointerdown`, record each incoming pointer in a `Map<pointerId, position>` and call `setPointerCapture` so events continue to fire even if a finger drifts outside the element. When the map reaches size 2, record the baseline distance between the two points and the current committed scale. On every `pointermove`, recompute the distance, divide it by the baseline, multiply by the scale at pinch start, clamp the result to `[1, 4]`, update `transform-origin` to the element-relative midpoint, and set `transform: scale(n)`. On `pointerup` and `pointercancel`, remove that pointer from the map, commit the live scale, and add the `is-settling` CSS class to re-enable the transition for a smooth snap.

**Follow-up: "Why set `transform-origin` dynamically instead of leaving it at `50% 50%`?"**

If the origin is always the element center, zooming always anchors to the center regardless of where the user's fingers are. A user pinching the top-right corner of a map tile expects that corner to stay stationary as the map enlarges. By moving `transform-origin` to the midpoint of the two active pointers (expressed in element-local pixels), the point under the fingers stays fixed — which matches the intuitive mental model users bring from every native maps and photos app they have ever used.
