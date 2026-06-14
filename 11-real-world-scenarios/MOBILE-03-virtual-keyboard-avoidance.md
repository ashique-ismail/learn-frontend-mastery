# Virtual Keyboard Avoidance

## The Idea

**In plain English:** On mobile devices, when a user taps an input field the software keyboard rises from the bottom of the screen and consumes 30–60% of the viewport. The browser may or may not resize the viewport, depending on the platform and browser version. Your carefully laid-out form can end up with the submit button hidden behind the keyboard, the focused input scrolled off-screen, or the entire layout jumping as the viewport dimensions change. Virtual keyboard avoidance is the set of techniques that keep the UI functional and visible when that keyboard appears.

**Real-world analogy:** Imagine you are working at a small desk and a colleague sits down beside you, pushing a large divider panel halfway across the desk surface.

- If you were smart about your layout, your notebook (the active input) is in the upper half of the desk and stays visible — you barely notice the divider.
- If your notebook was sitting in the lower half, the divider now covers it completely. You cannot write without craning over the panel.
- The "divider" is the virtual keyboard. The desk is your viewport. Your job as a developer is to ensure the notebook (the currently focused input) is always in the upper, still-visible portion of the desk when the divider slides in.

The complication: different "desks" (iOS Safari, Chrome Android, Firefox Android) move the divider in different ways — some shrink the desk surface, some just cover it, and some do both.

---

## Learning Objectives

- Understand the difference between `vh` and `dvh` and when each breaks on mobile
- Use the `visualViewport` API to measure the portion of the screen not covered by the keyboard
- Detect keyboard open/close events by listening to `visualViewport` resize
- Scroll a focused input into view programmatically with correct timing
- Handle iOS Safari's unique viewport behaviour and identify its specific bugs
- Avoid layout shift caused by the keyboard in full-screen and modal form contexts

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Know current visible viewport height | ❌ | `dvh` reflects the state at paint time, not during keyboard animation |
| React to keyboard open/close events | ❌ | No CSS event corresponds to keyboard visibility change |
| Scroll a specific input into the visible area | ❌ | `scroll-into-view` needs to be called with timing relative to keyboard animation |
| Detect the keyboard height precisely | ❌ | `visualViewport.height` gives this; CSS has no equivalent |
| Fix bottom-anchored buttons above the keyboard | ⚠️ | `env(keyboard-inset-height)` exists but has near-zero browser support in 2025 |
| Suppress iOS Safari rubber-band scroll when input is focused | ❌ | Requires `touchmove` event interception |
| Work around iOS repositioning the viewport on input focus | ❌ | Requires reading `visualViewport.offsetTop` to detect the shift |

**Conclusion:** CSS units (`dvh`, `svh`, `lvh`) solve the static layout problem — they describe the viewport correctly at rest. The dynamic problem (keyboard animating in mid-session, scroll position corrections, platform quirks) requires JavaScript listening on `visualViewport`.

---

## HTML & CSS Foundation

### Viewport Meta Tag (mandatory)

```html
<!-- Without this, mobile browsers apply a scaled-down desktop viewport -->
<meta name="viewport" content="width=device-width, initial-scale=1, interactive-widget=resizes-content">
```

The `interactive-widget=resizes-content` value (Chrome Android 108+) tells the browser to shrink the layout viewport when the keyboard opens, which makes `dvh` reflect the available space correctly. Without it, Chrome Android uses the legacy behaviour of not resizing the layout viewport.

### The Three Viewport Units

```css
/*
  vh  — 1% of the initial containing block height.
        Set once on page load. Does NOT change when the keyboard opens.
        On iOS Safari this equals the full screen height including the
        browser chrome, causing elements to be taller than the visible area.

  svh — small viewport height. Equal to the viewport when the browser chrome
        is fully visible (the smallest possible viewport). Safe floor value.

  dvh — dynamic viewport height. Updates as the browser chrome shows/hides.
        On iOS Safari it follows the address bar collapsing, but NOT the keyboard.
        On Chrome Android with resizes-content it does shrink with the keyboard.
        This is the recommended unit for full-screen layouts.

  lvh — large viewport height. Equal to the viewport when the browser chrome
        is fully hidden (the largest possible viewport). Useful for hero sections.
*/

/* Pattern: full-screen form that fills available space */
.form-screen {
  height: 100dvh;              /* shrinks with chrome on browsers that support it */
  height: 100svh;              /* fallback: never overflow even with chrome visible */
  display: flex;
  flex-direction: column;
}

/* The scrollable content area grows/shrinks */
.form-screen__body {
  flex: 1 1 auto;
  overflow-y: auto;
  -webkit-overflow-scrolling: touch; /* legacy iOS momentum scrolling */
}

/* The sticky bottom CTA — needs to move above the keyboard */
.form-screen__footer {
  flex: 0 0 auto;
  padding: 1rem;
  padding-bottom: calc(1rem + env(safe-area-inset-bottom));
}
```

### The CSS-only Keyboard Inset (where supported)

```css
/*
  env(keyboard-inset-height) is defined in the CSS Environment Variables spec.
  As of mid-2025 it is only supported in Chrome 94+ on Android when
  VirtualKeyboard API (navigator.virtualKeyboard.overlaysContent = true) is enabled.
  It is NOT supported on iOS Safari.
  Use it as progressive enhancement, always alongside a JS fallback.
*/
@supports (height: env(keyboard-inset-height)) {
  .form-screen__footer {
    margin-bottom: env(keyboard-inset-height);
  }
}
```

---

## React Implementation

### The `useVirtualKeyboard` Hook

```tsx
// useVirtualKeyboard.ts
import { useState, useEffect, useCallback, useRef } from 'react';

interface VirtualKeyboardState {
  isOpen: boolean;
  keyboardHeight: number;       // pixels the keyboard occupies
  visibleHeight: number;        // remaining visible viewport height
}

/**
 * Tracks virtual keyboard open/close by comparing visualViewport height
 * against window.innerHeight. Works on iOS Safari and Chrome Android.
 *
 * Caveats:
 *  - iOS Safari may report intermediate heights during the keyboard animation.
 *    The 150ms debounce filters those out.
 *  - On Chrome Android with resizes-content, window.innerHeight already
 *    reflects the keyboard. We use visualViewport.height as the source of truth.
 */
export function useVirtualKeyboard(): VirtualKeyboardState {
  const [state, setState] = useState<VirtualKeyboardState>({
    isOpen: false,
    keyboardHeight: 0,
    visibleHeight: typeof window !== 'undefined' ? window.innerHeight : 0,
  });

  const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null);

  const update = useCallback(() => {
    const vv = window.visualViewport;
    if (!vv) return;

    // visualViewport.height = visible area above the keyboard
    // window.innerHeight    = layout viewport (may not shrink with keyboard)
    const visibleHeight   = vv.height;
    const keyboardHeight  = Math.max(0, window.innerHeight - vv.height - vv.offsetTop);
    const isOpen          = keyboardHeight > 100; // 100px threshold avoids false positives

    setState({ isOpen, keyboardHeight, visibleHeight });
  }, []);

  const debouncedUpdate = useCallback(() => {
    if (timerRef.current) clearTimeout(timerRef.current);
    timerRef.current = setTimeout(update, 150);
  }, [update]);

  useEffect(() => {
    const vv = window.visualViewport;
    if (!vv) return;

    // Initial measurement
    update();

    vv.addEventListener('resize', debouncedUpdate);
    vv.addEventListener('scroll', debouncedUpdate); // iOS Safari fires scroll, not resize

    return () => {
      vv.removeEventListener('resize', debouncedUpdate);
      vv.removeEventListener('scroll', debouncedUpdate);
      if (timerRef.current) clearTimeout(timerRef.current);
    };
  }, [update, debouncedUpdate]);

  return state;
}
```

### The `useScrollInputIntoView` Hook

```tsx
// useScrollInputIntoView.ts
import { useCallback } from 'react';

/**
 * After the keyboard opens, the focused input may be partially or fully
 * hidden behind it. This hook returns a function that scrolls the input
 * into the visible area.
 *
 * Why not use element.scrollIntoView()?
 *  - On iOS Safari, scrollIntoView during the keyboard animation causes a
 *    double-scroll jank. We need to wait for the keyboard to finish opening,
 *    then scroll.
 *  - We scroll the nearest scrollable ancestor, not the whole window, so
 *    fixed headers are respected.
 */
export function useScrollInputIntoView() {
  return useCallback((element: HTMLElement | null, keyboardHeight: number) => {
    if (!element) return;

    const vv = window.visualViewport;
    if (!vv) return;

    const rect = element.getBoundingClientRect();
    const visibleBottom = vv.height; // top of the keyboard in viewport coords

    // Add padding so the input isn't right at the keyboard edge
    const padding = 16;
    const inputBottom = rect.bottom + padding;

    if (inputBottom > visibleBottom) {
      const overlapAmount = inputBottom - visibleBottom;

      // Find the nearest scrollable ancestor
      let scrollParent: HTMLElement | null = element.parentElement;
      while (scrollParent) {
        const style = window.getComputedStyle(scrollParent);
        const overflow = style.overflow + style.overflowY;
        if (/auto|scroll/.test(overflow)) break;
        scrollParent = scrollParent.parentElement;
      }

      if (scrollParent) {
        scrollParent.scrollBy({ top: overlapAmount, behavior: 'smooth' });
      } else {
        window.scrollBy({ top: overlapAmount, behavior: 'smooth' });
      }
    }
  }, []);
}
```

### Full Working Component: Mobile-Safe Form

```tsx
// MobileForm.tsx
import React, { useRef, useEffect, useId } from 'react';
import { useVirtualKeyboard } from './useVirtualKeyboard';
import { useScrollInputIntoView } from './useScrollInputIntoView';

interface FormField {
  name: string;
  label: string;
  type: 'text' | 'email' | 'tel' | 'password';
  autoComplete?: string;
}

const FIELDS: FormField[] = [
  { name: 'name',     label: 'Full name',     type: 'text',     autoComplete: 'name'     },
  { name: 'email',    label: 'Email address', type: 'email',    autoComplete: 'email'    },
  { name: 'phone',    label: 'Phone number',  type: 'tel',      autoComplete: 'tel'      },
  { name: 'password', label: 'Password',      type: 'password', autoComplete: 'new-password' },
];

export function MobileForm() {
  const formRef           = useRef<HTMLFormElement>(null);
  const footerRef         = useRef<HTMLDivElement>(null);
  const { isOpen, keyboardHeight, visibleHeight } = useVirtualKeyboard();
  const scrollInputIntoView = useScrollInputIntoView();
  const formId = useId();

  // When the keyboard opens, push the footer above it using a CSS custom property.
  // This is more reliable than position:fixed + bottom because position:fixed
  // is broken on iOS Safari inside scroll containers.
  useEffect(() => {
    const footer = footerRef.current;
    if (!footer) return;

    if (isOpen) {
      // Lift the footer above the keyboard
      footer.style.transform = `translateY(-${keyboardHeight}px)`;
      footer.style.transition = 'transform 0.25s ease-out';
    } else {
      footer.style.transform = 'translateY(0)';
    }
  }, [isOpen, keyboardHeight]);

  // When the keyboard opens, scroll the currently focused input into view
  // after the keyboard animation finishes (250ms is the typical keyboard
  // animation duration on both iOS and Android).
  useEffect(() => {
    if (!isOpen) return;

    const timer = setTimeout(() => {
      const active = document.activeElement as HTMLElement | null;
      if (active && formRef.current?.contains(active)) {
        scrollInputIntoView(active, keyboardHeight);
      }
    }, 300); // slightly longer than keyboard animation

    return () => clearTimeout(timer);
  }, [isOpen, keyboardHeight, scrollInputIntoView]);

  const handleFocus = (e: React.FocusEvent<HTMLInputElement>) => {
    // On iOS Safari, the browser scrolls the input into view itself,
    // but it often scrolls too far, hiding the input behind the keyboard.
    // We correct this after a delay.
    const input = e.currentTarget;
    setTimeout(() => scrollInputIntoView(input, keyboardHeight), 350);
  };

  return (
    <div
      className="form-screen"
      style={{
        // Use visibleHeight as an inline style fallback for browsers that
        // do not support dvh. The hook keeps this updated in real-time.
        height: `${visibleHeight}px`,
      }}
    >
      {/* Scrollable form body */}
      <div className="form-screen__body">
        <form
          id={formId}
          ref={formRef}
          className="form-screen__form"
          noValidate
          onSubmit={e => { e.preventDefault(); /* submit logic */ }}
        >
          <h1 className="form-screen__title">Create account</h1>

          {FIELDS.map(field => (
            <div key={field.name} className="form-group">
              <label htmlFor={`${formId}-${field.name}`} className="form-label">
                {field.label}
              </label>
              <input
                id={`${formId}-${field.name}`}
                name={field.name}
                type={field.type}
                autoComplete={field.autoComplete}
                className="form-input"
                onFocus={handleFocus}
                /*
                  enterKeyHint controls the Return key label on the virtual keyboard.
                  'next' shows "Next" for all but the last field; 'done' closes the keyboard.
                */
                enterKeyHint={field.name === 'password' ? 'done' : 'next'}
              />
            </div>
          ))}

          {/* Extra padding at the bottom so the last input
              is never hidden under the footer when scrolled to bottom */}
          <div style={{ height: '6rem' }} aria-hidden="true" />
        </form>
      </div>

      {/* Bottom CTA — slides above keyboard when it opens */}
      <div ref={footerRef} className="form-screen__footer">
        <button
          type="submit"
          form={formId}
          className="btn btn--primary btn--full"
        >
          Create account
        </button>
        <p className="form-screen__terms">
          By continuing you agree to our Terms of Service.
        </p>
      </div>
    </div>
  );
}
```

---

## Angular Implementation

### VirtualKeyboard Service

```typescript
// virtual-keyboard.service.ts
import { Injectable, signal, computed, OnDestroy, NgZone, inject } from '@angular/core';
import { DOCUMENT } from '@angular/common';

@Injectable({ providedIn: 'root' })
export class VirtualKeyboardService implements OnDestroy {
  private zone     = inject(NgZone);
  private document = inject(DOCUMENT);

  private _visibleHeight = signal(
    this.document.defaultView?.innerHeight ?? 0
  );
  private _keyboardHeight = signal(0);

  readonly visibleHeight  = this._visibleHeight.asReadonly();
  readonly keyboardHeight = this._keyboardHeight.asReadonly();
  readonly isOpen         = computed(() => this._keyboardHeight() > 100);

  private timer: ReturnType<typeof setTimeout> | null = null;
  private vv: VisualViewport | null = null;

  constructor() {
    this.vv = this.document.defaultView?.visualViewport ?? null;
    if (!this.vv) return;

    // Run outside Angular zone — these events fire frequently during animation.
    // We only re-enter the zone (triggering change detection) when values stabilise.
    this.zone.runOutsideAngular(() => {
      this.vv!.addEventListener('resize', this.onViewportChange);
      this.vv!.addEventListener('scroll', this.onViewportChange);
    });

    this.measure(); // initial measurement
  }

  private onViewportChange = () => {
    if (this.timer) clearTimeout(this.timer);
    this.timer = setTimeout(() => {
      this.zone.run(() => this.measure());
    }, 150);
  };

  private measure(): void {
    const vv = this.vv;
    const win = this.document.defaultView;
    if (!vv || !win) return;

    const visibleHeight  = vv.height;
    const keyboardHeight = Math.max(0, win.innerHeight - vv.height - vv.offsetTop);

    this._visibleHeight.set(visibleHeight);
    this._keyboardHeight.set(keyboardHeight);
  }

  scrollInputIntoView(element: HTMLElement): void {
    const vv = this.vv;
    if (!vv || !element) return;

    const rect         = element.getBoundingClientRect();
    const visibleBottom = vv.height;
    const padding       = 16;

    if (rect.bottom + padding > visibleBottom) {
      const overlapAmount = rect.bottom + padding - visibleBottom;
      window.scrollBy({ top: overlapAmount, behavior: 'smooth' });
    }
  }

  ngOnDestroy(): void {
    this.vv?.removeEventListener('resize', this.onViewportChange);
    this.vv?.removeEventListener('scroll', this.onViewportChange);
    if (this.timer) clearTimeout(this.timer);
  }
}
```

### Mobile Form Component

```typescript
// mobile-form.component.ts
import {
  Component, OnInit, OnDestroy, inject, effect,
  ElementRef, ViewChild
} from '@angular/core';
import { FormsModule } from '@angular/forms';
import { NgFor } from '@angular/common';
import { VirtualKeyboardService } from './virtual-keyboard.service';

interface FormField {
  name: string;
  label: string;
  type: string;
  autoComplete: string;
  enterKeyHint: string;
}

@Component({
  selector: 'app-mobile-form',
  standalone: true,
  imports: [FormsModule, NgFor],
  template: `
    <div
      class="form-screen"
      [style.height.px]="kb.visibleHeight()"
    >
      <!-- Scrollable body -->
      <div class="form-screen__body" #scrollBody>
        <form class="form-screen__form" (ngSubmit)="onSubmit()" novalidate>
          <h1 class="form-screen__title">Create account</h1>

          <div
            *ngFor="let field of fields; trackBy: trackByName"
            class="form-group"
          >
            <label [for]="'field-' + field.name" class="form-label">
              {{ field.label }}
            </label>
            <input
              [id]="'field-' + field.name"
              [name]="field.name"
              [type]="field.type"
              [attr.autocomplete]="field.autoComplete"
              [attr.enterkeyhint]="field.enterKeyHint"
              class="form-input"
              (focus)="onInputFocus($event)"
            />
          </div>

          <!-- Bottom padding so last input is never hidden under footer -->
          <div style="height: 6rem" aria-hidden="true"></div>
        </form>
      </div>

      <!-- Footer: lifted above keyboard via transform -->
      <div class="form-screen__footer" #footer>
        <button type="submit" class="btn btn--primary btn--full">
          Create account
        </button>
        <p class="form-screen__terms">
          By continuing you agree to our Terms of Service.
        </p>
      </div>
    </div>
  `,
})
export class MobileFormComponent implements OnInit, OnDestroy {
  kb = inject(VirtualKeyboardService);
  private el = inject(ElementRef);

  @ViewChild('footer', { static: true }) footerRef!: ElementRef<HTMLDivElement>;

  fields: FormField[] = [
    { name: 'name',     label: 'Full name',     type: 'text',     autoComplete: 'name',         enterKeyHint: 'next' },
    { name: 'email',    label: 'Email address', type: 'email',    autoComplete: 'email',        enterKeyHint: 'next' },
    { name: 'phone',    label: 'Phone number',  type: 'tel',      autoComplete: 'tel',          enterKeyHint: 'next' },
    { name: 'password', label: 'Password',      type: 'password', autoComplete: 'new-password', enterKeyHint: 'done' },
  ];

  private scrollTimer: ReturnType<typeof setTimeout> | null = null;

  constructor() {
    // React to keyboard open/close — lift footer above keyboard
    effect(() => {
      const keyboardHeight = this.kb.keyboardHeight();
      const footer = this.footerRef?.nativeElement;
      if (!footer) return;

      if (keyboardHeight > 0) {
        footer.style.transform = `translateY(-${keyboardHeight}px)`;
        footer.style.transition = 'transform 0.25s ease-out';
      } else {
        footer.style.transform = 'translateY(0)';
      }
    });

    // Scroll focused input into view after keyboard finishes animating
    effect(() => {
      const isOpen = this.kb.isOpen();
      if (!isOpen) return;

      if (this.scrollTimer) clearTimeout(this.scrollTimer);
      this.scrollTimer = setTimeout(() => {
        const active = document.activeElement as HTMLElement | null;
        if (active && this.el.nativeElement.contains(active)) {
          this.kb.scrollInputIntoView(active);
        }
      }, 300);
    });
  }

  ngOnInit(): void {}

  ngOnDestroy(): void {
    if (this.scrollTimer) clearTimeout(this.scrollTimer);
  }

  onInputFocus(event: FocusEvent): void {
    const input = event.target as HTMLElement;
    // Correct iOS Safari's own scroll attempt after a delay
    setTimeout(() => this.kb.scrollInputIntoView(input), 350);
  }

  onSubmit(): void {
    // Dismiss keyboard on submit
    (document.activeElement as HTMLElement)?.blur();
  }

  trackByName(_: number, field: FormField): string {
    return field.name;
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Focused input always visible above keyboard | `scrollInputIntoView` called on `focus` and on `visualViewport` resize |
| CTA button reachable without scrolling | Footer lifted via `translateY` equal to keyboard height |
| `enterKeyHint` set on all inputs | Correctly labelled Return key (`next` / `done`) guides users through the form |
| `autocomplete` attributes set | Reduces keyboard interactions by enabling browser autofill |
| Input labels never behind keyboard | Labels are above their inputs; the scrollable body scrolls them into view |
| No content permanently hidden | Extra `6rem` bottom padding ensures the last input can scroll above the footer |
| Safe area insets respected | Footer padding uses `env(safe-area-inset-bottom)` for notched devices |
| Touch targets minimum 44x44px | `min-height: 44px` on all inputs and buttons |
| Keyboard dismissal on submit | `blur()` called on active element in `onSubmit` handler |

---

## Production Pitfalls

**1. `100vh` overflows on iOS Safari**
iOS Safari calculates `100vh` as the full screen height including the top address bar and bottom toolbar chrome. An element set to `height: 100vh` is physically taller than the visible area, causing overflow that the user cannot scroll to see. Fix: use `height: 100dvh` with `height: 100svh` as the fallback. Apply the `visualViewport.height` value as an inline style for browsers without `dvh` support.

**2. `visualViewport` does not fire `resize` on iOS Safari — it fires `scroll`**
On iOS Safari, when the keyboard opens the browser pans the viewport upward rather than shrinking it. This fires a `scroll` event on `visualViewport`, not a `resize` event. Code that only listens to `resize` will never detect the keyboard on iOS Safari. Always attach both listeners:
```ts
visualViewport.addEventListener('resize', handler);
visualViewport.addEventListener('scroll', handler);
```

**3. iOS Safari repositions the viewport, not the content**
When an input is focused, iOS Safari adds an `offsetTop` to `visualViewport` that represents how far the viewport has been panned upward. If you calculate keyboard height as `window.innerHeight - visualViewport.height`, you get a wrong value on iOS. The correct formula is:
```ts
const keyboardHeight = Math.max(0, window.innerHeight - visualViewport.height - visualViewport.offsetTop);
```
Ignoring `offsetTop` is the most common source of incorrect keyboard height measurements on iOS.

**4. `scrollIntoView` conflicts with the browser's own scroll attempt**
When an input is focused, iOS Safari and Chrome Android both attempt to scroll the input into view themselves. If you also call `scrollIntoView()` simultaneously, the two scroll operations fight each other, producing jank. Fix: always call your scroll correction with a `setTimeout` of at least 300ms — long enough for the browser's own scroll and the keyboard animation to complete.

**5. `position: fixed` elements break inside scroll containers on iOS Safari**
A footer with `position: fixed; bottom: 0` works on desktop but on iOS Safari, when the footer is inside an element with `-webkit-overflow-scrolling: touch`, it loses its fixed positioning and scrolls with the content. Fix: use the `translateY` approach — keep the footer in normal flow and lift it above the keyboard using a CSS transform keyed to `visualViewport.height`.

**6. Keyboard height changes midway through a transition**
On Android, switching from a full QWERTY keyboard to a numeric keypad (e.g. when the user focuses a `type="tel"` input) causes the keyboard to resize without fully closing. The `visualViewport` resize event fires multiple times. If your listener updates layout on every event rather than debouncing, you get visible layout thrashing. Apply a 150ms debounce to all `visualViewport` event handlers.

**7. `dvh` does not update during the keyboard animation on Chrome Android without `resizes-content`**
`dvh` updates lazily at paint time, not frame-by-frame during the keyboard animation. Without the `interactive-widget=resizes-content` meta tag, Chrome Android does not shrink the layout viewport at all when the keyboard opens. The result is that `height: 100dvh` elements remain full-screen height even though the keyboard is covering the bottom half. The `visualViewport.height` inline style approach bypasses both problems because it reads the actual visible area in real time.

**8. The 300ms focus delay heuristic fails on slow-to-open keyboards**
Some Android devices with third-party keyboards (SwiftKey, Gboard in certain configurations) animate for 400–500ms. A fixed 300ms delay means `scrollInputIntoView` is called before the keyboard is fully open, so the scroll correction is based on a partial keyboard height. Fix: listen for the `visualViewport` resize to stabilise (two consecutive readings within 10px) rather than relying on a fixed timeout.

---

## Interview Angle

**Q: "A user fills in a form on mobile and the submit button is hidden behind the keyboard. How do you fix that?"**

The first step is to distinguish between the two root causes: either the button is below the fold at rest and the keyboard pushed visible content upward, or the button is in a fixed-bottom footer that is sitting behind the keyboard. The fix for a fixed footer is to read `window.visualViewport.height` on every `resize` and `scroll` event of `visualViewport`, compute the keyboard height as `window.innerHeight - visualViewport.height - visualViewport.offsetTop`, then apply `translateY(-${keyboardHeight}px)` to the footer element. For the scroll-based case you call `scrollBy` on the nearest scrollable ancestor to move the active input into the visible area above the keyboard. On iOS Safari, you must listen to both `resize` and `scroll` events on `visualViewport` because the platform fires `scroll` rather than `resize` when it pans the viewport on input focus.

**Follow-up: "What is the difference between `vh`, `svh`, and `dvh`, and which one should you use for a full-screen mobile form?"**

`vh` is frozen at page load time and equals the full screen height on iOS Safari including browser chrome, making it unreliable for any element that must fit within the visible area. `svh` (small viewport height) is the viewport height when the browser chrome is maximally visible — it is the safe conservative floor and will never overflow, but elements sized in `svh` feel short on desktop or when chrome is collapsed. `dvh` (dynamic viewport height) updates as the browser chrome shows and hides, making it the best unit for full-screen layouts. However, `dvh` does not update during the keyboard animation on Chrome Android unless the `interactive-widget=resizes-content` meta tag is set, so it must be paired with the `visualViewport.height` inline style as a runtime fallback. The production-safe pattern is `height: 100svh` as the CSS baseline, overridden by `height: 100dvh` where supported, with a JavaScript listener on `visualViewport` that sets the actual available height as an inline style in real time.
