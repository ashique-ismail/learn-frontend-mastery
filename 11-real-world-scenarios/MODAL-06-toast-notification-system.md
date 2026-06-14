# Toast / Notification System

## The Idea

**In plain English:** A toast notification system lets your application display short, self-dismissing messages to the user — "File saved", "Network error", "3 items added to cart" — without navigating away from the current page or blocking interaction. Any part of the app can trigger a toast at any time. The system queues them, displays them in a stacked column, and removes them automatically after a timeout.

**Real-world analogy:** Think of a restaurant pass — the narrow counter between the kitchen and the dining room.

- The **kitchen (any component)** calls out "Order up!" and places the plate on the pass. It does not care which waiter picks it up.
- The **pass (the toast queue)** holds plates in arrival order (FIFO). If three orders finish simultaneously, they stack on the pass.
- The **waiters (the toast renderer)** pick up plates one at a time and carry them to the customer. If the customer waves them over (hovers), the waiter pauses before taking the empty plate back — the auto-dismiss timer is suspended.
- Once delivered, the plate disappears — the customer sees the information and the pass is clear again.

The key insight: the *trigger* (any component) and the *display* (a fixed overlay) are completely decoupled. They communicate only through a shared queue, not through props or parent-child relationships.

---

## Learning Objectives

- Understand why a singleton service or context is the right architecture for a cross-cutting concern like notifications
- Build a FIFO queue with auto-dismiss timers that pause on hover
- Render toasts in a React portal attached to `document.body` so z-index and stacking context are never a problem
- Stack multiple toasts with `translateY` and animate enter/exit smoothly
- Apply the correct ARIA roles: `role="status"` for non-urgent toasts and `role="alert"` for errors
- Avoid the common pitfalls: timer leaks, stale closures, and layout thrash during stacking

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Render a toast from inside a deeply nested component | ❌ | CSS has no mechanism to inject markup from an arbitrary point in the DOM |
| Maintain a FIFO queue of pending notifications | ❌ | CSS has no data structures or array state |
| Auto-dismiss after N milliseconds | ❌ | CSS has no timers; `animation-fill-mode: forwards` can hide an element but cannot remove it from the DOM |
| Pause the dismiss timer on hover | ❌ | CSS cannot read or mutate a timer reference |
| Deduplicate identical toasts | ❌ | Requires comparing queue contents |
| Limit the visible stack to N toasts | ❌ | Requires counting live nodes with logic |
| Announce new toasts to screen readers | ❌ | `aria-live` attribute must be present before content is inserted; orchestrating that is a JS concern |
| Animate exit (slide out / fade out before DOM removal) | ⚠️ | CSS can animate, but JS must wait for the animation to finish before removing the node |

**Conclusion:** CSS owns the visual layer — position, slide-in animation, color variants, stacking offsets. JS owns the lifecycle layer — queue management, timers, portal mounting, and ARIA orchestration.

---

## HTML & CSS Foundation

### The Structure

```html
<!-- Toast container — rendered via JS portal to document.body -->
<div
  class="toast-region"
  aria-label="Notifications"
  aria-live="polite"    <!-- overridden to assertive for role=alert toasts -->
  aria-atomic="false"   <!-- each toast is announced individually -->
>

  <!-- Individual toast — type variants: info | success | warning | error -->
  <div
    class="toast toast--success is-entering"
    role="status"
    aria-live="polite"
  >
    <span class="toast__icon" aria-hidden="true">✓</span>
    <p class="toast__message">File saved successfully.</p>
    <button
      class="toast__dismiss"
      aria-label="Dismiss notification"
    >✕</button>
  </div>

  <!-- Error toast uses role=alert for immediate interruption -->
  <div
    class="toast toast--error"
    role="alert"
  >
    <span class="toast__icon" aria-hidden="true">!</span>
    <p class="toast__message">Failed to save. Please try again.</p>
    <button class="toast__dismiss" aria-label="Dismiss notification">✕</button>
  </div>

</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --toast-width: 360px;
  --toast-gap: 0.5rem;
  --toast-radius: 6px;
  --toast-shadow: 0 4px 16px rgba(0, 0, 0, 0.14);
  --toast-duration: 0.25s;
  --toast-easing: cubic-bezier(0.4, 0, 0.2, 1);

  --toast-info-bg:    #1a73e8;
  --toast-success-bg: #188038;
  --toast-warning-bg: #e37400;
  --toast-error-bg:   #c5221f;
  --toast-text:       #ffffff;
}

/* ─── Region: fixed overlay, bottom-right corner ─── */
.toast-region {
  position: fixed;
  bottom: 1.5rem;
  right: 1.5rem;
  z-index: 9999;           /* above modals, drawers, dropdowns */
  display: flex;
  flex-direction: column-reverse; /* newest toast at the bottom */
  gap: var(--toast-gap);
  width: var(--toast-width);
  max-width: calc(100vw - 3rem);
  pointer-events: none;    /* clicks pass through the region gap */
}

/* ─── Individual toast ─── */
.toast {
  display: flex;
  align-items: flex-start;
  gap: 0.75rem;
  padding: 0.875rem 1rem;
  border-radius: var(--toast-radius);
  box-shadow: var(--toast-shadow);
  color: var(--toast-text);
  font-size: 0.9375rem;
  line-height: 1.4;
  pointer-events: all;     /* individual toasts are interactive */
  will-change: transform, opacity;
  /* Default: visible */
  transform: translateX(0);
  opacity: 1;
  transition:
    transform var(--toast-duration) var(--toast-easing),
    opacity   var(--toast-duration) var(--toast-easing);
}

/* ─── Variants ─── */
.toast--info    { background: var(--toast-info-bg);    }
.toast--success { background: var(--toast-success-bg); }
.toast--warning { background: var(--toast-warning-bg); }
.toast--error   { background: var(--toast-error-bg);   }

/* ─── Enter animation: slide in from the right ─── */
.toast.is-entering {
  transform: translateX(110%);
  opacity: 0;
}

/*
  JS removes .is-entering on the next frame, triggering the
  transition back to transform: translateX(0) / opacity: 1.
*/

/* ─── Exit animation: slide out to the right ─── */
.toast.is-exiting {
  transform: translateX(110%);
  opacity: 0;
}

/* ─── Layout within the toast ─── */
.toast__icon {
  flex-shrink: 0;
  font-size: 1rem;
  margin-top: 0.1em;
}

.toast__message {
  flex: 1;
  margin: 0;
}

.toast__dismiss {
  flex-shrink: 0;
  background: none;
  border: none;
  color: inherit;
  opacity: 0.7;
  cursor: pointer;
  font-size: 1rem;
  padding: 0;
  line-height: 1;
  transition: opacity 0.15s;
}

.toast__dismiss:hover,
.toast__dismiss:focus-visible {
  opacity: 1;
}

/* ─── Progress bar — visual timer indicator ─── */
.toast__progress {
  position: absolute;
  bottom: 0;
  left: 0;
  height: 3px;
  background: rgba(255, 255, 255, 0.5);
  border-radius: 0 0 var(--toast-radius) var(--toast-radius);
  /* Width driven by inline style, animated via CSS transition */
  transition: width linear;  /* duration set inline to match auto-dismiss delay */
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .toast,
  .toast__progress {
    transition: none;
  }
  .toast.is-entering,
  .toast.is-exiting {
    transform: translateX(0);
    opacity: 0;
  }
}
```

**What CSS owns:** slide-in/out animation, color variants, stacking layout via flexbox, progress bar drain, reduced-motion fallback.

**What CSS cannot own:** when to add/remove `.is-entering` / `.is-exiting`, timer management, portal mounting, queue ordering.

---

## React Implementation

### Types

```tsx
// toast.types.ts
export type ToastVariant = 'info' | 'success' | 'warning' | 'error';

export interface Toast {
  id: string;
  message: string;
  variant: ToastVariant;
  duration: number;        // ms — 0 means persistent (no auto-dismiss)
  dismissible: boolean;
}

export interface ToastOptions {
  message: string;
  variant?: ToastVariant;
  duration?: number;
  dismissible?: boolean;
}
```

### The Context and Queue Logic

```tsx
// ToastContext.tsx
import {
  createContext, useContext, useCallback,
  useState, useRef, useEffect, useId
} from 'react';
import { createPortal } from 'react-dom';
import type { Toast, ToastOptions, ToastVariant } from './toast.types';

const MAX_VISIBLE = 5;

interface ToastContextValue {
  show: (options: ToastOptions) => string;   // returns toast id
  dismiss: (id: string) => void;
  dismissAll: () => void;
  // Convenience helpers
  info:    (message: string, opts?: Partial<ToastOptions>) => string;
  success: (message: string, opts?: Partial<ToastOptions>) => string;
  warning: (message: string, opts?: Partial<ToastOptions>) => string;
  error:   (message: string, opts?: Partial<ToastOptions>) => string;
}

const ToastContext = createContext<ToastContextValue | null>(null);

export function useToast(): ToastContextValue {
  const ctx = useContext(ToastContext);
  if (!ctx) throw new Error('useToast must be used inside <ToastProvider>');
  return ctx;
}

// ── Provider ──────────────────────────────────────────────────
export function ToastProvider({ children }: { children: React.ReactNode }) {
  const [toasts, setToasts] = useState<Toast[]>([]);
  // Track which toasts are currently in their exit animation
  const [exiting, setExiting] = useState<Set<string>>(new Set());
  // Timer refs: map from toast id -> timer handle
  const timers = useRef<Map<string, ReturnType<typeof setTimeout>>>(new Map());
  const counter = useRef(0);

  const generateId = () => `toast-${Date.now()}-${++counter.current}`;

  // ── Dismiss: trigger exit animation, then remove from state ──
  const dismiss = useCallback((id: string) => {
    // Clear any pending auto-dismiss timer
    const timer = timers.current.get(id);
    if (timer !== undefined) {
      clearTimeout(timer);
      timers.current.delete(id);
    }

    // Start exit animation
    setExiting(prev => new Set(prev).add(id));

    // Remove from DOM after animation completes (matches --toast-duration: 0.25s)
    setTimeout(() => {
      setToasts(prev => prev.filter(t => t.id !== id));
      setExiting(prev => {
        const next = new Set(prev);
        next.delete(id);
        return next;
      });
    }, 260);
  }, []);

  const dismissAll = useCallback(() => {
    toasts.forEach(t => dismiss(t.id));
  }, [toasts, dismiss]);

  // ── Show: add to queue, schedule auto-dismiss ──
  const show = useCallback((options: ToastOptions): string => {
    const id = generateId();
    const toast: Toast = {
      id,
      message:     options.message,
      variant:     options.variant    ?? 'info',
      duration:    options.duration   ?? 4000,
      dismissible: options.dismissible ?? true,
    };

    setToasts(prev => {
      // Enforce max visible: drop oldest if over limit
      const next = [...prev, toast];
      return next.length > MAX_VISIBLE ? next.slice(next.length - MAX_VISIBLE) : next;
    });

    // Schedule auto-dismiss
    if (toast.duration > 0) {
      const timer = setTimeout(() => dismiss(id), toast.duration);
      timers.current.set(id, timer);
    }

    return id;
  }, [dismiss]);

  // Convenience helpers
  const make = (variant: ToastVariant) =>
    (message: string, opts?: Partial<ToastOptions>) =>
      show({ ...opts, message, variant });

  const info    = useCallback(make('info'),    [show]);
  const success = useCallback(make('success'), [show]);
  const warning = useCallback(make('warning'), [show]);
  const error   = useCallback(make('error'),   [show]);

  // Cleanup all timers on unmount
  useEffect(() => {
    return () => {
      timers.current.forEach(t => clearTimeout(t));
    };
  }, []);

  const value: ToastContextValue = { show, dismiss, dismissAll, info, success, warning, error };

  return (
    <ToastContext.Provider value={value}>
      {children}
      {createPortal(
        <ToastRegion
          toasts={toasts}
          exiting={exiting}
          onDismiss={dismiss}
          onMouseEnter={(id) => {
            // Pause timer on hover
            const timer = timers.current.get(id);
            if (timer !== undefined) {
              clearTimeout(timer);
              timers.current.delete(id);
            }
          }}
          onMouseLeave={(id, duration) => {
            // Resume timer on mouse leave (use remaining duration or a fixed resume time)
            if (duration > 0) {
              const timer = setTimeout(() => dismiss(id), duration / 2);
              timers.current.set(id, timer);
            }
          }}
        />,
        document.body
      )}
    </ToastContext.Provider>
  );
}

// ── Toast region (rendered in portal) ─────────────────────────
interface ToastRegionProps {
  toasts: Toast[];
  exiting: Set<string>;
  onDismiss: (id: string) => void;
  onMouseEnter: (id: string) => void;
  onMouseLeave: (id: string, duration: number) => void;
}

function ToastRegion({ toasts, exiting, onDismiss, onMouseEnter, onMouseLeave }: ToastRegionProps) {
  return (
    <div className="toast-region" aria-label="Notifications">
      {toasts.map(toast => (
        <ToastItem
          key={toast.id}
          toast={toast}
          isExiting={exiting.has(toast.id)}
          onDismiss={onDismiss}
          onMouseEnter={onMouseEnter}
          onMouseLeave={onMouseLeave}
        />
      ))}
    </div>
  );
}

// ── Single toast item ──────────────────────────────────────────
interface ToastItemProps {
  toast: Toast;
  isExiting: boolean;
  onDismiss: (id: string) => void;
  onMouseEnter: (id: string) => void;
  onMouseLeave: (id: string, duration: number) => void;
}

const ICONS: Record<ToastVariant, string> = {
  info:    'ℹ',
  success: '✓',
  warning: '⚠',
  error:   '✕',
};

function ToastItem({ toast, isExiting, onDismiss, onMouseEnter, onMouseLeave }: ToastItemProps) {
  const itemRef = useRef<HTMLDivElement>(null);

  // Trigger enter animation: mount with .is-entering, remove on next frame
  useEffect(() => {
    const el = itemRef.current;
    if (!el) return;
    el.classList.add('is-entering');
    const frame = requestAnimationFrame(() => {
      requestAnimationFrame(() => el.classList.remove('is-entering'));
    });
    return () => cancelAnimationFrame(frame);
  }, []);

  // Apply exit class
  useEffect(() => {
    if (isExiting) {
      itemRef.current?.classList.add('is-exiting');
    }
  }, [isExiting]);

  // Errors use role="alert" (assertive); everything else uses role="status" (polite)
  const role = toast.variant === 'error' ? 'alert' : 'status';
  const ariaLive = toast.variant === 'error' ? 'assertive' : 'polite';

  return (
    <div
      ref={itemRef}
      className={`toast toast--${toast.variant}`}
      role={role}
      aria-live={ariaLive}
      aria-atomic="true"
      onMouseEnter={() => onMouseEnter(toast.id)}
      onMouseLeave={() => onMouseLeave(toast.id, toast.duration)}
    >
      <span className="toast__icon" aria-hidden="true">
        {ICONS[toast.variant]}
      </span>
      <p className="toast__message">{toast.message}</p>
      {toast.dismissible && (
        <button
          className="toast__dismiss"
          aria-label="Dismiss notification"
          onClick={() => onDismiss(toast.id)}
        >
          ✕
        </button>
      )}
    </div>
  );
}
```

### Usage from Any Component

```tsx
// SaveButton.tsx — trigger a toast from deep in the tree
import { useToast } from './ToastContext';

export function SaveButton() {
  const toast = useToast();

  const handleSave = async () => {
    try {
      await saveDocument();
      toast.success('Document saved.');
    } catch (err) {
      toast.error('Save failed. Check your connection.', { duration: 0, dismissible: true });
    }
  };

  return <button onClick={handleSave}>Save</button>;
}

// main.tsx — wrap the app once at the root
import { ToastProvider } from './ToastContext';

createRoot(document.getElementById('root')!).render(
  <ToastProvider>
    <App />
  </ToastProvider>
);
```

---

## Angular Implementation

### Model

```typescript
// toast.model.ts
export type ToastVariant = 'info' | 'success' | 'warning' | 'error';

export interface Toast {
  id: string;
  message: string;
  variant: ToastVariant;
  duration: number;
  dismissible: boolean;
}
```

### Singleton Service

```typescript
// toast.service.ts
import { Injectable, signal, computed } from '@angular/core';
import type { Toast, ToastVariant } from './toast.model';

@Injectable({ providedIn: 'root' })   // singleton — same instance app-wide
export class ToastService {
  private _toasts = signal<Toast[]>([]);
  private timers  = new Map<string, ReturnType<typeof setTimeout>>();
  private counter = 0;
  private readonly MAX_VISIBLE = 5;

  // Public read-only signal consumed by the outlet component
  readonly toasts = this._toasts.asReadonly();

  private generateId(): string {
    return `toast-${Date.now()}-${++this.counter}`;
  }

  show(options: Partial<Toast> & { message: string }): string {
    const id = this.generateId();
    const toast: Toast = {
      id,
      message:     options.message,
      variant:     options.variant     ?? 'info',
      duration:    options.duration    ?? 4000,
      dismissible: options.dismissible ?? true,
    };

    this._toasts.update(prev => {
      const next = [...prev, toast];
      // Drop the oldest toast if the stack exceeds the limit
      return next.length > this.MAX_VISIBLE
        ? next.slice(next.length - this.MAX_VISIBLE)
        : next;
    });

    if (toast.duration > 0) {
      const timer = setTimeout(() => this.dismiss(id), toast.duration);
      this.timers.set(id, timer);
    }

    return id;
  }

  dismiss(id: string): void {
    const timer = this.timers.get(id);
    if (timer !== undefined) {
      clearTimeout(timer);
      this.timers.delete(id);
    }
    this._toasts.update(prev => prev.filter(t => t.id !== id));
  }

  dismissAll(): void {
    this.timers.forEach(t => clearTimeout(t));
    this.timers.clear();
    this._toasts.set([]);
  }

  pauseTimer(id: string): void {
    const timer = this.timers.get(id);
    if (timer !== undefined) {
      clearTimeout(timer);
      this.timers.delete(id);
    }
  }

  resumeTimer(id: string, duration: number): void {
    if (duration > 0) {
      // Resume with half the original duration
      const timer = setTimeout(() => this.dismiss(id), duration / 2);
      this.timers.set(id, timer);
    }
  }

  // Convenience helpers
  info   (message: string, opts?: Partial<Toast>) { return this.show({ ...opts, message, variant: 'info'    }); }
  success(message: string, opts?: Partial<Toast>) { return this.show({ ...opts, message, variant: 'success' }); }
  warning(message: string, opts?: Partial<Toast>) { return this.show({ ...opts, message, variant: 'warning' }); }
  error  (message: string, opts?: Partial<Toast>) { return this.show({ ...opts, message, variant: 'error'   }); }
}
```

### Toast Outlet Component

```typescript
// toast-outlet.component.ts
import {
  Component, inject, signal, OnDestroy
} from '@angular/core';
import { NgFor, NgIf } from '@angular/common';
import { ToastService } from './toast.service';
import type { Toast } from './toast.model';

@Component({
  selector: 'app-toast-outlet',
  standalone: true,
  imports: [NgFor, NgIf],
  // Place this in your app shell template: <app-toast-outlet />
  template: `
    <div class="toast-region" aria-label="Notifications">
      <div
        *ngFor="let toast of toastService.toasts(); trackBy: trackById"
        #toastEl
        class="toast"
        [class]="'toast toast--' + toast.variant"
        [attr.role]="toast.variant === 'error' ? 'alert' : 'status'"
        [attr.aria-live]="toast.variant === 'error' ? 'assertive' : 'polite'"
        aria-atomic="true"
        (mouseenter)="toastService.pauseTimer(toast.id)"
        (mouseleave)="toastService.resumeTimer(toast.id, toast.duration)"
      >
        <span class="toast__icon" aria-hidden="true">{{ iconFor(toast.variant) }}</span>
        <p class="toast__message">{{ toast.message }}</p>
        <button
          *ngIf="toast.dismissible"
          class="toast__dismiss"
          aria-label="Dismiss notification"
          (click)="toastService.dismiss(toast.id)"
        >✕</button>
      </div>
    </div>
  `,
})
export class ToastOutletComponent {
  toastService = inject(ToastService);

  private readonly ICONS: Record<string, string> = {
    info: 'ℹ', success: '✓', warning: '⚠', error: '✕',
  };

  iconFor(variant: string): string {
    return this.ICONS[variant] ?? '';
  }

  trackById(_: number, toast: Toast): string {
    return toast.id;
  }
}
```

### Triggering from Any Component

```typescript
// document-editor.component.ts
import { Component, inject } from '@angular/core';
import { ToastService } from './toast.service';

@Component({
  selector: 'app-document-editor',
  standalone: true,
  template: `<button (click)="save()">Save</button>`,
})
export class DocumentEditorComponent {
  private toast = inject(ToastService);

  async save() {
    try {
      await this.saveDocument();
      this.toast.success('Document saved.');
    } catch {
      this.toast.error('Save failed. Check your connection.', { duration: 0 });
    }
  }

  private saveDocument(): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, 500));
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Non-urgent toasts announced politely | `role="status"` + `aria-live="polite"` — screen reader finishes current sentence before reading |
| Error toasts interrupt immediately | `role="alert"` + `aria-live="assertive"` — screen reader stops and reads immediately |
| Each toast announced as a complete unit | `aria-atomic="true"` — reader does not announce partial updates |
| Region labelled for navigation | `aria-label="Notifications"` on the container |
| Dismiss button has accessible label | `aria-label="Dismiss notification"` — the ✕ icon is decorative |
| Icon is decorative | `aria-hidden="true"` on the icon span |
| Dismiss button reachable by keyboard | Standard `<button>` element with visible `:focus-visible` ring |
| Hovering a toast pauses the timer | `mouseenter` / `mouseleave` handlers — user can read at their own pace |
| Persistent toasts for errors | `duration: 0` disables auto-dismiss so the user must actively acknowledge |
| Animations respect reduced motion | `@media (prefers-reduced-motion: reduce)` disables all transitions |

---

## Production Pitfalls

**1. Timer leak when the component unmounts with toasts pending**
If the `ToastProvider` (React) or `ToastService` (Angular) is destroyed while timers are running, those `setTimeout` callbacks will fire against a detached state — causing either silent no-ops or "Cannot update state on unmounted component" warnings. Fix: always clear all timers in the cleanup function (`useEffect` return / `ngOnDestroy`).

**2. Stale closure over `dismiss` in the timer callback**
In React, if `dismiss` is recreated on each render and the timer captures an old version, it may call a stale function that no longer has the correct closure over `setToasts`. Fix: wrap `dismiss` in `useCallback` with stable dependencies, or use a ref pattern (`dismissRef.current = dismiss`) and call `dismissRef.current(id)` inside the timer.

**3. The `aria-live` region must exist before content is inserted**
Screen readers observe `aria-live` regions at page load. If you mount the `<div aria-live="polite">` at the same time as the first toast's text, many readers will not announce it. Fix: render the toast region container (`<div class="toast-region" aria-live="polite">`) unconditionally from app startup, even when it is empty. Only the toast items inside it are conditional.

**4. Portal and z-index fights**
Attaching the toast container to `document.body` via a portal is not just convenience — it is a correctness requirement. Any ancestor with `transform`, `filter`, `perspective`, or `will-change` properties creates a new stacking context, making `z-index: 9999` on a child inside that ancestor useless. By rendering directly on `body`, you bypass all ancestor stacking contexts entirely.

**5. Rapid-fire toasts flood the screen**
If a loop calls `toast.error()` 50 times (for example, 50 failed network requests in parallel), the user sees 50 toasts. Fix: enforce `MAX_VISIBLE` by slicing the array in the `show` method, AND consider a deduplication check: if a toast with the same message is already in the queue, increment a counter on it rather than adding a new entry.

**6. Exit animation is skipped — toast disappears instantly**
The most common mistake is calling `setToasts(prev => prev.filter(...))` immediately on dismiss. This removes the DOM node before the CSS exit transition can play. Fix: the two-phase dismiss pattern — first add `.is-exiting` (triggering the CSS animation), then remove from state after the animation duration (e.g., `setTimeout(..., 260)`). The `260ms` value should be a few milliseconds longer than the CSS `transition` duration to avoid a race condition.

**7. Mobile viewports: toasts overlap the soft keyboard**
On mobile, a fixed `bottom: 1.5rem` container can be hidden behind the virtual keyboard when an input is focused. Fix: use `env(safe-area-inset-bottom)` and detect keyboard visibility via `window.visualViewport` resize events to shift the container upward when needed.

---

## Interview Angle

**Q: "How would you design a toast notification system that can be triggered from any component in a large React application?"**

The architecture centers on a singleton — a `ToastContext` provider placed at the root of the application. Any component that calls `useToast()` gets the same `show` / `dismiss` functions regardless of its position in the tree. The queue lives in a single `useState` array at the provider level. Each `show` call appends to the queue and schedules a `setTimeout` for auto-dismiss. The provider renders the toast region via `createPortal(…, document.body)` so the output always escapes the component tree's stacking context. This is the same pattern as how dialog and tooltip libraries like Radix UI work — context for the API surface, portal for the rendering target.

**Follow-up: "Why `createPortal` rather than just rendering at the bottom of `<App>`?"**

Rendering inside `<App>` means the toast region is a child of whatever ancestor wraps `<App>`. If any ancestor has `transform` or `filter` applied (common in page transition animations), a `z-index: 9999` on the toast becomes relative to that stacking context, not the viewport — the toast can appear underneath a modal or dropdown. `createPortal` to `document.body` sidesteps this entirely because `body` has no ancestor stacking context.

**Follow-up: "When would you use `role='alert'` vs `role='status'` and why does it matter?"**

`role="status"` maps to `aria-live="polite"` — the screen reader waits for a natural pause before reading the toast aloud. This is appropriate for confirmations like "Saved" or "Item added" that are informational and not urgent. `role="alert"` maps to `aria-live="assertive"` — the screen reader interrupts whatever it is currently reading and announces the toast immediately. This is appropriate for errors that require the user's attention right now. Misusing `assertive` for all toasts is an accessibility anti-pattern: it constantly interrupts the user's reading flow for routine confirmations, which is the equivalent of a screen reader shouting "File saved!" over a sentence the user was trying to hear.
