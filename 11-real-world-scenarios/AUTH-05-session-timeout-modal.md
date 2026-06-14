# Session Timeout Warning Modal

## The Idea

**In plain English:** When a user's session is about to expire, show a modal that counts down from, say, 120 seconds and gives two choices: "Stay logged in" (which pings the server to extend the session) or "Log out now" (which ends it immediately). If the countdown reaches zero without input, log the user out automatically. The catch is that if the user is in a different browser tab (or the tab is backgrounded), you must not expire their session — they are still active in another tab.

**Real-world analogy:** Think of a hotel room's door lock.

- The **keycard** is the session token — it only works for a fixed window.
- The **front desk calls your room ten minutes before checkout** to ask if you want to extend — that is the timeout warning modal.
- If you are **eating at the hotel restaurant** when the call comes, you are still physically at the hotel (active in another tab). The front desk should not remove your keycard just because you didn't pick up the room phone.
- If you **left the hotel entirely** and didn't respond to the call, checkout proceeds automatically — the countdown reaching zero with no interaction.
- The **"Extend stay" button** fires a keep-alive API call that resets the server-side session timer without the user needing to navigate anywhere.
- **Visibility API** is the front desk checking whether the phone in your room was even reachable (tab is visible/hidden).

The key insight: the countdown runs entirely in the browser but the authoritative session timer lives on the server. The browser must sync back to the server to extend, and must not expire sessions in background tabs.

---

## Learning Objectives

- Implement a precise countdown timer using `useInterval` in React and `RxJS interval` in Angular
- Use the `Page Visibility API` (`document.visibilitychange`) to pause the countdown when the tab is hidden
- Fire a keep-alive ping to the server when the user chooses to extend
- Handle the "extend vs logout" choice with clean state transitions
- Avoid false logouts when users have the app open in multiple tabs
- Clear all intervals and subscriptions properly on unmount/destroy

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Display the modal overlay and layout | ✅ | — |
| Animate countdown progress bar | ⚠️ | CSS animation can count down visually, but the value isn't dynamic |
| Track remaining seconds as an integer | ❌ | CSS has no numeric state |
| Pause timer when tab is hidden | ❌ | `visibilitychange` is a DOM event requiring JS |
| Fire keep-alive HTTP request on extend | ❌ | CSS cannot make network calls |
| Reset countdown after successful keep-alive | ❌ | Requires JS state mutation |
| Log out automatically at zero | ❌ | Requires JS to redirect/clear tokens |
| Coordinate across browser tabs | ❌ | Requires `BroadcastChannel` or `localStorage` events |

---

## HTML & CSS Foundation

### The Structure

```html
<!-- Modal backdrop -->
<div
  class="timeout-overlay"
  role="dialog"
  aria-modal="true"
  aria-labelledby="timeout-title"
  aria-describedby="timeout-desc"
>
  <div class="timeout-modal">
    <h2 id="timeout-title" class="timeout-title">Still there?</h2>

    <p id="timeout-desc" class="timeout-desc">
      Your session will expire in
      <strong>
        <span class="timeout-countdown" aria-live="assertive" aria-atomic="true">
          1:58
        </span>
      </strong>
      due to inactivity.
    </p>

    <!-- Countdown progress ring or bar -->
    <div class="timeout-progress" role="progressbar"
         aria-valuenow="118" aria-valuemin="0" aria-valuemax="120"
         aria-label="Time remaining">
      <div class="timeout-progress__fill" style="width: 98.3%;"></div>
    </div>

    <div class="timeout-actions">
      <button class="btn btn--primary" id="extend-btn">Stay logged in</button>
      <button class="btn btn--ghost"   id="logout-btn">Log out now</button>
    </div>
  </div>
</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --timeout-overlay-bg:   rgba(0, 0, 0, 0.6);
  --timeout-modal-bg:     #ffffff;
  --timeout-radius:       12px;
  --timeout-shadow:       0 20px 60px rgba(0, 0, 0, 0.3);
  --timeout-progress-bg:  #e5e7eb;
  --timeout-progress-fg:  #f59e0b;   /* amber — urgency without panic */
  --timeout-progress-low: #ef4444;   /* red when < 30 s */
  --timeout-transition:   0.3s ease;
}

/* ─── Overlay ─── */
.timeout-overlay {
  position: fixed;
  inset: 0;
  background: var(--timeout-overlay-bg);
  display: grid;
  place-items: center;
  z-index: 9999;
  animation: fadeIn 0.2s ease;
}

@keyframes fadeIn {
  from { opacity: 0; }
  to   { opacity: 1; }
}

/* ─── Modal card ─── */
.timeout-modal {
  background: var(--timeout-modal-bg);
  border-radius: var(--timeout-radius);
  box-shadow: var(--timeout-shadow);
  padding: 2rem;
  width: min(440px, calc(100vw - 2rem));
  text-align: center;
  animation: slideUp 0.25s cubic-bezier(0.34, 1.56, 0.64, 1);
}

@keyframes slideUp {
  from { transform: translateY(24px); opacity: 0; }
  to   { transform: translateY(0);    opacity: 1; }
}

.timeout-title {
  margin: 0 0 0.5rem;
  font-size: 1.375rem;
  font-weight: 700;
}

.timeout-desc {
  margin: 0 0 1.25rem;
  font-size: 1rem;
  color: #4b5563;
  line-height: 1.5;
}

.timeout-countdown {
  font-variant-numeric: tabular-nums;   /* prevents layout shift as digits change */
  font-size: 1.125rem;
  color: #1f2937;
}

/* ─── Progress bar ─── */
.timeout-progress {
  height: 6px;
  border-radius: 3px;
  background: var(--timeout-progress-bg);
  overflow: hidden;
  margin-bottom: 1.75rem;
}

.timeout-progress__fill {
  height: 100%;
  background: var(--timeout-progress-fg);
  border-radius: 3px;
  transition: width 1s linear, background-color 0.5s ease;
}

/* Turn red when low — JS adds this class to the modal */
.timeout-modal.is-critical .timeout-progress__fill {
  background: var(--timeout-progress-low);
}

/* ─── Buttons ─── */
.timeout-actions {
  display: flex;
  gap: 0.75rem;
  justify-content: center;
  flex-wrap: wrap;
}

.btn {
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  font-size: 0.9375rem;
  font-weight: 600;
  cursor: pointer;
  border: 2px solid transparent;
  transition: background-color 0.15s, border-color 0.15s, opacity 0.15s;
  min-width: 140px;
}

.btn--primary {
  background: #2563eb;
  color: #fff;
}
.btn--primary:hover { background: #1d4ed8; }
.btn--primary:focus-visible {
  outline: 3px solid #93c5fd;
  outline-offset: 2px;
}

.btn--ghost {
  background: transparent;
  color: #4b5563;
  border-color: #d1d5db;
}
.btn--ghost:hover { background: #f9fafb; }

/* Disabled state while keep-alive request is in-flight */
.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .timeout-overlay,
  .timeout-modal,
  .timeout-progress__fill {
    animation: none;
    transition: none;
  }
}
```

---

## React Implementation

### `useInterval` — the reliable countdown primitive

```tsx
// useInterval.ts
import { useEffect, useRef } from 'react';

/**
 * Declarative setInterval hook.
 * Pass null as delay to pause the interval.
 * Sourced from Dan Abramov's pattern — delay changes take effect on next tick.
 */
export function useInterval(callback: () => void, delay: number | null) {
  const savedCallback = useRef(callback);

  // Always run the latest callback without resetting the interval
  useEffect(() => {
    savedCallback.current = callback;
  }, [callback]);

  useEffect(() => {
    if (delay === null) return;
    const id = setInterval(() => savedCallback.current(), delay);
    return () => clearInterval(id);
  }, [delay]);
}
```

### `useSessionTimeout` — the core logic hook

```tsx
// useSessionTimeout.ts
import { useState, useEffect, useCallback, useRef } from 'react';
import { useInterval } from './useInterval';

export interface SessionTimeoutOptions {
  /** Total seconds before auto-logout */
  warningDuration: number;
  /** Called when the user confirms extend; must return a Promise */
  onKeepAlive: () => Promise<void>;
  /** Called when session expires or user chooses to log out */
  onLogout: () => void;
}

export interface SessionTimeoutState {
  secondsLeft: number;
  isVisible: boolean;
  isPinging: boolean;
  isCritical: boolean;   // < 30 seconds remaining
  extend: () => void;
  logout: () => void;
}

export function useSessionTimeout({
  warningDuration,
  onKeepAlive,
  onLogout,
}: SessionTimeoutOptions): SessionTimeoutState {
  const [secondsLeft, setSecondsLeft] = useState(warningDuration);
  const [isPinging,   setIsPinging]   = useState(false);
  const [isTabHidden, setIsTabHidden] = useState(
    () => document.visibilityState === 'hidden'
  );

  // Tick every second unless: tab is hidden OR a keep-alive is in-flight
  const shouldTick = !isTabHidden && !isPinging && secondsLeft > 0;
  useInterval(() => {
    setSecondsLeft(s => s - 1);
  }, shouldTick ? 1000 : null);

  // Auto-logout when countdown hits zero
  useEffect(() => {
    if (secondsLeft <= 0) {
      onLogout();
    }
  }, [secondsLeft, onLogout]);

  // Visibility API — pause when user switches away
  useEffect(() => {
    const handleVisibility = () => {
      setIsTabHidden(document.visibilityState === 'hidden');
    };
    document.addEventListener('visibilitychange', handleVisibility);
    return () => document.removeEventListener('visibilitychange', handleVisibility);
  }, []);

  // BroadcastChannel — if another tab extends, close this modal
  const channelRef = useRef<BroadcastChannel | null>(null);
  useEffect(() => {
    channelRef.current = new BroadcastChannel('session-timeout');
    channelRef.current.onmessage = (e: MessageEvent) => {
      if (e.data?.type === 'SESSION_EXTENDED') {
        setSecondsLeft(warningDuration);
      }
      if (e.data?.type === 'SESSION_LOGOUT') {
        onLogout();
      }
    };
    return () => channelRef.current?.close();
  }, [warningDuration, onLogout]);

  const extend = useCallback(async () => {
    if (isPinging) return;
    setIsPinging(true);
    try {
      await onKeepAlive();
      setSecondsLeft(warningDuration);
      channelRef.current?.postMessage({ type: 'SESSION_EXTENDED' });
    } catch {
      // Keep-alive failed — the server already terminated the session
      onLogout();
    } finally {
      setIsPinging(false);
    }
  }, [isPinging, onKeepAlive, warningDuration, onLogout]);

  const logout = useCallback(() => {
    channelRef.current?.postMessage({ type: 'SESSION_LOGOUT' });
    onLogout();
  }, [onLogout]);

  return {
    secondsLeft,
    isVisible: true,   // The caller controls when to mount this component
    isPinging,
    isCritical: secondsLeft <= 30,
    extend,
    logout,
  };
}
```

### `SessionTimeoutModal` — the component

```tsx
// SessionTimeoutModal.tsx
import { useCallback } from 'react';
import { useSessionTimeout } from './useSessionTimeout';

interface Props {
  warningDuration?: number;   // default 120 s
  onKeepAlive: () => Promise<void>;
  onLogout: () => void;
}

function formatTime(seconds: number): string {
  const m = Math.floor(seconds / 60);
  const s = seconds % 60;
  return `${m}:${String(s).padStart(2, '0')}`;
}

export function SessionTimeoutModal({
  warningDuration = 120,
  onKeepAlive,
  onLogout,
}: Props) {
  const { secondsLeft, isPinging, isCritical, extend, logout } =
    useSessionTimeout({ warningDuration, onKeepAlive, onLogout });

  const progressPct = (secondsLeft / warningDuration) * 100;

  // Focus the primary button when the modal mounts
  const primaryRef = useCallback((el: HTMLButtonElement | null) => {
    el?.focus();
  }, []);

  return (
    <div
      className="timeout-overlay"
      role="dialog"
      aria-modal="true"
      aria-labelledby="timeout-title"
      aria-describedby="timeout-desc"
    >
      <div className={`timeout-modal ${isCritical ? 'is-critical' : ''}`}>
        <h2 id="timeout-title" className="timeout-title">Still there?</h2>

        <p id="timeout-desc" className="timeout-desc">
          Your session will expire in{' '}
          <strong>
            <span
              className="timeout-countdown"
              aria-live="assertive"
              aria-atomic="true"
              aria-label={`${secondsLeft} seconds remaining`}
            >
              {formatTime(secondsLeft)}
            </span>
          </strong>{' '}
          due to inactivity.
        </p>

        <div
          className="timeout-progress"
          role="progressbar"
          aria-valuenow={secondsLeft}
          aria-valuemin={0}
          aria-valuemax={warningDuration}
          aria-label="Time remaining"
        >
          <div
            className="timeout-progress__fill"
            style={{ width: `${progressPct}%` }}
          />
        </div>

        <div className="timeout-actions">
          <button
            ref={primaryRef}
            className="btn btn--primary"
            onClick={extend}
            disabled={isPinging}
            aria-busy={isPinging}
          >
            {isPinging ? 'Extending...' : 'Stay logged in'}
          </button>
          <button
            className="btn btn--ghost"
            onClick={logout}
            disabled={isPinging}
          >
            Log out now
          </button>
        </div>
      </div>
    </div>
  );
}
```

### Wiring into your app

```tsx
// App.tsx (simplified)
import { useState } from 'react';
import { SessionTimeoutModal } from './SessionTimeoutModal';

const WARN_AFTER_MS = 28 * 60 * 1000;   // show modal 2 min before 30-min session expires

export function App() {
  const [showTimeout, setShowTimeout] = useState(false);

  // In production this fires from an idle timer (e.g. @idleTimer/react, or
  // a useEffect that resets on user activity events)
  useEffect(() => {
    const t = setTimeout(() => setShowTimeout(true), WARN_AFTER_MS);
    return () => clearTimeout(t);
  }, []);

  const keepAlive = async () => {
    await fetch('/api/session/keep-alive', { method: 'POST', credentials: 'include' });
    setShowTimeout(false);          // hide modal after successful ping
    // restart the outer idle timer here
  };

  const handleLogout = () => {
    fetch('/api/logout', { method: 'POST', credentials: 'include' });
    window.location.href = '/login';
  };

  return (
    <>
      {/* ... your app ... */}
      {showTimeout && (
        <SessionTimeoutModal
          warningDuration={120}
          onKeepAlive={keepAlive}
          onLogout={handleLogout}
        />
      )}
    </>
  );
}
```

---

## Angular Implementation

### Service using RxJS `interval`

```typescript
// session-timeout.service.ts
import { Injectable, signal, computed, OnDestroy, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import {
  interval, Subject, BehaviorSubject, Subscription,
  switchMap, takeUntil, take, tap, catchError, EMPTY
} from 'rxjs';
import { fromEvent, merge } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class SessionTimeoutService implements OnDestroy {
  private readonly http      = inject(HttpClient);
  private readonly WARNING_S = 120;

  // Signals drive the template
  readonly secondsLeft = signal(this.WARNING_S);
  readonly isPinging   = signal(false);
  readonly isCritical  = computed(() => this.secondsLeft() <= 30);

  private tick$        = new Subject<void>();
  private destroy$     = new Subject<void>();
  private channel      = new BroadcastChannel('session-timeout');
  private tickSub?:    Subscription;
  private visibilitySub?: Subscription;
  private paused       = false;

  constructor() {
    this.listenVisibility();
    this.listenBroadcast();
  }

  /** Call this to start the countdown (e.g. from app shell after idle threshold) */
  start() {
    this.secondsLeft.set(this.WARNING_S);
    this.tickSub?.unsubscribe();

    this.tickSub = interval(1000)
      .pipe(takeUntil(this.destroy$))
      .subscribe(() => {
        if (this.paused || this.isPinging()) return;
        const next = this.secondsLeft() - 1;
        this.secondsLeft.set(next);
        if (next <= 0) this.doLogout();
      });
  }

  extend(): void {
    if (this.isPinging()) return;
    this.isPinging.set(true);

    this.http.post('/api/session/keep-alive', {})
      .pipe(
        take(1),
        tap(() => {
          this.secondsLeft.set(this.WARNING_S);
          this.isPinging.set(false);
          this.channel.postMessage({ type: 'SESSION_EXTENDED' });
        }),
        catchError(() => {
          this.isPinging.set(false);
          this.doLogout();
          return EMPTY;
        }),
        takeUntil(this.destroy$)
      )
      .subscribe();
  }

  logout(): void {
    this.channel.postMessage({ type: 'SESSION_LOGOUT' });
    this.doLogout();
  }

  private doLogout(): void {
    this.tickSub?.unsubscribe();
    this.http.post('/api/logout', {}).pipe(take(1)).subscribe();
    window.location.href = '/login';
  }

  private listenVisibility(): void {
    fromEvent(document, 'visibilitychange')
      .pipe(takeUntil(this.destroy$))
      .subscribe(() => {
        this.paused = document.visibilityState === 'hidden';
      });
  }

  private listenBroadcast(): void {
    this.channel.onmessage = (e: MessageEvent) => {
      if (e.data?.type === 'SESSION_EXTENDED') {
        this.secondsLeft.set(this.WARNING_S);
      }
      if (e.data?.type === 'SESSION_LOGOUT') {
        this.doLogout();
      }
    };
  }

  formatTime(seconds: number): string {
    const m = Math.floor(seconds / 60);
    const s = seconds % 60;
    return `${m}:${String(s).padStart(2, '0')}`;
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
    this.tickSub?.unsubscribe();
    this.channel.close();
  }
}
```

### Standalone modal component

```typescript
// session-timeout-modal.component.ts
import { Component, inject, OnInit } from '@angular/core';
import { NgIf } from '@angular/common';
import { SessionTimeoutService } from './session-timeout.service';

@Component({
  selector: 'app-session-timeout-modal',
  standalone: true,
  imports: [NgIf],
  template: `
    <div
      class="timeout-overlay"
      role="dialog"
      aria-modal="true"
      aria-labelledby="timeout-title"
      aria-describedby="timeout-desc"
    >
      <div class="timeout-modal" [class.is-critical]="svc.isCritical()">
        <h2 id="timeout-title" class="timeout-title">Still there?</h2>

        <p id="timeout-desc" class="timeout-desc">
          Your session will expire in
          <strong>
            <span
              class="timeout-countdown"
              aria-live="assertive"
              aria-atomic="true"
              [attr.aria-label]="svc.secondsLeft() + ' seconds remaining'"
            >
              {{ svc.formatTime(svc.secondsLeft()) }}
            </span>
          </strong>
          due to inactivity.
        </p>

        <div
          class="timeout-progress"
          role="progressbar"
          [attr.aria-valuenow]="svc.secondsLeft()"
          aria-valuemin="0"
          aria-valuemax="120"
          aria-label="Time remaining"
        >
          <div
            class="timeout-progress__fill"
            [style.width.%]="progressPct()"
          ></div>
        </div>

        <div class="timeout-actions">
          <button
            #primaryBtn
            class="btn btn--primary"
            [disabled]="svc.isPinging()"
            [attr.aria-busy]="svc.isPinging()"
            (click)="svc.extend()"
          >
            {{ svc.isPinging() ? 'Extending...' : 'Stay logged in' }}
          </button>
          <button
            class="btn btn--ghost"
            [disabled]="svc.isPinging()"
            (click)="svc.logout()"
          >
            Log out now
          </button>
        </div>
      </div>
    </div>
  `,
})
export class SessionTimeoutModalComponent implements OnInit {
  readonly svc = inject(SessionTimeoutService);

  progressPct = () =>
    (this.svc.secondsLeft() / 120) * 100;

  ngOnInit(): void {
    this.svc.start();
  }
}
```

### Rendering the modal conditionally from the shell

```typescript
// app.component.ts
import { Component, signal, effect, inject, OnInit } from '@angular/core';
import { NgIf } from '@angular/common';
import { SessionTimeoutModalComponent } from './session-timeout-modal.component';

const IDLE_THRESHOLD_MS = 28 * 60 * 1000;

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [NgIf, SessionTimeoutModalComponent],
  template: `
    <!-- your app content -->
    <app-session-timeout-modal *ngIf="showTimeoutWarning()" />
  `,
})
export class AppComponent implements OnInit {
  showTimeoutWarning = signal(false);
  private idleTimer?: ReturnType<typeof setTimeout>;

  ngOnInit(): void {
    this.resetIdleTimer();
    // Reset idle on user activity
    ['click', 'keydown', 'mousemove', 'touchstart'].forEach(evt =>
      document.addEventListener(evt, () => this.resetIdleTimer(), { passive: true })
    );
  }

  private resetIdleTimer(): void {
    clearTimeout(this.idleTimer);
    this.showTimeoutWarning.set(false);
    this.idleTimer = setTimeout(
      () => this.showTimeoutWarning.set(true),
      IDLE_THRESHOLD_MS
    );
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Modal is announced immediately on appearance | `role="dialog"` + `aria-modal="true"` causes screen readers to announce it |
| Dialog has an accessible name | `aria-labelledby="timeout-title"` points to the h2 |
| Countdown announced as it changes | `aria-live="assertive"` on the seconds span — assertive is correct here because this is genuinely urgent |
| Countdown readable as a label not just digits | `aria-label="N seconds remaining"` on the span gives screen readers the full sentence |
| Progress bar is described | `role="progressbar"` with `aria-valuenow`, `aria-valuemin`, `aria-valuemax`, and `aria-label` |
| Primary button focused on mount | `ref` callback (`useCallback`) in React / `#primaryBtn` + `ngOnInit` autofocus in Angular |
| Buttons disabled and marked `aria-busy` during ping | Prevents double-submit and informs assistive tech that work is in progress |
| Focus trapped inside modal | `role="dialog"` + `aria-modal` is sufficient for most screen reader/browser combos; add explicit trap for JAWS |
| Escape key closes modal (logs out or focuses extend) | `keydown` listener; Escape in a timeout context should log out, not silently dismiss |
| Color not the only urgency indicator | Critical state changes both progress bar color AND aria-live content |
| Reduced motion respected | CSS `@media (prefers-reduced-motion: reduce)` disables all transitions and animations |

---

## Production Pitfalls

**1. `setInterval` drifts under heavy CPU load**
`setInterval(fn, 1000)` does not guarantee exactly 1 000 ms between calls. Under a heavy GC pause or CPU spike, ticks can be skipped or batched. Fix: record `Date.now()` at mount and derive `secondsLeft` from elapsed wall-clock time rather than from a decrement counter. The `useInterval` hook above still handles start/stop, but the callback reads `startTime.current` and calls `Math.max(0, warningDuration - Math.floor((Date.now() - startTime.current) / 1000))`.

**2. Tab throttling by the browser**
Modern browsers throttle `setInterval` in background tabs to ≥ 1 000 ms or even suspend it entirely. This is actually desirable for the pause behavior, but the side-effect is that when the tab becomes visible again, you may get a burst of missed ticks all at once. Fix: on `visibilitychange` to `visible`, recalculate `secondsLeft` from wall-clock elapsed time rather than trusting accumulated decrements.

**3. `BroadcastChannel` is not supported in all environments**
Safari added `BroadcastChannel` support in 15.4 (2022), so most production targets are fine. However server-side rendering environments (Next.js, Angular Universal) do not have `BroadcastChannel`. Fix: guard construction with `typeof BroadcastChannel !== 'undefined'`, and fall back to `localStorage` events (`storage` event fires cross-tab).

**4. Keep-alive request race condition**
If the user clicks "Stay logged in" twice quickly, two POST requests fire. The second one is harmless (server just extends again) but the `isPinging` flag could be set to `false` by the first response before the second one lands, allowing a third click. Fix: the `isPinging` guard is set immediately before the request and only cleared in the `finally` block. Use a single `AbortController` ref to cancel any in-flight request when a new one starts.

**5. Modal mounts on page load in SSR**
If the session warning state is derived from a cookie or token expiry timestamp and that logic runs on the server, the modal could be rendered in the initial HTML before JS hydrates. This causes a flash and mis-announces the dialog before the countdown has started. Fix: always render the modal client-side only (`'use client'` in Next.js, `isPlatformBrowser` guard in Angular).

**6. Auto-logout fires on sleeping tabs, blocking background tasks**
If the user has a long-running background tab (e.g., a file export in progress), forcibly navigating to `/login` will abort it. The `visibilitychange` pause handles the warning countdown, but the outer idle timer (the one that triggers the modal) must also check tab visibility before firing. Fix: in the idle timer callback, check `document.visibilityState === 'visible'` before setting `showTimeoutWarning` to true — sleeping tabs get a free extension until they become active.

---

## Interview Angle

**Q: "How would you implement a session timeout modal that doesn't log out users who are active in another tab?"**

The core mechanism is `BroadcastChannel`. Every tab runs the same countdown, but when any tab successfully pings the keep-alive endpoint it broadcasts a `SESSION_EXTENDED` message. All other tabs receive the message and reset their own countdowns without needing to hit the server. Separately, the `visibilitychange` API pauses the per-tab countdown while the tab is hidden — so a background tab never reaches zero and never auto-navigates to `/login`. The server session is authoritative; the browser countdown is just a UX hint, and the keep-alive call is what actually extends server-side state.

**Follow-up: "The countdown drifts by several seconds over a two-minute window — how do you fix it?"**

`setInterval` is not a wall-clock — it schedules ticks at least N ms apart, not exactly N ms apart. Under CPU pressure, GC pauses, or browser background throttling, ticks accumulate delay. The fix is to derive `secondsLeft` from `Date.now() - startTimestamp` on every tick rather than decrementing a counter. The interval still fires once per second to trigger a re-render, but the value it shows is always based on real elapsed time. This also handles the burst of ticks that fires when a throttled background tab becomes active: the single re-render after `visibilitychange` will show the correct remaining time regardless of how many ticks were skipped.
