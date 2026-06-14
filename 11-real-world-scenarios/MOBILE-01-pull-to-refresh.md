# Pull-to-Refresh

## The Idea

**In plain English:** The user drags the page downward past the top of its scroll position. While dragging, a loading indicator stretches into view — giving tactile feedback proportional to how far they've pulled. Once they cross a threshold distance and release, the indicator snaps into a fixed loading state and a data-fetch kicks off. When the fetch completes, the indicator animates out and the new content appears.

**Real-world analogy:** Think of a kitchen paper-towel holder with a spring tension mechanism.

- **Pulling the sheet down** = the touchmove gesture dragging the container below its natural rest position.
- **The resistance you feel** = rubber-band easing — the indicator moves less than your finger does, creating a physical "this has weight" sensation.
- **The click/release point** = the pull distance threshold. Cross it and the towel tears off (fetch triggers). Fall short and the spring pulls the sheet back (cancel, snap back to zero).
- **The new sheet appearing** = the refreshed data rendered after the network call resolves.
- **The spring snapping back** = the closing animation that returns the container to its normal scroll position.

The key insight: pull-to-refresh is not a scroll event. It is a touch gesture that starts only when the scroll position is already at the top. Conflating the two is the root cause of most broken implementations.

---

## Learning Objectives

- Distinguish between scroll events and touch events, and know when to use each
- Implement pull detection using `touchstart`, `touchmove`, and `touchend` with correct passive listener strategy
- Apply `overscroll-behavior: contain` to prevent the native browser pull-to-refresh from firing
- Model rubber-band easing to make the drag feel physical rather than mechanical
- Build a loading spinner with a CSS `@keyframes` animation that is decoupled from the pull distance
- Implement the same gesture with Pointer Events as a non-touch fallback for stylus and desktop testing
- Wire the complete flow in React (custom hook) and Angular (signal-based service)
- Avoid the production pitfalls: passive listener violations, iOS overscroll bleed, premature trigger on horizontal swipe

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Prevent native browser pull-to-refresh | ✅ with `overscroll-behavior: contain` | — |
| Detect when the user starts dragging downward | ❌ | CSS has no gesture or touch event awareness |
| Know whether scroll position is already at the top | ❌ | Requires reading `scrollTop` in JS |
| Calculate pull distance from touch coordinates | ❌ | `touchmove` clientY delta is a JS-only value |
| Apply rubber-band resistance to the drag | ❌ | Non-linear math (`sqrt`, `log`) is not expressible in CSS |
| Trigger a data fetch when threshold is crossed | ❌ | Network calls require JS |
| Animate the spinner once loading begins | ✅ with `@keyframes` | — |
| Snap the indicator back to zero on cancel | ⚠️ | CSS `transition` can animate back, but JS must remove the inline transform |
| Announce "refreshing" to screen readers | ❌ | `aria-live` toggling requires JS |

**Conclusion:** CSS owns the spinner animation, the rubber-band transition, and suppressing the browser's native overscroll. JS owns every piece of gesture logic, state, and side-effect orchestration.

---

## HTML & CSS Foundation

### The Structure

```html
<!--
  .ptr-container  — the scrollable content wrapper
  .ptr-indicator  — the pull indicator that stretches from the top edge
  .ptr-spinner    — the loading spinner inside the indicator
-->
<div class="ptr-root" role="region" aria-label="Pull to refresh">

  <!-- The indicator lives ABOVE the scrollable content -->
  <div
    class="ptr-indicator"
    aria-hidden="true"
    aria-live="polite"
    aria-atomic="true"
  >
    <div class="ptr-spinner" aria-hidden="true"></div>
  </div>

  <!-- The scrollable content container -->
  <div class="ptr-container" tabindex="0">
    <!-- your list / feed / content here -->
  </div>

</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --ptr-indicator-height: 64px;
  --ptr-threshold: 80px;         /* JS reads this too — keep in sync */
  --ptr-spinner-size: 28px;
  --ptr-spinner-stroke: 3px;
  --ptr-color: #0066cc;
  --ptr-transition: 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

/* ─── Root wrapper — establishes stacking context ─── */
.ptr-root {
  position: relative;
  overflow: hidden;              /* clips the indicator when retracted */
  height: 100%;
}

/* ─── Scrollable container ─── */
.ptr-container {
  height: 100%;
  overflow-y: auto;
  overflow-x: hidden;

  /*
    KEY: overscroll-behavior: contain stops two things:
      1. The browser's own pull-to-refresh (Chrome Android, Safari)
      2. Scroll chaining to a parent element
    Without this, the browser fires its own PTR before JS can intercept.
  */
  overscroll-behavior-y: contain;

  /* Smooth momentum scrolling on iOS */
  -webkit-overflow-scrolling: touch;

  /*
    will-change: transform tells the compositor to keep this on its own
    layer, so translating it during the pull gesture does not repaint.
  */
  will-change: transform;
  transition: transform var(--ptr-transition);
}

/*
  When actively pulling, JS sets an inline transform on .ptr-container.
  Remove the transition during active drag so it follows the finger exactly.
  JS adds/removes this class.
*/
.ptr-container.is-pulling {
  transition: none;
}

/* ─── Pull indicator ─── */
.ptr-indicator {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: var(--ptr-indicator-height);
  display: flex;
  align-items: center;
  justify-content: center;

  /*
    Start above the viewport. JS pushes it down by translating
    .ptr-container downward — the indicator becomes visible as the
    container moves away from the top edge.
  */
  transform: translateY(calc(-1 * var(--ptr-indicator-height)));
  pointer-events: none;
}

/* ─── Loading spinner ─── */
.ptr-spinner {
  width: var(--ptr-spinner-size);
  height: var(--ptr-spinner-size);
  border: var(--ptr-spinner-stroke) solid color-mix(in srgb, var(--ptr-color) 25%, transparent);
  border-top-color: var(--ptr-color);
  border-radius: 50%;

  /*
    The spin animation is always declared here but only plays when
    JS adds .is-loading. This keeps animation-play-state as the
    single toggle rather than adding/removing the whole animation.
  */
  animation: ptr-spin 0.7s linear infinite;
  animation-play-state: paused;

  /*
    During the pull phase, opacity tracks pull progress via a CSS
    custom property set by JS: opacity = clamp(0, pullRatio, 1)
  */
  opacity: var(--ptr-pull-ratio, 0);
  transform: rotate(calc(var(--ptr-pull-ratio, 0) * 270deg));
  transition: opacity 0.2s, transform 0.2s;
}

/* When loading is active: spin continuously, fully opaque */
.ptr-root.is-loading .ptr-spinner {
  animation-play-state: running;
  opacity: 1;
  transform: none;
  transition: none;
}

@keyframes ptr-spin {
  to { transform: rotate(360deg); }
}

/* ─── Snap-back: when loading completes, JS removes is-loading.
   .ptr-container already has its transition, so it smoothly
   returns to translateY(0). ─── */

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .ptr-container {
    transition: none;
  }
  .ptr-spinner {
    animation: none;
    /* Use opacity pulse instead of spin */
    animation: ptr-pulse 1.2s ease-in-out infinite;
    animation-play-state: paused;
  }
  .ptr-root.is-loading .ptr-spinner {
    animation-play-state: running;
  }
  @keyframes ptr-pulse {
    0%, 100% { opacity: 0.3; }
    50%       { opacity: 1;   }
  }
}
```

**What CSS owns:** overscroll suppression, spinner keyframe animation, the transition that snaps the container back to rest, rubber-band-feel on release via the cubic-bezier easing, and the reduced-motion fallback.

**What CSS cannot own:** detecting when `scrollTop === 0`, computing pull distance, applying rubber-band math, triggering the fetch, and updating the `--ptr-pull-ratio` custom property.

---

## React Implementation

### The Hook

```tsx
// usePullToRefresh.ts
import { useRef, useCallback, useEffect } from 'react';

interface Options {
  onRefresh: () => Promise<void>;
  threshold?: number;     // px before trigger fires, default 80
  maxPull?: number;       // px cap on visible pull distance, default 120
  resistance?: number;    // rubber-band divisor 1–4, default 2.5
}

interface PullToRefreshState {
  containerRef: React.RefObject<HTMLDivElement>;
  rootRef: React.RefObject<HTMLDivElement>;
}

export function usePullToRefresh({
  onRefresh,
  threshold = 80,
  maxPull   = 120,
  resistance = 2.5,
}: Options): PullToRefreshState {
  const containerRef = useRef<HTMLDivElement>(null);
  const rootRef      = useRef<HTMLDivElement>(null);

  // Mutable refs — avoids stale closures without triggering re-renders
  const startY      = useRef(0);
  const currentPull = useRef(0);
  const isPulling   = useRef(false);
  const isLoading   = useRef(false);

  const applyPull = useCallback((distance: number) => {
    const container = containerRef.current;
    const root      = rootRef.current;
    if (!container || !root) return;

    // Rubber-band easing: resistance makes the element move less than the finger.
    // Using sqrt gives a natural "heavy object" feel — fast at first, slows as
    // you pull further. Capped at maxPull so content never disappears off-screen.
    const rubberBand = Math.min(Math.sqrt(distance * resistance) * 10, maxPull);

    container.style.transform = `translateY(${rubberBand}px)`;

    // Drive the spinner opacity and pre-rotation via a CSS custom property.
    // This lets CSS handle the visual interpolation while JS only sets the ratio.
    const ratio = Math.min(rubberBand / threshold, 1);
    root.style.setProperty('--ptr-pull-ratio', String(ratio));
  }, [threshold, maxPull, resistance]);

  const reset = useCallback(() => {
    const container = containerRef.current;
    const root      = rootRef.current;
    if (!container || !root) return;

    container.classList.remove('is-pulling');
    container.style.transform = '';
    root.style.removeProperty('--ptr-pull-ratio');
    isPulling.current = false;
    currentPull.current = 0;
  }, []);

  const triggerRefresh = useCallback(async () => {
    const root      = rootRef.current;
    const container = containerRef.current;
    if (!root || !container || isLoading.current) return;

    isLoading.current = true;
    root.classList.add('is-loading');

    // Snap the container to the indicator height so the spinner stays visible
    container.classList.remove('is-pulling');
    container.style.transform = `translateY(64px)`;   // matches --ptr-indicator-height

    // Announce to screen readers
    root.setAttribute('aria-busy', 'true');

    try {
      await onRefresh();
    } finally {
      isLoading.current = false;
      root.classList.remove('is-loading');
      root.setAttribute('aria-busy', 'false');
      reset();
    }
  }, [onRefresh, reset]);

  useEffect(() => {
    const container = containerRef.current;
    if (!container) return;

    // ── Touch handlers ──

    const onTouchStart = (e: TouchEvent) => {
      // Only activate when already scrolled to the very top
      if (container.scrollTop > 0) return;
      if (isLoading.current) return;

      startY.current = e.touches[0].clientY;
      isPulling.current = true;
      container.classList.add('is-pulling');
    };

    const onTouchMove = (e: TouchEvent) => {
      if (!isPulling.current) return;

      const dy = e.touches[0].clientY - startY.current;

      // Ignore upward swipes and horizontal-dominant swipes
      if (dy <= 0) {
        reset();
        return;
      }

      // Prevent the page from scrolling during a downward pull from top
      // NOTE: this listener must NOT be passive for preventDefault() to work.
      e.preventDefault();

      currentPull.current = dy;
      applyPull(dy);
    };

    const onTouchEnd = () => {
      if (!isPulling.current) return;

      if (currentPull.current >= threshold) {
        triggerRefresh();
      } else {
        reset();
      }
    };

    // ── Pointer Events fallback (stylus, mouse on desktop, future inputs) ──

    const onPointerDown = (e: PointerEvent) => {
      if (e.pointerType === 'touch') return;  // touch is handled above
      if (container.scrollTop > 0) return;
      if (isLoading.current) return;

      startY.current = e.clientY;
      isPulling.current = true;
      container.classList.add('is-pulling');
      container.setPointerCapture(e.pointerId);
    };

    const onPointerMove = (e: PointerEvent) => {
      if (e.pointerType === 'touch') return;
      if (!isPulling.current) return;

      const dy = e.clientY - startY.current;
      if (dy <= 0) { reset(); return; }

      currentPull.current = dy;
      applyPull(dy);
    };

    const onPointerUp = (e: PointerEvent) => {
      if (e.pointerType === 'touch') return;
      if (!isPulling.current) return;

      if (currentPull.current >= threshold) {
        triggerRefresh();
      } else {
        reset();
      }
    };

    /*
      IMPORTANT: touchmove must be non-passive so that e.preventDefault()
      works. The browser defaults touchmove to passive on the root scroller
      for performance. We attach to the container element (not window/document)
      with { passive: false } to override this only where needed.
    */
    container.addEventListener('touchstart',  onTouchStart,  { passive: true  });
    container.addEventListener('touchmove',   onTouchMove,   { passive: false });
    container.addEventListener('touchend',    onTouchEnd,    { passive: true  });
    container.addEventListener('pointerdown', onPointerDown);
    container.addEventListener('pointermove', onPointerMove);
    container.addEventListener('pointerup',   onPointerUp);

    return () => {
      container.removeEventListener('touchstart',  onTouchStart);
      container.removeEventListener('touchmove',   onTouchMove);
      container.removeEventListener('touchend',    onTouchEnd);
      container.removeEventListener('pointerdown', onPointerDown);
      container.removeEventListener('pointermove', onPointerMove);
      container.removeEventListener('pointerup',   onPointerUp);
    };
  }, [applyPull, reset, threshold, triggerRefresh]);

  return { containerRef, rootRef };
}
```

### The Component

```tsx
// PullToRefresh.tsx
import { useCallback, useState } from 'react';
import { usePullToRefresh } from './usePullToRefresh';

interface Props {
  children: React.ReactNode;
  onRefresh: () => Promise<void>;
  threshold?: number;
}

export function PullToRefresh({ children, onRefresh, threshold = 80 }: Props) {
  const { containerRef, rootRef } = usePullToRefresh({ onRefresh, threshold });

  return (
    <div
      ref={rootRef}
      className="ptr-root"
      role="region"
      aria-label="Pull to refresh"
    >
      {/* Indicator — visually hidden until pulled */}
      <div className="ptr-indicator" aria-hidden="true">
        <div className="ptr-spinner" />
      </div>

      {/* Scrollable content */}
      <div ref={containerRef} className="ptr-container">
        {children}
      </div>
    </div>
  );
}

/* ── Usage example ── */
function FeedPage() {
  const [items, setItems] = useState<string[]>(['Post 1', 'Post 2', 'Post 3']);

  const handleRefresh = useCallback(async () => {
    // Simulate network fetch
    await new Promise<void>(resolve => setTimeout(resolve, 1500));
    setItems(prev => [`New post at ${new Date().toLocaleTimeString()}`, ...prev]);
  }, []);

  return (
    <PullToRefresh onRefresh={handleRefresh}>
      <ul>
        {items.map((item, i) => (
          <li key={i} style={{ padding: '1rem', borderBottom: '1px solid #eee' }}>
            {item}
          </li>
        ))}
      </ul>
    </PullToRefresh>
  );
}
```

---

## Angular Implementation

### Service

```typescript
// pull-to-refresh.service.ts
import { Injectable, signal, computed } from '@angular/core';

export type PtrPhase = 'idle' | 'pulling' | 'loading' | 'complete';

@Injectable({ providedIn: 'root' })
export class PullToRefreshService {
  private _phase    = signal<PtrPhase>('idle');
  private _pullRatio = signal(0);   // 0–1, drives CSS --ptr-pull-ratio

  readonly phase     = this._phase.asReadonly();
  readonly pullRatio = this._pullRatio.asReadonly();
  readonly isLoading = computed(() => this._phase() === 'loading');

  setPulling(ratio: number) {
    this._phase.set('pulling');
    this._pullRatio.set(Math.min(ratio, 1));
  }

  async startLoading(onRefresh: () => Promise<void>) {
    this._phase.set('loading');
    this._pullRatio.set(1);
    try {
      await onRefresh();
      this._phase.set('complete');
    } finally {
      // Brief 'complete' phase so the spinner visually finishes its rotation
      setTimeout(() => this.reset(), 300);
    }
  }

  reset() {
    this._phase.set('idle');
    this._pullRatio.set(0);
  }
}
```

### Component

```typescript
// pull-to-refresh.component.ts
import {
  Component, Input, OnDestroy, AfterViewInit,
  ElementRef, ViewChild, inject, effect
} from '@angular/core';
import { NgClass, NgStyle } from '@angular/common';
import { PullToRefreshService } from './pull-to-refresh.service';

@Component({
  selector: 'app-pull-to-refresh',
  standalone: true,
  imports: [NgClass, NgStyle],
  template: `
    <div
      #root
      class="ptr-root"
      role="region"
      aria-label="Pull to refresh"
      [attr.aria-busy]="ptrService.isLoading()"
      [style.--ptr-pull-ratio]="ptrService.pullRatio()"
    >
      <!-- Indicator -->
      <div class="ptr-indicator" aria-hidden="true">
        <div
          class="ptr-spinner"
          [class.is-loading]="ptrService.isLoading()"
        ></div>
      </div>

      <!-- Scrollable container -->
      <div
        #container
        class="ptr-container"
        [ngClass]="{
          'is-pulling': ptrService.phase() === 'pulling',
          'is-loading': ptrService.isLoading()
        }"
        [ngStyle]="containerTransform"
      >
        <ng-content />
      </div>
    </div>
  `,
})
export class PullToRefreshComponent implements AfterViewInit, OnDestroy {
  @Input({ required: true }) onRefresh!: () => Promise<void>;
  @Input() threshold  = 80;
  @Input() maxPull    = 120;
  @Input() resistance = 2.5;

  @ViewChild('container') containerEl!: ElementRef<HTMLDivElement>;

  ptrService = inject(PullToRefreshService);

  private startY       = 0;
  private currentPull  = 0;
  private pulling      = false;
  private listeners: Array<() => void> = [];

  // Derived style for the container transform
  get containerTransform(): Record<string, string> {
    const phase = this.ptrService.phase();
    if (phase === 'loading' || phase === 'complete') {
      return { transform: 'translateY(64px)', transition: '' };
    }
    if (phase === 'pulling') {
      return { transform: `translateY(${this.rubberBand}px)`, transition: 'none' };
    }
    return { transform: 'translateY(0)' };
  }

  private get rubberBand(): number {
    return Math.min(Math.sqrt(this.currentPull * this.resistance) * 10, this.maxPull);
  }

  ngAfterViewInit() {
    const el = this.containerEl.nativeElement;

    const onTouchStart = (e: TouchEvent) => {
      if (el.scrollTop > 0 || this.ptrService.isLoading()) return;
      this.startY = e.touches[0].clientY;
      this.pulling = true;
    };

    const onTouchMove = (e: TouchEvent) => {
      if (!this.pulling) return;
      const dy = e.touches[0].clientY - this.startY;
      if (dy <= 0) { this.cancelPull(); return; }
      e.preventDefault();
      this.currentPull = dy;
      this.ptrService.setPulling(this.rubberBand / this.threshold);
    };

    const onTouchEnd = () => {
      if (!this.pulling) return;
      this.pulling = false;
      if (this.currentPull >= this.threshold) {
        this.ptrService.startLoading(this.onRefresh);
      } else {
        this.cancelPull();
      }
      this.currentPull = 0;
    };

    const onPointerDown = (e: PointerEvent) => {
      if (e.pointerType === 'touch') return;
      if (el.scrollTop > 0 || this.ptrService.isLoading()) return;
      this.startY = e.clientY;
      this.pulling = true;
      el.setPointerCapture(e.pointerId);
    };

    const onPointerMove = (e: PointerEvent) => {
      if (e.pointerType === 'touch' || !this.pulling) return;
      const dy = e.clientY - this.startY;
      if (dy <= 0) { this.cancelPull(); return; }
      this.currentPull = dy;
      this.ptrService.setPulling(this.rubberBand / this.threshold);
    };

    const onPointerUp = (e: PointerEvent) => {
      if (e.pointerType === 'touch' || !this.pulling) return;
      this.pulling = false;
      if (this.currentPull >= this.threshold) {
        this.ptrService.startLoading(this.onRefresh);
      } else {
        this.cancelPull();
      }
      this.currentPull = 0;
    };

    el.addEventListener('touchstart',  onTouchStart,  { passive: true  });
    el.addEventListener('touchmove',   onTouchMove,   { passive: false });
    el.addEventListener('touchend',    onTouchEnd,    { passive: true  });
    el.addEventListener('pointerdown', onPointerDown);
    el.addEventListener('pointermove', onPointerMove);
    el.addEventListener('pointerup',   onPointerUp);

    this.listeners.push(
      () => el.removeEventListener('touchstart',  onTouchStart),
      () => el.removeEventListener('touchmove',   onTouchMove),
      () => el.removeEventListener('touchend',    onTouchEnd),
      () => el.removeEventListener('pointerdown', onPointerDown),
      () => el.removeEventListener('pointermove', onPointerMove),
      () => el.removeEventListener('pointerup',   onPointerUp),
    );
  }

  private cancelPull() {
    this.pulling = false;
    this.currentPull = 0;
    this.ptrService.reset();
  }

  ngOnDestroy() {
    this.listeners.forEach(fn => fn());
    this.ptrService.reset();
  }
}
```

### Usage in a Feed Component

```typescript
// feed.component.ts
import { Component, signal } from '@angular/core';
import { PullToRefreshComponent } from './pull-to-refresh.component';

@Component({
  selector: 'app-feed',
  standalone: true,
  imports: [PullToRefreshComponent],
  template: `
    <app-pull-to-refresh [onRefresh]="handleRefresh">
      <ul>
        @for (item of items(); track item.id) {
          <li style="padding: 1rem; border-bottom: 1px solid #eee">
            {{ item.text }}
          </li>
        }
      </ul>
    </app-pull-to-refresh>
  `,
})
export class FeedComponent {
  items = signal([
    { id: 1, text: 'Post 1' },
    { id: 2, text: 'Post 2' },
  ]);

  handleRefresh = async () => {
    await new Promise<void>(resolve => setTimeout(resolve, 1200));
    const next = { id: Date.now(), text: `Refreshed at ${new Date().toLocaleTimeString()}` };
    this.items.update(prev => [next, ...prev]);
  };
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Region is labeled | `role="region"` + `aria-label="Pull to refresh"` on root |
| Loading state announced | `aria-busy="true"` set on root when fetch is in flight |
| Indicator is hidden from assistive tech during pull | `aria-hidden="true"` on `.ptr-indicator` at all times (visual-only) |
| Refresh completion announced | `aria-busy="false"` after fetch resolves; `aria-live` region can add a "Content updated" message |
| Spinner respects reduced motion | `prefers-reduced-motion` switches spin to opacity pulse via `@media` |
| Minimum touch target for cancel | Not applicable — gesture-based; no button to size |
| Keyboard alternative | Add a visible "Refresh" button in the header — PTR is a touch enhancement, not the only path |
| No content hidden during loading | The existing list remains visible while spinner shows above it |

---

## Production Pitfalls

**1. `touchmove` passive listener violation crashes the gesture**
Chrome logs a warning and ignores `e.preventDefault()` if the `touchmove` listener is registered as `passive: true` (the browser default for root-level listeners). Always attach to the scroll container element — not `document` or `window` — and explicitly pass `{ passive: false }`. This limits the performance impact to the specific element rather than blocking the entire page scroll thread.

**2. Pull fires mid-scroll instead of only at `scrollTop === 0`**
A common bug is checking `scrollTop` at `touchmove` time instead of `touchstart`. By `touchmove` the user may have already scrolled up from a non-zero position. Gate the pull activation at `touchstart`: if `scrollTop > 0` at that moment, do not set `isPulling = true`. If scroll is at zero, the `touchmove` phase can confidently intercept downward movement.

**3. Horizontal swipe accidentally triggers the pull**
On wide content (horizontal scroll, carousels), a horizontal swipe has a small positive dy component. Fix: compute both `dx` and `dy` at the first `touchmove` event after `touchstart`. If `Math.abs(dx) > Math.abs(dy)`, cancel the pull — the user intends a horizontal gesture, not a vertical pull.

**4. iOS Safari ignores `overscroll-behavior` on the `body` or `html` element**
`overscroll-behavior-y: contain` must be set on the *scrollable container element* itself — the element with `overflow-y: auto` — not on `body` or `html`. iOS Safari only respects this property on the element doing the scrolling. Setting it on `body` stops body scroll chaining but does nothing for nested scrollers.

**5. The spinner stays visible if the component unmounts mid-fetch**
If the user navigates away while a refresh is in flight, the `onRefresh` promise resolves after the component is gone. In React, guard the `finally` block with a `useRef` mounted flag. In Angular, store a destroyed flag in `ngOnDestroy`. Without this, calling `reset()` on an unmounted component either throws a React state-update warning or triggers Angular's `ExpressionChangedAfterItHasBeenChecked` error.

**6. Rubber-band resistance that feels wrong**
A linear `distance / resistance` formula moves the element at a fixed fraction of finger speed — it feels mechanical and cheap. A `sqrt`-based formula (`Math.sqrt(distance * k) * 10`) starts fast and decelerates, mimicking the feel of stretching a physical material. Test on a real device: simulator momentum is unreliable. Values for `resistance` between `2` and `3` work well; below `1.5` the pull feels too loose, above `4` it feels unresponsive.

**7. The snap-back transition plays during the active pull**
Adding `transition: transform 0.3s` to `.ptr-container` globally causes the container to rubber-band lag behind the finger during the pull. The fix is the `.is-pulling` class that removes `transition: none` while the gesture is active. Only restore the transition for the snap-back phase (remove `.is-pulling` before triggering the snap).

---

## Interview Angle

**Q: "How would you implement pull-to-refresh without using a library?"**

Strong answer covers four points:
1. **Gesture detection gate** — PTR only activates when `scrollTop === 0` at `touchstart`. This is the most commonly missed requirement. Any other starting position means the user is scrolling, not pulling.
2. **CSS prerequisites** — `overscroll-behavior-y: contain` on the scroll container suppresses the browser's native PTR before your JS runs. Without it, both handlers fire and you get a double-refresh.
3. **Rubber-band easing** — A `sqrt`-based formula (`Math.sqrt(dy * k) * 10`) makes the pull feel physical. The linear alternative feels like dragging a div, not pulling against resistance.
4. **Passive listener strategy** — `touchstart` and `touchend` are passive (correct for performance). `touchmove` must be `{ passive: false }` so `e.preventDefault()` can block native scroll during the downward pull. Attach to the container element, not `window`.

**Follow-up: "What is the difference between using Touch Events and Pointer Events for this?"**

Touch Events (`touchstart`/`touchmove`/`touchend`) are touch-only. They give you a `touches` list, which is useful when you need to detect multi-touch (pinch during a pull). Pointer Events (`pointerdown`/`pointermove`/`pointerup`) are a unified model covering mouse, touch, and stylus — you check `e.pointerType` to differentiate. For PTR, Touch Events handle the primary mobile use case. Adding a Pointer Events path alongside it covers stylus and lets you test the gesture with a mouse on desktop. `setPointerCapture` on the container ensures the `pointermove` and `pointerup` events are always delivered to the same element even if the pointer moves outside it — there is no equivalent in Touch Events since `touchmove` already fires on the originating element.
