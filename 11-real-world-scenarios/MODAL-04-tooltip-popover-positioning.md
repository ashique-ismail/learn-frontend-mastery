# Tooltip & Popover Positioning (Floating UI)

## The Idea

**In plain English:** You have a small element — a button, an icon, a form field — and you need to display a floating box (tooltip, popover, dropdown) near it. That box must stay visible as the user scrolls, must not clip outside the viewport, and must correctly point its arrow back to the anchor element. The challenge is that the "best" position depends on how much room is available in all four directions, and that changes constantly as the page scrolls and the window resizes.

**Real-world analogy:** Think of a food truck parked at an intersection.

- The truck (floating element) always wants to park directly in front of its counter window (the anchor element).
- But if there is a wall on that side (viewport edge), the truck shifts to whichever side has room — it **flips** to the opposite side of the intersection.
- If even the flipped side is tight, the truck shuffles left or right along the kerb to stay fully visible — that is **shift**.
- The arrow on the menu board (the arrow element) always swings to point back at the counter window regardless of where the truck ends up.
- When the truck moves to a new parking spot after the lunch rush (scroll/resize), it recalculates the best position from scratch.

The key insight: position is not a CSS property you set once — it is a computed value you recalculate every time the environment changes.

---

## Learning Objectives

- Understand why CSS `position: absolute/fixed` alone cannot solve floating UI placement
- Use Floating UI's `computePosition` with `flip`, `shift`, `offset`, and `arrow` middleware
- Implement `autoUpdate` to reposition on scroll, resize, and ancestor mutations
- Position the arrow element correctly using the middleware-provided data
- Replicate the same behavior in Angular using the CDK Overlay with `FlexibleConnectedPositionStrategy`
- Apply correct ARIA semantics for tooltips (`role="tooltip"`, `aria-describedby`) and popovers (`popover` API, `aria-expanded`)

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Place element near anchor | ✅ `position: absolute` inside same container | — |
| Flip to opposite side when near viewport edge | ❌ | CSS has no concept of overflow detection |
| Shift along an axis to keep fully in viewport | ❌ | CSS cannot read available space at runtime |
| Keep position updated as page scrolls | ❌ | No CSS scroll listener |
| Keep position updated on window resize | ❌ | No CSS resize listener |
| Position arrow to always point at anchor after flip/shift | ❌ | Arrow offset depends on computed final position |
| Anchor to an element inside a scrolling container | ❌ | `position: fixed` ignores scroll containers; `absolute` clips to them |
| Detect if a containing block has `transform` applied | ❌ | `position: fixed` breaks inside transformed ancestors |

**Conclusion:** CSS owns the visual style of the floating element (colors, shadow, border-radius, arrow shape). JS — specifically a layout engine like Floating UI — owns the position computation.

---

## HTML & CSS Foundation

### The Structure

```html
<!-- The anchor element -->
<button
  id="my-button"
  type="button"
  aria-describedby="my-tooltip"
>
  Hover me
</button>

<!-- Tooltip (initially hidden) -->
<div
  id="my-tooltip"
  role="tooltip"
  class="tooltip"
  data-placement="top"
>
  This action saves your changes
  <div class="tooltip-arrow" data-floating-arrow></div>
</div>

<!-- Popover (HTML Popover API) -->
<button
  type="button"
  popovertarget="my-popover"
  aria-expanded="false"
>
  Open popover
</button>

<div
  id="my-popover"
  popover
  class="popover"
  role="dialog"
  aria-label="Filter options"
>
  <div class="popover-arrow" data-floating-arrow></div>
  <p>Popover content here</p>
</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --tooltip-bg: #1a1a1a;
  --tooltip-color: #fff;
  --tooltip-radius: 4px;
  --tooltip-padding: 6px 10px;
  --tooltip-font: 0.8125rem;       /* 13px */
  --tooltip-shadow: 0 2px 8px rgba(0,0,0,0.25);
  --tooltip-arrow-size: 8px;

  --popover-bg: #fff;
  --popover-shadow: 0 4px 24px rgba(0,0,0,0.15);
  --popover-radius: 8px;
  --popover-arrow-size: 10px;
}

/* ─── Tooltip base ─── */
.tooltip {
  /* Position set entirely by Floating UI — never hardcode top/left here */
  position: absolute;
  top: 0;
  left: 0;
  width: max-content;
  max-width: 280px;

  background: var(--tooltip-bg);
  color: var(--tooltip-color);
  padding: var(--tooltip-padding);
  border-radius: var(--tooltip-radius);
  font-size: var(--tooltip-font);
  line-height: 1.4;
  box-shadow: var(--tooltip-shadow);
  pointer-events: none;          /* tooltips never receive mouse events */
  white-space: nowrap;

  /* Hidden state */
  opacity: 0;
  visibility: hidden;
  transition: opacity 0.15s, visibility 0.15s;
}

.tooltip.is-visible {
  opacity: 1;
  visibility: visible;
}

/* ─── Tooltip arrow ─── */
/*
  The arrow is a rotated square. Floating UI places it via `top`/`left`
  inline styles and we rotate it based on the current placement direction.
*/
.tooltip-arrow,
.popover-arrow {
  position: absolute;
  width: var(--tooltip-arrow-size);
  height: var(--tooltip-arrow-size);
  background: inherit;
  transform: rotate(45deg);
  pointer-events: none;
}

/* Arrow positioning relative to the floating element's edge */
[data-placement^="top"]    .tooltip-arrow { bottom: calc(var(--tooltip-arrow-size) / -2); }
[data-placement^="bottom"] .tooltip-arrow { top:    calc(var(--tooltip-arrow-size) / -2); }
[data-placement^="left"]   .tooltip-arrow { right:  calc(var(--tooltip-arrow-size) / -2); }
[data-placement^="right"]  .tooltip-arrow { left:   calc(var(--tooltip-arrow-size) / -2); }

/* ─── Popover base ─── */
.popover {
  position: absolute;
  top: 0;
  left: 0;
  width: max-content;
  max-width: 320px;

  background: var(--popover-bg);
  border-radius: var(--popover-radius);
  box-shadow: var(--popover-shadow);
  padding: 1rem;
  border: 1px solid rgba(0,0,0,0.08);

  /* Native popover resets */
  margin: 0;
  inset: unset;     /* override UA stylesheet */
}

.popover-arrow {
  --popover-arrow-size: 10px;
  width: var(--popover-arrow-size);
  height: var(--popover-arrow-size);
  background: var(--popover-bg);
  border-left: 1px solid rgba(0,0,0,0.08);
  border-top: 1px solid rgba(0,0,0,0.08);
}

[data-placement^="top"]    .popover-arrow { bottom: calc(var(--popover-arrow-size) / -2); transform: rotate(225deg); }
[data-placement^="bottom"] .popover-arrow { top:    calc(var(--popover-arrow-size) / -2); transform: rotate(45deg);  }
[data-placement^="left"]   .popover-arrow { right:  calc(var(--popover-arrow-size) / -2); transform: rotate(135deg); }
[data-placement^="right"]  .popover-arrow { left:   calc(var(--popover-arrow-size) / -2); transform: rotate(315deg); }

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .tooltip {
    transition: none;
  }
}
```

**What CSS owns:** visual style (colors, border-radius, shadow, arrow shape), show/hide transition, pointer-events suppression on tooltips, placement-aware arrow rotation via `data-placement`.

**What CSS cannot own:** the actual `top`/`left` values, which side to place on, how far to shift, and the arrow's cross-axis offset — all computed by Floating UI at runtime.

---

## React Implementation

### Core Hook — `useFloating`

```tsx
// useFloating.ts
import { useState, useRef, useCallback, useEffect } from 'react';
import {
  computePosition,
  flip,
  shift,
  offset,
  arrow,
  autoUpdate,
  type Placement,
  type ComputePositionReturn,
} from '@floating-ui/dom';

interface UseFloatingOptions {
  placement?: Placement;
  offsetPx?: number;
  enabled?: boolean;
}

interface UseFloatingReturn {
  refs: {
    setReference: (el: Element | null) => void;
    setFloating: (el: HTMLElement | null) => void;
    setArrow: (el: HTMLElement | null) => void;
  };
  floatingStyles: React.CSSProperties;
  arrowStyles: React.CSSProperties;
  currentPlacement: Placement;
  update: () => void;
}

export function useFloating({
  placement = 'top',
  offsetPx = 8,
  enabled = true,
}: UseFloatingOptions = {}): UseFloatingReturn {
  const referenceRef = useRef<Element | null>(null);
  const floatingRef  = useRef<HTMLElement | null>(null);
  const arrowRef     = useRef<HTMLElement | null>(null);
  const cleanupRef   = useRef<(() => void) | null>(null);

  const [floatingStyles, setFloatingStyles] = useState<React.CSSProperties>({
    position: 'absolute',
    top: 0,
    left: 0,
  });
  const [arrowStyles, setArrowStyles]       = useState<React.CSSProperties>({});
  const [currentPlacement, setCurrentPlacement] = useState<Placement>(placement);

  const applyPosition = useCallback(
    (result: ComputePositionReturn) => {
      const { x, y, placement: finalPlacement, middlewareData } = result;

      setFloatingStyles({
        position: 'absolute',
        top: 0,
        left: 0,
        transform: `translate(${Math.round(x)}px, ${Math.round(y)}px)`,
      });

      setCurrentPlacement(finalPlacement);

      // Arrow positioning
      const arrowData = middlewareData.arrow;
      if (arrowData) {
        const { x: ax, y: ay } = arrowData;
        setArrowStyles({
          left: ax != null ? `${Math.round(ax)}px` : '',
          top:  ay != null ? `${Math.round(ay)}px` : '',
        });
      }
    },
    []
  );

  const update = useCallback(() => {
    const reference = referenceRef.current;
    const floating  = floatingRef.current;
    if (!reference || !floating || !enabled) return;

    computePosition(reference, floating, {
      placement,
      middleware: [
        offset(offsetPx),
        flip({
          fallbackAxisSideDirection: 'start',  // prefer start-aligned fallback
        }),
        shift({ padding: 8 }),                 // 8px from viewport edge minimum
        arrow({ element: arrowRef }),
      ],
    }).then(applyPosition);
  }, [placement, offsetPx, enabled, applyPosition]);

  // Wire up autoUpdate (scroll, resize, ancestor mutations)
  useEffect(() => {
    const reference = referenceRef.current;
    const floating  = floatingRef.current;

    cleanupRef.current?.();

    if (!reference || !floating || !enabled) return;

    cleanupRef.current = autoUpdate(reference, floating, update, {
      animationFrame: false,  // true only for animated anchors
      ancestorScroll: true,
      ancestorResize: true,
      elementResize: true,
    });

    return () => {
      cleanupRef.current?.();
      cleanupRef.current = null;
    };
  }, [update, enabled]);

  const setReference = useCallback((el: Element | null) => {
    referenceRef.current = el;
    update();
  }, [update]);

  const setFloating = useCallback((el: HTMLElement | null) => {
    floatingRef.current = el;
    update();
  }, [update]);

  const setArrow = useCallback((el: HTMLElement | null) => {
    arrowRef.current = el;
  }, []);

  return {
    refs: { setReference, setFloating, setArrow },
    floatingStyles,
    arrowStyles,
    currentPlacement,
    update,
  };
}
```

### Tooltip Component

```tsx
// Tooltip.tsx
import { useState, useId, type ReactNode } from 'react';
import { useFloating } from './useFloating';

interface TooltipProps {
  content: string;
  placement?: 'top' | 'bottom' | 'left' | 'right';
  children: ReactNode;
}

export function Tooltip({ content, placement = 'top', children }: TooltipProps) {
  const [isVisible, setIsVisible] = useState(false);
  const tooltipId = useId();

  const { refs, floatingStyles, arrowStyles, currentPlacement } = useFloating({
    placement,
    offsetPx: 10,
    enabled: isVisible,
  });

  const show = () => setIsVisible(true);
  const hide = () => setIsVisible(false);

  return (
    <>
      {/* Anchor: clone children to inject ref + ARIA */}
      <span
        ref={refs.setReference}
        aria-describedby={isVisible ? tooltipId : undefined}
        onMouseEnter={show}
        onMouseLeave={hide}
        onFocus={show}
        onBlur={hide}
        style={{ display: 'inline-block' }}
      >
        {children}
      </span>

      {/* Tooltip */}
      <div
        id={tooltipId}
        ref={refs.setFloating}
        role="tooltip"
        className={`tooltip ${isVisible ? 'is-visible' : ''}`}
        data-placement={currentPlacement}
        style={floatingStyles}
        aria-hidden={!isVisible}
      >
        {content}
        <div
          ref={refs.setArrow}
          className="tooltip-arrow"
          style={arrowStyles}
        />
      </div>
    </>
  );
}

// Usage:
// <Tooltip content="Save your changes" placement="top">
//   <button>Save</button>
// </Tooltip>
```

### Popover Component

```tsx
// Popover.tsx
import { useState, useRef, useId, type ReactNode } from 'react';
import { useFloating } from './useFloating';

interface PopoverProps {
  trigger: ReactNode;
  content: ReactNode;
  placement?: 'top' | 'bottom' | 'left' | 'right';
  label: string;   // aria-label for the popover dialog
}

export function Popover({ trigger, content, placement = 'bottom', label }: PopoverProps) {
  const [isOpen, setIsOpen] = useState(false);
  const popoverId  = useId();
  const triggerRef = useRef<HTMLButtonElement>(null);

  const { refs, floatingStyles, arrowStyles, currentPlacement } = useFloating({
    placement,
    offsetPx: 12,
    enabled: isOpen,
  });

  const toggle = () => setIsOpen(prev => !prev);

  const close = () => {
    setIsOpen(false);
    triggerRef.current?.focus();   // return focus to trigger on close
  };

  return (
    <>
      <button
        ref={(el) => {
          (triggerRef as React.MutableRefObject<HTMLButtonElement | null>).current = el;
          refs.setReference(el);
        }}
        type="button"
        aria-expanded={isOpen}
        aria-controls={popoverId}
        aria-haspopup="dialog"
        onClick={toggle}
      >
        {trigger}
      </button>

      {isOpen && (
        <div
          id={popoverId}
          ref={refs.setFloating}
          role="dialog"
          aria-label={label}
          className="popover"
          data-placement={currentPlacement}
          style={floatingStyles}
        >
          <div
            ref={refs.setArrow}
            className="popover-arrow"
            style={arrowStyles}
          />
          {content}
          <button
            className="popover-close"
            aria-label="Close"
            onClick={close}
          >
            ×
          </button>
        </div>
      )}
    </>
  );
}
```

---

## Angular Implementation

### Floating UI Service

```typescript
// floating-ui.service.ts
import { Injectable, NgZone } from '@angular/core';
import {
  computePosition,
  flip,
  shift,
  offset,
  arrow,
  autoUpdate,
  type Placement,
  type AutoUpdateOptions,
} from '@floating-ui/dom';

export interface FloatingPosition {
  x: number;
  y: number;
  placement: Placement;
  arrowX?: number;
  arrowY?: number;
}

@Injectable({ providedIn: 'root' })
export class FloatingUiService {
  private readonly zone = new NgZone({});

  /**
   * Compute position once and apply it.
   * Returns the cleanup function from autoUpdate.
   */
  attach(
    reference: Element,
    floating: HTMLElement,
    arrowEl: HTMLElement | null,
    placement: Placement,
    offsetPx: number,
    onUpdate: (pos: FloatingPosition) => void,
    options?: Partial<AutoUpdateOptions>
  ): () => void {
    const compute = () => {
      this.zone.runOutsideAngular(() => {
        computePosition(reference, floating, {
          placement,
          middleware: [
            offset(offsetPx),
            flip({ fallbackAxisSideDirection: 'start' }),
            shift({ padding: 8 }),
            ...(arrowEl ? [arrow({ element: arrowEl })] : []),
          ],
        }).then(({ x, y, placement: finalPlacement, middlewareData }) => {
          this.zone.run(() => {
            onUpdate({
              x: Math.round(x),
              y: Math.round(y),
              placement: finalPlacement,
              arrowX: middlewareData.arrow?.x != null ? Math.round(middlewareData.arrow.x) : undefined,
              arrowY: middlewareData.arrow?.y != null ? Math.round(middlewareData.arrow.y) : undefined,
            });
          });
        });
      });
    };

    return autoUpdate(reference, floating, compute, {
      ancestorScroll: true,
      ancestorResize: true,
      elementResize: true,
      ...options,
    });
  }
}
```

### Tooltip Directive

```typescript
// tooltip.directive.ts
import {
  Directive, Input, OnDestroy, ElementRef,
  HostListener, signal, computed, inject,
  ViewContainerRef, TemplateRef
} from '@angular/core';
import { Overlay, OverlayRef } from '@angular/cdk/overlay';
import { ComponentPortal } from '@angular/cdk/portal';
import { FloatingUiService } from './floating-ui.service';
import { TooltipOverlayComponent } from './tooltip-overlay.component';

@Directive({
  selector: '[appTooltip]',
  standalone: true,
})
export class TooltipDirective implements OnDestroy {
  @Input('appTooltip')       tooltipText  = '';
  @Input() tooltipPlacement  = 'top' as const;

  private el        = inject(ElementRef<HTMLElement>);
  private overlay   = inject(Overlay);
  private floating  = inject(FloatingUiService);

  private overlayRef: OverlayRef | null = null;
  private cleanupFn: (() => void) | null = null;
  private componentRef: ReturnType<typeof ComponentPortal.prototype.attach> | null = null;

  @HostListener('mouseenter')
  @HostListener('focusin')
  show() {
    if (this.overlayRef) return;

    // Create a body-attached overlay (no clipping from scroll containers)
    this.overlayRef = this.overlay.create({
      positionStrategy: this.overlay
        .position()
        .global()
        .top('0')
        .left('0'),
      scrollStrategy: this.overlay.scrollStrategies.reposition(),
    });

    const portal = new ComponentPortal(TooltipOverlayComponent);
    const ref    = this.overlayRef.attach(portal);
    ref.instance.text = this.tooltipText;

    // Hand off to Floating UI for actual positioning
    const floatingEl = ref.location.nativeElement as HTMLElement;
    const arrowEl    = floatingEl.querySelector<HTMLElement>('[data-floating-arrow]');

    this.cleanupFn = this.floating.attach(
      this.el.nativeElement,
      floatingEl,
      arrowEl,
      this.tooltipPlacement,
      10,
      ({ x, y, placement, arrowX, arrowY }) => {
        floatingEl.style.transform = `translate(${x}px, ${y}px)`;
        floatingEl.dataset['placement'] = placement;
        if (arrowEl) {
          if (arrowX != null) arrowEl.style.left = `${arrowX}px`;
          if (arrowY != null) arrowEl.style.top  = `${arrowY}px`;
        }
      }
    );
  }

  @HostListener('mouseleave')
  @HostListener('focusout')
  hide() {
    this.cleanupFn?.();
    this.cleanupFn = null;
    this.overlayRef?.dispose();
    this.overlayRef = null;
  }

  ngOnDestroy() {
    this.hide();
  }
}
```

### Tooltip Overlay Component

```typescript
// tooltip-overlay.component.ts
import { Component, Input } from '@angular/core';

@Component({
  selector: 'app-tooltip-overlay',
  standalone: true,
  template: `
    <div
      role="tooltip"
      class="tooltip is-visible"
    >
      {{ text }}
      <div class="tooltip-arrow" data-floating-arrow></div>
    </div>
  `,
})
export class TooltipOverlayComponent {
  @Input() text = '';
}
```

### Popover Component (CDK Overlay + FlexibleConnectedPositionStrategy)

```typescript
// popover.component.ts
import {
  Component, Input, Output, EventEmitter, OnDestroy,
  ElementRef, ViewChild, TemplateRef, inject, signal
} from '@angular/core';
import {
  Overlay, OverlayRef, OverlayConfig,
  FlexibleConnectedPositionStrategy, ConnectedPosition
} from '@angular/cdk/overlay';
import { TemplatePortal } from '@angular/cdk/portal';
import { ViewContainerRef } from '@angular/core';
import { NgTemplateOutlet } from '@angular/common';

@Component({
  selector: 'app-popover',
  standalone: true,
  imports: [NgTemplateOutlet],
  template: `
    <!-- Trigger -->
    <button
      #triggerEl
      type="button"
      [attr.aria-expanded]="isOpen()"
      [attr.aria-controls]="popoverId"
      aria-haspopup="dialog"
      (click)="toggle()"
    >
      <ng-content select="[trigger]" />
    </button>

    <!-- Popover content template (rendered in CDK overlay) -->
    <ng-template #popoverTpl>
      <div
        [id]="popoverId"
        role="dialog"
        [attr.aria-label]="label"
        class="popover"
        [attr.data-placement]="currentPlacement()"
      >
        <div class="popover-arrow" data-floating-arrow></div>
        <ng-content />
        <button
          class="popover-close"
          aria-label="Close popover"
          (click)="close()"
        >×</button>
      </div>
    </ng-template>
  `,
})
export class PopoverComponent implements OnDestroy {
  @Input() label       = '';
  @Input() placement   = 'bottom' as 'top' | 'bottom' | 'left' | 'right';

  @ViewChild('triggerEl', { static: true }) triggerEl!: ElementRef<HTMLButtonElement>;
  @ViewChild('popoverTpl', { static: true }) popoverTpl!: TemplateRef<void>;

  readonly popoverId      = `popover-${Math.random().toString(36).slice(2)}`;
  readonly isOpen         = signal(false);
  readonly currentPlacement = signal<string>(this.placement);

  private overlay  = inject(Overlay);
  private vcr      = inject(ViewContainerRef);
  private floating = inject(FloatingUiService);

  private overlayRef: OverlayRef | null = null;
  private cleanupFn: (() => void) | null = null;

  // Map our placement strings to CDK's ConnectedPosition
  private get cdkPositions(): ConnectedPosition[] {
    const map: Record<string, ConnectedPosition[]> = {
      bottom: [
        { originX: 'center', originY: 'bottom', overlayX: 'center', overlayY: 'top', offsetY: 12 },
        { originX: 'center', originY: 'top',    overlayX: 'center', overlayY: 'bottom', offsetY: -12 },
      ],
      top: [
        { originX: 'center', originY: 'top',    overlayX: 'center', overlayY: 'bottom', offsetY: -12 },
        { originX: 'center', originY: 'bottom', overlayX: 'center', overlayY: 'top', offsetY: 12 },
      ],
      left: [
        { originX: 'start', originY: 'center', overlayX: 'end', overlayY: 'center', offsetX: -12 },
        { originX: 'end',   originY: 'center', overlayX: 'start', overlayY: 'center', offsetX: 12 },
      ],
      right: [
        { originX: 'end',   originY: 'center', overlayX: 'start', overlayY: 'center', offsetX: 12 },
        { originX: 'start', originY: 'center', overlayX: 'end', overlayY: 'center', offsetX: -12 },
      ],
    };
    return map[this.placement] ?? map['bottom'];
  }

  toggle() {
    this.isOpen() ? this.close() : this.open();
  }

  open() {
    if (this.overlayRef) return;

    const positionStrategy: FlexibleConnectedPositionStrategy = this.overlay
      .position()
      .flexibleConnectedTo(this.triggerEl)
      .withPositions(this.cdkPositions)
      .withPush(true)           // push into viewport if needed (like `shift`)
      .withFlexibleDimensions(false);

    // Track which CDK position was applied
    positionStrategy.positionChanges.subscribe(change => {
      const p = change.connectionPair;
      if (p.overlayY === 'top')    this.currentPlacement.set('bottom');
      else if (p.overlayY === 'bottom') this.currentPlacement.set('top');
      else if (p.overlayX === 'start')  this.currentPlacement.set('right');
      else                              this.currentPlacement.set('left');
    });

    const config: OverlayConfig = {
      positionStrategy,
      scrollStrategy: this.overlay.scrollStrategies.reposition(),
      hasBackdrop: true,
      backdropClass: 'cdk-overlay-transparent-backdrop',
    };

    this.overlayRef = this.overlay.create(config);
    this.overlayRef.attach(new TemplatePortal(this.popoverTpl, this.vcr));
    this.overlayRef.backdropClick().subscribe(() => this.close());

    this.isOpen.set(true);
  }

  close() {
    this.cleanupFn?.();
    this.cleanupFn = null;
    this.overlayRef?.dispose();
    this.overlayRef = null;
    this.isOpen.set(false);
    this.triggerEl.nativeElement.focus();   // return focus to trigger
  }

  ngOnDestroy() {
    this.close();
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Tooltip uses `role="tooltip"` | Set on the floating element |
| Anchor references tooltip with `aria-describedby` | Set to tooltip `id` only while visible; removed when hidden |
| Tooltip is not the only conveyor of information | Color or icon must also indicate the same state |
| Tooltip hidden from AT when not visible | `aria-hidden="true"` while closed; `display: none` or `visibility: hidden` |
| Tooltip appears on keyboard focus, not only hover | `onfocus`/`onblur` event handlers alongside `mouseenter`/`mouseleave` |
| Tooltip not obscured by hover — stays open while pointer is over it | CDK / Floating UI mouse-tracking: `pointer-events: none` on tooltip itself |
| Popover uses `role="dialog"` with `aria-label` | Set on the popover panel |
| Trigger has `aria-expanded` updated on open/close | Bound to `isOpen` signal |
| Trigger has `aria-haspopup="dialog"` | Static attribute |
| Focus moves into popover on open | First focusable element receives focus; or use `cdkTrapFocus` directive |
| Focus returns to trigger on close | Explicitly call `triggerEl.focus()` in close handler |
| Popover closable with Escape key | `HostListener('keydown.escape')` or CDK overlay keyboard handler |
| Arrow element hidden from AT | `aria-hidden="true"` on arrow div |
| Minimum touch target for trigger | 44×44px minimum via CSS |

---

## Production Pitfalls

**1. Using `position: fixed` inside a `transform`ed ancestor**
`position: fixed` does not escape a containing block created by `transform`, `filter`, `perspective`, or `will-change`. The floating element will be positioned relative to the transformed ancestor, not the viewport. Floating UI's `computePosition` handles this by measuring via `getBoundingClientRect` — but you must append the floating element to `document.body` (or an unaffected ancestor), not inline in the component tree. CDK Overlay does this automatically.

**2. Forgetting to call `autoUpdate` cleanup**
`autoUpdate` registers scroll and resize listeners across every scrollable ancestor. If you destroy the component without calling the returned cleanup function, those listeners leak and cause errors on the next scroll event (the floating element no longer exists but the listener still fires). Always store and call the cleanup in `useEffect` return / `ngOnDestroy`.

**3. Arrow position incorrect after flip**
The arrow's cross-axis position (the `x` value when placement is `top`/`bottom`, the `y` value when placement is `left`/`right`) is provided by the `arrow` middleware in `middlewareData.arrow`. Developers often apply only the primary-axis offset from CSS (`bottom: -4px` for `top` placement) and ignore the cross-axis shift. After a `shift` middleware adjustment, the floating element may have moved horizontally while the arrow stayed centered — making the arrow point to empty space instead of the anchor. Always apply both `arrowX` and `arrowY` from middleware data.

**4. FOUC (flash of unstyled/unpositioned content)**
If the floating element is in the DOM but Floating UI has not computed position yet (first render), it will briefly appear at `top: 0; left: 0` in the page corner. Fix: start with `opacity: 0; visibility: hidden` and only reveal after the first `computePosition` resolves. In the hooks above this is handled by the `is-visible` class applied after the position is ready.

**5. `shift` middleware with `padding: 0` clips to viewport edge**
Without a padding value, `shift` allows the floating element to touch the viewport edge exactly. Content near the edge becomes visually tight and can clip on small screens. Always pass `shift({ padding: 8 })` at minimum. On mobile, increase to 12–16px to match safe-area insets.

**6. Animated anchors require `animationFrame: true` in `autoUpdate`**
If the anchor moves as part of a CSS animation or JS-driven transform (e.g., a button inside a collapsing sidebar), `autoUpdate` with the default `animationFrame: false` will not track the movement. Enable `animationFrame: true` for animated anchors — but be aware this adds a `requestAnimationFrame` loop, so disable it once the anchor is no longer animated.

**7. CDK `scrollStrategies.reposition()` does not update arrow position**
CDK's own reposition scroll strategy updates the overlay panel position but knows nothing about your arrow element. When using the hybrid approach (CDK for overlay management, Floating UI for arrow positioning), you must listen to `positionChanges` on the strategy and re-run your arrow calculation manually — otherwise the arrow will point at the wrong location after the user scrolls.

---

## Interview Angle

**Q: "Why not just use `position: absolute` and manually compute `top`/`left` based on `getBoundingClientRect`?"**

That approach works for the initial render in a static layout, but breaks across three common scenarios. First, when the anchor is near a viewport edge, the floating element clips — you need flip/shift logic that accounts for all four viewport boundaries and updates dynamically. Second, any scrollable ancestor between the anchor and the floating element will cause the position to drift on scroll unless you recompute it — which means writing your own scroll-detection logic across every ancestor. Third, a `transform` on any ancestor makes `position: fixed` behave like `absolute`, which invalidates any assumption about coordinate spaces. Floating UI solves all three with `computePosition` + the middleware pipeline + `autoUpdate`. The few kilobytes it costs are a worthwhile trade against maintaining hundreds of lines of edge-case geometry code.

**Follow-up: "When would you choose CDK Overlay over Floating UI directly?"**

CDK Overlay adds two things Floating UI does not handle: the overlay container appended to `<body>` (which solves the transformed-ancestor problem structurally), and built-in accessibility primitives like `cdkTrapFocus`, backdrop management, and keyboard dismiss. For simple tooltips and popovers in a non-Angular codebase, Floating UI alone is sufficient — it is framework-agnostic and lighter. In an Angular application that already uses Angular Material or CDK, `FlexibleConnectedPositionStrategy` is already battle-tested across every browser and device in that ecosystem, so it is the lower-risk choice. The two are not mutually exclusive: you can use CDK for overlay management and delegate the fine-grained arrow computation to Floating UI's `computePosition`.
