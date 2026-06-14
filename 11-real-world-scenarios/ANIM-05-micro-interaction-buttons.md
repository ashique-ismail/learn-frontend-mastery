# Micro-Interaction Button States

## The Idea

**In plain English:** A submit button that silently ignores a second click and instead shows the user exactly what is happening — a spinner while the request is in flight, a checkmark when it succeeds, an error icon when it fails — and then resets itself so the user can try again. Every state transition is animated so the change never feels jarring.

**Real-world analogy:** Think of an ATM cash-withdrawal button.

- You press **Withdraw** — the button physically depresses and the screen shows a spinner (idle → loading).
- The machine counts the notes internally — you cannot press Withdraw again during this time; the button is locked (double-submit prevention).
- The drawer opens and the screen flashes a green tick — the button briefly shows "Done" before resetting (loading → success → idle).
- If the machine is out of notes it flashes red with "Unable to dispense" — you see the error, then the button resets so you can choose a different amount (loading → error → idle).

The **button face** is a tiny stage: it morphs between four distinct props — label, spinner, checkmark, X-mark — using animation to make each transition legible rather than abrupt. The **locking mechanism** is invisible but critical; without it a frustrated user double-taps and fires two API calls.

---

## Learning Objectives

- Model a four-state button machine (idle / loading / success / error) with typed state
- Drive a CSS spinner using `@keyframes` and understand why `animation` beats `transition` for continuous rotation
- Morph an SVG checkmark using `stroke-dashoffset` so the path appears to draw itself
- Prevent double-submit at the component level without relying on server-side idempotency alone
- Trigger the Web Vibration API for tactile feedback on success and error on mobile
- Reset state automatically after a success/error dwell period using a cleanup-safe timer
- Write accessible ARIA that describes the live button state to screen readers

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Animate the spinner rotation | ✅ `@keyframes` | — |
| Draw the checkmark via `stroke-dashoffset` | ✅ CSS animation | — |
| Track which of four states is active | ❌ | CSS has no enum state; it reacts to classes but cannot set them |
| Disable the button during loading | ❌ | `pointer-events: none` visually blocks but does not set `disabled` or update ARIA |
| Fire the Vibration API on state change | ❌ | CSS cannot call `navigator.vibrate()` |
| Auto-reset after a 2-second success dwell | ❌ | CSS has no `setTimeout` equivalent |
| Prevent a queued second click from firing | ❌ | Requires a JS ref/flag to gate the async call |
| Announce state change to screen readers | ❌ | `aria-live` attribute must be toggled by JS |

**Conclusion:** CSS owns the visual morphing — spinner rotation, checkmark stroke, color transitions, size shifts. JS owns the state machine, the async gate, the timer, and the ARIA updates.

---

## HTML & CSS Foundation

### The Structure

```html
<!--
  data-state drives all visual changes via CSS attribute selectors.
  Values: "idle" | "loading" | "success" | "error"
-->
<button
  class="action-btn"
  data-state="idle"
  aria-live="polite"
  aria-label="Save changes"
>
  <!-- Slot 1: default label -->
  <span class="btn-label">Save changes</span>

  <!-- Slot 2: loading spinner (CSS-drawn ring) -->
  <span class="btn-spinner" aria-hidden="true"></span>

  <!-- Slot 3: success checkmark (inline SVG for stroke-dashoffset trick) -->
  <svg class="btn-icon btn-icon--success" viewBox="0 0 24 24" aria-hidden="true">
    <polyline class="check-path" points="4,13 9,18 20,6" />
  </svg>

  <!-- Slot 4: error X mark -->
  <svg class="btn-icon btn-icon--error" viewBox="0 0 24 24" aria-hidden="true">
    <line class="x-path x-path--1" x1="5" y1="5" x2="19" y2="19" />
    <line class="x-path x-path--2" x1="19" y1="5" x2="5"  y2="19" />
  </svg>
</button>
```

### The CSS

```css
/* ─── Design tokens ─── */
:root {
  --btn-idle-bg:    #2563eb;
  --btn-idle-fg:    #ffffff;
  --btn-success-bg: #16a34a;
  --btn-error-bg:   #dc2626;
  --btn-radius:     0.5rem;
  --btn-h:          2.75rem;
  --btn-transition: 0.25s cubic-bezier(0.4, 0, 0.2, 1);
  --btn-min-w:      9rem;
}

/* ─── Base button ─── */
.action-btn {
  position: relative;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: var(--btn-min-w);
  height: var(--btn-h);
  padding: 0 1.5rem;
  border: none;
  border-radius: var(--btn-radius);
  background: var(--btn-idle-bg);
  color: var(--btn-idle-fg);
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  overflow: hidden;
  transition:
    background var(--btn-transition),
    transform  var(--btn-transition),
    box-shadow var(--btn-transition);
}

/* Subtle press-down on activation */
.action-btn:active:not([disabled]) {
  transform: scale(0.97);
}

/* Focus ring visible for keyboard users */
.action-btn:focus-visible {
  outline: 3px solid #93c5fd;
  outline-offset: 3px;
}

/* ─── Per-slot visibility: only one slot is shown per state ─── */
.btn-label,
.btn-spinner,
.btn-icon {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: opacity var(--btn-transition), transform var(--btn-transition);
}

/* Default: label visible, others hidden */
.btn-label   { opacity: 1; transform: scale(1); }
.btn-spinner { opacity: 0; transform: scale(0.6); }
.btn-icon    { opacity: 0; transform: scale(0.6); }

/* ─── idle state (explicit, same as default) ─── */
.action-btn[data-state="idle"] .btn-label   { opacity: 1; transform: scale(1); }
.action-btn[data-state="idle"] .btn-spinner { opacity: 0; transform: scale(0.6); }
.action-btn[data-state="idle"] .btn-icon    { opacity: 0; transform: scale(0.6); }

/* ─── loading state ─── */
.action-btn[data-state="loading"] {
  background: var(--btn-idle-bg);
  cursor: not-allowed;
}
.action-btn[data-state="loading"] .btn-label   { opacity: 0; transform: scale(0.6); }
.action-btn[data-state="loading"] .btn-spinner { opacity: 1; transform: scale(1);   }
.action-btn[data-state="loading"] .btn-icon    { opacity: 0; transform: scale(0.6); }

/* ─── success state ─── */
.action-btn[data-state="success"] {
  background: var(--btn-success-bg);
}
.action-btn[data-state="success"] .btn-label              { opacity: 0; transform: scale(0.6); }
.action-btn[data-state="success"] .btn-spinner            { opacity: 0; transform: scale(0.6); }
.action-btn[data-state="success"] .btn-icon--success      { opacity: 1; transform: scale(1);   }
.action-btn[data-state="success"] .btn-icon--error        { opacity: 0; }

/* ─── error state ─── */
.action-btn[data-state="error"] {
  background: var(--btn-error-bg);
}
.action-btn[data-state="error"] .btn-label           { opacity: 0; transform: scale(0.6); }
.action-btn[data-state="error"] .btn-spinner         { opacity: 0; transform: scale(0.6); }
.action-btn[data-state="error"] .btn-icon--error     { opacity: 1; transform: scale(1);   }
.action-btn[data-state="error"] .btn-icon--success   { opacity: 0; }

/* ─── Spinner: CSS ring via border trick ─── */
.btn-spinner::before {
  content: '';
  width: 1.25rem;
  height: 1.25rem;
  border: 3px solid rgba(255, 255, 255, 0.35);
  border-top-color: #ffffff;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

/* ─── Checkmark: SVG stroke-dashoffset draw-on animation ─── */
.check-path {
  fill: none;
  stroke: #ffffff;
  stroke-width: 2.5;
  stroke-linecap: round;
  stroke-linejoin: round;
  /*
    Total polyline length ≈ 22px for this viewBox/points.
    Set dasharray = dashoffset = 22 so the stroke is invisible.
    When state becomes "success", animate dashoffset to 0 to draw it.
  */
  stroke-dasharray: 22;
  stroke-dashoffset: 22;
}

.action-btn[data-state="success"] .check-path {
  animation: draw-check 0.35s 0.1s cubic-bezier(0.4, 0, 0.2, 1) forwards;
}

@keyframes draw-check {
  to { stroke-dashoffset: 0; }
}

/* ─── Error X: same trick, two lines ─── */
.x-path {
  fill: none;
  stroke: #ffffff;
  stroke-width: 2.5;
  stroke-linecap: round;
  stroke-dasharray: 20;
  stroke-dashoffset: 20;
}

.action-btn[data-state="error"] .x-path--1 {
  animation: draw-x 0.2s 0.05s ease-out forwards;
}
.action-btn[data-state="error"] .x-path--2 {
  animation: draw-x 0.2s 0.18s ease-out forwards;
}

@keyframes draw-x {
  to { stroke-dashoffset: 0; }
}

/* ─── Reduced motion: keep state changes, drop animations ─── */
@media (prefers-reduced-motion: reduce) {
  .action-btn,
  .btn-label,
  .btn-spinner,
  .btn-icon {
    transition: none;
  }
  .btn-spinner::before {
    animation: none;
    border-top-color: #ffffff;   /* show a static arc instead */
  }
  .check-path,
  .x-path {
    animation: none;
    stroke-dashoffset: 0;        /* checkmark / X visible immediately */
  }
}
```

**What CSS owns:** slot visibility cross-fades, background color transitions, spinner rotation, checkmark and X draw-on animations, press-down transform, focus ring, reduced-motion fallbacks.

**What CSS cannot own:** which `data-state` value is set, the async lock, the auto-reset timer, and the Vibration API call.

---

## React Implementation

### The State Machine Hook

```tsx
// useButtonState.ts
import { useState, useRef, useCallback, useEffect } from 'react';

export type ButtonState = 'idle' | 'loading' | 'success' | 'error';

interface UseButtonStateOptions {
  /** How long (ms) to show success/error before resetting. Default 2000. */
  resetDelay?: number;
  /** Trigger haptic feedback via Vibration API. Default true. */
  haptic?: boolean;
}

interface UseButtonStateReturn {
  state: ButtonState;
  /** Wrap your async handler in this — it manages state automatically. */
  trigger: (asyncFn: () => Promise<void>) => void;
  isLocked: boolean;
}

export function useButtonState({
  resetDelay = 2000,
  haptic = true,
}: UseButtonStateOptions = {}): UseButtonStateReturn {
  const [state, setState] = useState<ButtonState>('idle');

  // Ref-based lock: prevents a second trigger() call while loading.
  // Using a ref rather than deriving from state avoids the stale-closure
  // problem where the click handler captures an old state value.
  const isLockedRef = useRef(false);

  // Track the reset timer so we can cancel it on unmount
  const resetTimerRef = useRef<ReturnType<typeof setTimeout> | null>(null);

  // Cancel any pending reset on unmount to prevent setState on an
  // unmounted component (and the associated React 18 warning in strict mode)
  useEffect(() => {
    return () => {
      if (resetTimerRef.current !== null) {
        clearTimeout(resetTimerRef.current);
      }
    };
  }, []);

  const vibrate = useCallback((pattern: number | number[]) => {
    if (haptic && 'vibrate' in navigator) {
      try {
        navigator.vibrate(pattern);
      } catch {
        // Vibration API can throw in some browser contexts; swallow silently
      }
    }
  }, [haptic]);

  const scheduleReset = useCallback(() => {
    resetTimerRef.current = setTimeout(() => {
      setState('idle');
      isLockedRef.current = false;
      resetTimerRef.current = null;
    }, resetDelay);
  }, [resetDelay]);

  const trigger = useCallback((asyncFn: () => Promise<void>) => {
    // Hard gate: if locked, ignore the click entirely
    if (isLockedRef.current) return;

    isLockedRef.current = true;
    setState('loading');

    asyncFn()
      .then(() => {
        setState('success');
        // Short double-pulse: success haptic
        vibrate([50, 30, 50]);
        scheduleReset();
      })
      .catch(() => {
        setState('error');
        // Longer single buzz: error haptic
        vibrate(200);
        scheduleReset();
      });
  }, [vibrate, scheduleReset]);

  return {
    state,
    trigger,
    isLocked: isLockedRef.current,
  };
}
```

### The Button Component

```tsx
// ActionButton.tsx
import { useRef, useEffect } from 'react';
import { useButtonState, ButtonState } from './useButtonState';

interface ActionButtonProps {
  /** The action to execute when the button is clicked. Must return a Promise. */
  onAction: () => Promise<void>;
  label: string;
  loadingLabel?: string;
  successLabel?: string;
  errorLabel?: string;
  resetDelay?: number;
  haptic?: boolean;
  className?: string;
}

export function ActionButton({
  onAction,
  label,
  loadingLabel = 'Saving…',
  successLabel = 'Saved!',
  errorLabel   = 'Failed',
  resetDelay,
  haptic,
  className = '',
}: ActionButtonProps) {
  const { state, trigger } = useButtonState({ resetDelay, haptic });

  // The visible text label per state — announced to screen readers via
  // aria-live="polite" on the button itself
  const visibleLabel: Record<ButtonState, string> = {
    idle:    label,
    loading: loadingLabel,
    success: successLabel,
    error:   errorLabel,
  };

  return (
    <button
      className={`action-btn ${className}`}
      data-state={state}
      disabled={state === 'loading'}
      aria-live="polite"
      aria-label={visibleLabel[state]}
      onClick={() => trigger(onAction)}
    >
      {/* Label slot — also used by screen readers as fallback text */}
      <span className="btn-label" aria-hidden="true">
        {label}
      </span>

      {/* Spinner slot */}
      <span className="btn-spinner" aria-hidden="true" />

      {/* Success checkmark */}
      <svg
        className="btn-icon btn-icon--success"
        viewBox="0 0 24 24"
        aria-hidden="true"
      >
        <polyline className="check-path" points="4,13 9,18 20,6" />
      </svg>

      {/* Error X mark */}
      <svg
        className="btn-icon btn-icon--error"
        viewBox="0 0 24 24"
        aria-hidden="true"
      >
        <line className="x-path x-path--1" x1="5" y1="5"  x2="19" y2="19" />
        <line className="x-path x-path--2" x1="19" y1="5" x2="5"  y2="19" />
      </svg>
    </button>
  );
}
```

### Usage

```tsx
// SaveForm.tsx
import { ActionButton } from './ActionButton';

async function saveProfile(data: FormData): Promise<void> {
  const res = await fetch('/api/profile', { method: 'PUT', body: data });
  if (!res.ok) throw new Error('Save failed');
}

export function SaveForm() {
  const handleSave = async () => {
    const data = new FormData(document.getElementById('profile-form') as HTMLFormElement);
    await saveProfile(data);
  };

  return (
    <form id="profile-form">
      {/* form fields */}
      <ActionButton
        onAction={handleSave}
        label="Save changes"
        loadingLabel="Saving…"
        successLabel="Saved!"
        errorLabel="Failed — retry"
        resetDelay={2500}
        haptic={true}
      />
    </form>
  );
}
```

---

## Angular Implementation

### The Button State Service

```typescript
// button-state.service.ts
import { Injectable, signal, computed } from '@angular/core';

export type ButtonState = 'idle' | 'loading' | 'success' | 'error';

@Injectable()   // not providedIn: 'root' — each button instance gets its own
export class ButtonStateService {
  private _state  = signal<ButtonState>('idle');
  private _locked = false;
  private _timer: ReturnType<typeof setTimeout> | null = null;

  readonly state   = this._state.asReadonly();
  readonly isLocked = computed(() => this._state() === 'loading');

  async trigger(
    asyncFn: () => Promise<void>,
    resetDelay = 2000,
    haptic     = true,
  ): Promise<void> {
    if (this._locked) return;
    this._locked = true;
    this._state.set('loading');

    try {
      await asyncFn();
      this._state.set('success');
      this.vibrate(haptic, [50, 30, 50]);
    } catch {
      this._state.set('error');
      this.vibrate(haptic, 200);
    } finally {
      this._timer = setTimeout(() => {
        this._state.set('idle');
        this._locked = false;
        this._timer  = null;
      }, resetDelay);
    }
  }

  cleanup(): void {
    if (this._timer !== null) {
      clearTimeout(this._timer);
      this._timer = null;
    }
  }

  private vibrate(enabled: boolean, pattern: number | number[]): void {
    if (!enabled) return;
    try {
      if ('vibrate' in navigator) navigator.vibrate(pattern);
    } catch { /* swallow */ }
  }
}
```

### The Standalone Button Component

```typescript
// action-button.component.ts
import {
  Component, Input, OnDestroy, inject, computed,
  ChangeDetectionStrategy,
} from '@angular/core';
import { ButtonStateService, ButtonState } from './button-state.service';

@Component({
  selector: 'app-action-button',
  standalone: true,
  providers: [ButtonStateService],   // per-instance service — each button has its own state
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <button
      class="action-btn"
      [attr.data-state]="svc.state()"
      [disabled]="svc.isLocked()"
      [attr.aria-label]="currentLabel()"
      aria-live="polite"
      (click)="handleClick()"
    >
      <!-- Label -->
      <span class="btn-label" aria-hidden="true">{{ label }}</span>

      <!-- Spinner -->
      <span class="btn-spinner" aria-hidden="true"></span>

      <!-- Success checkmark -->
      <svg class="btn-icon btn-icon--success" viewBox="0 0 24 24" aria-hidden="true">
        <polyline class="check-path" points="4,13 9,18 20,6" />
      </svg>

      <!-- Error X -->
      <svg class="btn-icon btn-icon--error" viewBox="0 0 24 24" aria-hidden="true">
        <line class="x-path x-path--1" x1="5" y1="5"  x2="19" y2="19" />
        <line class="x-path x-path--2" x1="19" y1="5" x2="5"  y2="19" />
      </svg>
    </button>
  `,
})
export class ActionButtonComponent implements OnDestroy {
  @Input({ required: true }) label!: string;
  @Input() loadingLabel = 'Saving…';
  @Input() successLabel = 'Saved!';
  @Input() errorLabel   = 'Failed';
  @Input() resetDelay   = 2000;
  @Input() haptic       = true;
  /** The async function to run on click. Must return Promise<void>. */
  @Input({ required: true }) action!: () => Promise<void>;

  protected svc = inject(ButtonStateService);

  protected currentLabel = computed<string>(() => {
    const map: Record<ButtonState, string> = {
      idle:    this.label,
      loading: this.loadingLabel,
      success: this.successLabel,
      error:   this.errorLabel,
    };
    return map[this.svc.state()];
  });

  handleClick(): void {
    this.svc.trigger(this.action, this.resetDelay, this.haptic);
  }

  ngOnDestroy(): void {
    this.svc.cleanup();
  }
}
```

### Usage in a Parent Component

```typescript
// profile-form.component.ts
import { Component, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';
import { ActionButtonComponent } from './action-button.component';

@Component({
  selector: 'app-profile-form',
  standalone: true,
  imports: [ActionButtonComponent],
  template: `
    <form #formEl (ngSubmit)="false">
      <!-- form fields -->
      <app-action-button
        label="Save changes"
        loadingLabel="Saving…"
        successLabel="Saved!"
        errorLabel="Try again"
        [resetDelay]="2500"
        [haptic]="true"
        [action]="saveAction"
      />
    </form>
  `,
})
export class ProfileFormComponent {
  private http = inject(HttpClient);

  /** Arrow function preserves `this` context when passed as an @Input */
  saveAction = async (): Promise<void> => {
    await firstValueFrom(
      this.http.put('/api/profile', { /* payload */ })
    );
  };
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Button state announced to screen readers | `aria-live="polite"` on the button; `aria-label` updates per state |
| Button is truly disabled during loading | `disabled` attribute set (not just `pointer-events: none`) |
| Disabled button cannot receive focus | Native `disabled` removes it from the tab order |
| Visual label changes match announced text | `aria-label` mirrors the displayed label slot per state |
| Spinner has no accessible text | `aria-hidden="true"` on `.btn-spinner` |
| SVG icons have no accessible text | `aria-hidden="true"` on all SVG elements |
| Reduced motion respected | `@media (prefers-reduced-motion: reduce)` disables all animations; checkmark/X appear instantly |
| Minimum touch target 44×44px | Button `height: 2.75rem` (44px); width expands with label |
| Focus ring visible for keyboard navigation | `:focus-visible` outline defined |
| Color is not the sole indicator of state change | Icon (spinner/checkmark/X) reinforces color; text label also changes |

---

## Production Pitfalls

**1. Using `pointer-events: none` instead of `disabled` to block double-submit**
`pointer-events: none` blocks mouse events but not keyboard Enter/Space activations, form submission via JavaScript, or screen reader virtual cursor clicks. Always set the native `disabled` attribute during loading. The `disabled` attribute removes the element from the tab order, prevents all activation methods, and is automatically communicated to assistive technology. `pointer-events: none` is a visual-only supplement at best.

**2. Storing the async lock in component state instead of a ref**
If you derive `isLocked` from `state === 'loading'`, there is a one-render window between the first click's Promise resolution and the state update where a second click can slip through. A `useRef`/class field flag is synchronously set before the Promise is even created, closing that window entirely. The ref does not trigger re-renders; it is purely a gatekeeping mechanism.

**3. Forgetting to cancel the reset timer on unmount**
If the user navigates away before the 2-second success dwell expires, the `setTimeout` callback fires and calls `setState` on an unmounted component. In React 18 strict mode this produces a console warning; in older versions it can cause memory leaks. Always return a cleanup function from `useEffect` (React) or call `clearTimeout` in `ngOnDestroy` (Angular).

**4. Hard-coding the `stroke-dasharray` value for the checkmark**
The correct `stroke-dasharray` value equals the actual SVG path length. For a `<polyline points="4,13 9,18 20,6">` in a 24×24 viewBox the length is approximately 22. If you change the points, the animation will either draw partially or start from an already-visible position. Use `SVGGeometryElement.getTotalLength()` at runtime, or measure once and hard-code the correct value per path.

**5. Playing the Vibration API call on desktop**
`navigator.vibrate()` is a no-op on desktop browsers (they simply return `false`) but it can throw in some browser contexts such as cross-origin iframes. Always wrap it in a try/catch and guard with `'vibrate' in navigator`. Never block the state-machine progression on a vibration API response.

**6. `stroke-dashoffset` animation replaying after re-render**
In React, if the parent re-renders while the button is in the `success` state, the CSS animation restarts from scratch because React re-creates the DOM node. Guard against this by keying the SVG node only on the transition into `success`, not on every render. In practice the simplest fix is to leave the SVG in the DOM at all times and let the `data-state` attribute control visibility — the animation re-triggers only when `data-state` switches to `"success"`, not on unrelated parent re-renders.

**7. Not resetting the `stroke-dashoffset` when leaving the success state**
CSS animations run to completion and leave the element in its final state (`forwards` fill mode). When the button resets to `idle`, the checkmark slot becomes `opacity: 0` via the transition — but the `stroke-dashoffset` remains at `0`. On the next success cycle the animation fires again from `0` to `0`, drawing instantly rather than animating. Fix: remove the animation class (or toggle `data-state` back through `idle`) so the browser resets `stroke-dashoffset` to the `stroke-dasharray` value before the next success cycle begins.

---

## Interview Angle

**Q: "Walk me through how you would build a button that shows a spinner, then a checkmark, then resets — and makes it impossible to click twice."**

A complete answer covers four layers. First, the **state machine**: model the button as a four-state enum (idle, loading, success, error) rather than multiple booleans — booleans allow impossible combinations like `isLoading && isSuccess`. Second, the **async gate**: use a ref-based flag, not derived state, so the lock is synchronous and cannot be bypassed by rapid successive clicks between React render cycles. Third, the **CSS layer**: own the visual morphing in CSS using `data-state` attribute selectors, `@keyframes` for the spinner, and `stroke-dashoffset` for the SVG draw-on; JS only sets the attribute. Fourth, the **cleanup contract**: the reset `setTimeout` handle must be cancelled on unmount or the callback will call `setState` on a dead component.

**Follow-up: "Why use `stroke-dashoffset` for the checkmark rather than just swapping an icon?"**

Swapping an icon — replacing a spinner image with a checkmark image — creates a hard cut that feels abrupt and cheap. `stroke-dashoffset` treats the checkmark path as a line being drawn in real time: the path length is set as `stroke-dasharray`, the offset starts equal to the length (so nothing is visible), and the animation drives the offset to zero so the stroke appears to draw itself. This single CSS property change costs nothing in JavaScript and produces a premium feel that signals intent — the operation is genuinely complete — rather than just a state flag flip.

**Follow-up: "When would you NOT use the Vibration API?"**

Skip it in three cases: when `prefers-reduced-motion` is active (users who opt out of motion likely also want to opt out of tactile disruption), when the button is inside a payment or destructive-action flow where a buzz might feel alarming rather than confirming, and when the vibration pattern exceeds 200ms which can feel like an error regardless of the actual outcome. Always let the user override it via a preference setting in your app.
