# OAuth Popup Flow

## The Idea

**In plain English:** You want to let a user sign in with GitHub, Google, or any OAuth provider without navigating away from your app. You open a small browser popup that handles the OAuth redirect dance, and when it finishes it passes the auth result back to the original tab. Your app listens for that message, closes the popup, and proceeds — all without a full-page reload.

**Real-world analogy:** Think of a hotel concierge desk.

- Your **hotel room** is the main app tab — where you want to stay.
- The **concierge desk** is the OAuth provider (GitHub, Google).
- A **runner** is the popup window you open to go get something from the concierge.
- The runner goes to the concierge, gets your key card (the auth code/token), and slips it under your door (postMessage back to the opener).
- You do not leave your room. The runner handles everything in a separate space and reports back.
- If the runner never comes back (popup blocked, user closed it), you call the front desk after a timeout and tell them to cancel the errand.

The key insight: `window.open` gives you a handle to the popup, `postMessage` gives the popup a channel back to you, and `window.opener` inside the popup is the reference to your room door.

---

## Learning Objectives

- Open a popup with precise dimensions and features using `window.open`
- Understand why you must validate `event.origin` on every `postMessage` listener
- Detect when a popup is blocked by the browser and fall back gracefully
- Detect when the user closes the popup without completing auth (abandon detection)
- Set a hard timeout so a stale popup never leaves your app in a loading state
- Clean up message listeners and timers on component unmount to prevent memory leaks and ghost callbacks
- Implement the pattern in React (custom hook) and Angular (service + standalone component)

---

## Why CSS Alone Isn't Enough

This scenario involves no CSS-specific problem — the challenge is entirely JavaScript and browser security. The table below covers the underlying requirements and what the browser provides versus what you must write.

| Requirement | Browser API | Why you must write code around it |
|---|---|---|
| Open a popup at a specific size and position | `window.open(url, name, features)` | No promise, no built-in error for blocked popups |
| Detect if the popup was blocked | Check return value for `null` | Not thrown as an error — silently returns `null` |
| Receive the auth result from the popup | `window.addEventListener('message', ...)` | Origin must be validated manually or XSS risk |
| Detect popup abandonment (user closes it) | Poll `popup.closed` | No `close` event fires on the opener from a cross-origin popup |
| Clean up if component unmounts mid-flow | `removeEventListener` + `clearInterval` + `popup.close()` | React/Angular lifecycle does not do this automatically |
| Time out a stalled auth flow | `setTimeout` | No built-in max-wait for popup communication |

---

## HTML & CSS Foundation

The popup flow is mostly JavaScript, but the trigger and loading state require a solid HTML and CSS foundation.

### The Trigger Button and Status Indicator

```html
<!-- Primary login trigger -->
<button
  class="oauth-btn"
  type="button"
  aria-label="Sign in with GitHub"
  aria-busy="false"
  data-provider="github"
>
  <svg class="oauth-btn__icon" aria-hidden="true" focusable="false">
    <!-- provider icon path -->
  </svg>
  <span class="oauth-btn__label">Sign in with GitHub</span>
  <span class="oauth-btn__spinner" aria-hidden="true"></span>
</button>

<!-- Non-visual status for screen readers -->
<div
  class="sr-only"
  role="status"
  aria-live="polite"
  id="oauth-status"
></div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --oauth-btn-height: 44px;
  --oauth-btn-radius: 6px;
  --oauth-btn-github: #24292e;
  --oauth-btn-google: #4285f4;
  --oauth-transition: 0.2s ease;
}

/* ─── Button base ─── */
.oauth-btn {
  display: inline-flex;
  align-items: center;
  gap: 0.625rem;
  height: var(--oauth-btn-height);
  padding: 0 1.25rem;
  background: var(--oauth-btn-github);
  color: #fff;
  border: none;
  border-radius: var(--oauth-btn-radius);
  font-size: 0.9375rem;
  font-weight: 500;
  cursor: pointer;
  position: relative;
  transition: opacity var(--oauth-transition), background var(--oauth-transition);
}

.oauth-btn:hover:not([aria-busy="true"]) {
  opacity: 0.88;
}

.oauth-btn[aria-busy="true"] {
  cursor: wait;
  pointer-events: none;   /* prevent double-click while polling */
}

/* ─── Provider variants ─── */
.oauth-btn[data-provider="google"] {
  background: var(--oauth-btn-google);
}

/* ─── Icon ─── */
.oauth-btn__icon {
  width: 20px;
  height: 20px;
  flex-shrink: 0;
}

/* ─── Loading spinner (hidden by default) ─── */
.oauth-btn__spinner {
  display: none;
  width: 16px;
  height: 16px;
  border: 2px solid rgba(255, 255, 255, 0.35);
  border-top-color: #fff;
  border-radius: 50%;
  animation: oauth-spin 0.65s linear infinite;
  flex-shrink: 0;
}

.oauth-btn[aria-busy="true"] .oauth-btn__spinner {
  display: block;
}

.oauth-btn[aria-busy="true"] .oauth-btn__icon {
  display: none;
}

@keyframes oauth-spin {
  to { transform: rotate(360deg); }
}

/* ─── Blocked popup warning banner ─── */
.oauth-popup-warning {
  display: flex;
  align-items: flex-start;
  gap: 0.5rem;
  padding: 0.75rem 1rem;
  background: #fff8e1;
  border: 1px solid #f9a825;
  border-radius: var(--oauth-btn-radius);
  font-size: 0.875rem;
  color: #5d4037;
  margin-top: 0.75rem;
}

/* ─── Screen reader only utility ─── */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .oauth-btn__spinner {
    animation: none;
    border-top-color: rgba(255, 255, 255, 0.7);
  }
}
```

**What CSS owns:** button appearance and state variants (`aria-busy`), spinner visibility, reduced-motion fallback, the blocked-popup warning banner layout.

**What CSS cannot own:** detecting a blocked popup, validating `postMessage` origins, polling `popup.closed`, setting timeouts, cleaning up listeners on unmount.

---

## React Implementation

### Types

```tsx
// oauth.types.ts
export type OAuthProvider = 'github' | 'google';

export interface OAuthSuccessPayload {
  type: 'OAUTH_SUCCESS';
  code: string;       // auth code — your server exchanges this for a token
  state: string;      // CSRF nonce you generated before opening the popup
}

export interface OAuthErrorPayload {
  type: 'OAUTH_ERROR';
  error: string;
}

export type OAuthMessage = OAuthSuccessPayload | OAuthErrorPayload;

export type OAuthStatus =
  | 'idle'
  | 'pending'       // popup open, waiting
  | 'blocked'       // window.open returned null
  | 'abandoned'     // user closed popup without finishing
  | 'timeout'       // POPUP_TIMEOUT_MS elapsed
  | 'success'
  | 'error';
```

### The Custom Hook

```tsx
// useOAuthPopup.ts
import { useState, useRef, useCallback, useEffect } from 'react';
import type { OAuthMessage, OAuthProvider, OAuthStatus } from './oauth.types';

const POPUP_WIDTH      = 600;
const POPUP_HEIGHT     = 700;
const POLL_INTERVAL_MS = 500;   // how often to check popup.closed
const POPUP_TIMEOUT_MS = 5 * 60 * 1000;  // 5 minutes — hard limit

// Generates a cryptographically random state nonce for CSRF protection
function generateState(): string {
  const array = new Uint8Array(16);
  crypto.getRandomValues(array);
  return Array.from(array, b => b.toString(16).padStart(2, '0')).join('');
}

// Centres the popup relative to the opener window
function popupFeatures(width: number, height: number): string {
  const left = window.screenX + (window.outerWidth  - width)  / 2;
  const top  = window.screenY + (window.outerHeight - height) / 2;
  return [
    `width=${width}`,
    `height=${height}`,
    `left=${Math.max(0, left)}`,
    `top=${Math.max(0, top)}`,
    'toolbar=no',
    'menubar=no',
    'scrollbars=yes',
    'resizable=no',
    'status=no',
  ].join(',');
}

interface UseOAuthPopupOptions {
  provider: OAuthProvider;
  authUrl: string;          // full provider URL including client_id, redirect_uri, scope
  allowedOrigin: string;    // e.g. 'https://myapp.com' — must match redirect_uri's origin
  onSuccess: (code: string, state: string) => void;
  onError?: (reason: OAuthStatus) => void;
}

interface UseOAuthPopupReturn {
  status: OAuthStatus;
  startAuth: () => void;
  cancel: () => void;
}

export function useOAuthPopup({
  provider,
  authUrl,
  allowedOrigin,
  onSuccess,
  onError,
}: UseOAuthPopupOptions): UseOAuthPopupReturn {
  const [status, setStatus] = useState<OAuthStatus>('idle');

  const popupRef       = useRef<Window | null>(null);
  const pollTimerRef   = useRef<ReturnType<typeof setInterval> | null>(null);
  const timeoutTimerRef = useRef<ReturnType<typeof setTimeout> | null>(null);
  const stateRef       = useRef<string>('');    // CSRF nonce
  // Stable ref to onError so the cleanup closure never stales
  const onErrorRef     = useRef(onError);
  useEffect(() => { onErrorRef.current = onError; }, [onError]);

  // ── Core cleanup: remove listener, clear timers, close popup ──
  const cleanup = useCallback((closePopup = false) => {
    if (pollTimerRef.current)    clearInterval(pollTimerRef.current);
    if (timeoutTimerRef.current) clearTimeout(timeoutTimerRef.current);
    pollTimerRef.current    = null;
    timeoutTimerRef.current = null;

    if (closePopup && popupRef.current && !popupRef.current.closed) {
      popupRef.current.close();
    }
    popupRef.current = null;
  }, []);

  // ── Message handler — defined once, stable via useCallback ──
  const handleMessage = useCallback((event: MessageEvent<OAuthMessage>) => {
    // 1. Origin check — MUST come first
    if (event.origin !== allowedOrigin) return;

    // 2. Source check — only accept messages from our specific popup
    if (event.source !== popupRef.current) return;

    const { data } = event;

    if (data.type === 'OAUTH_SUCCESS') {
      // 3. CSRF check — state nonce must match what we generated
      if (data.state !== stateRef.current) {
        setStatus('error');
        cleanup(true);
        onErrorRef.current?.('error');
        return;
      }
      setStatus('success');
      cleanup(true);
      onSuccess(data.code, data.state);
    } else if (data.type === 'OAUTH_ERROR') {
      setStatus('error');
      cleanup(true);
      onErrorRef.current?.('error');
    }
  }, [allowedOrigin, cleanup, onSuccess]);

  const startAuth = useCallback(() => {
    if (status === 'pending') return;   // already in-flight

    // Generate fresh CSRF nonce
    const state = generateState();
    stateRef.current = state;

    // Append state param to the auth URL
    const url = new URL(authUrl);
    url.searchParams.set('state', state);

    // Open the popup
    const popup = window.open(
      url.toString(),
      `oauth_${provider}`,
      popupFeatures(POPUP_WIDTH, POPUP_HEIGHT),
    );

    // Detect blocked popup
    if (!popup) {
      setStatus('blocked');
      onErrorRef.current?.('blocked');
      return;
    }

    popupRef.current = popup;
    setStatus('pending');

    // Register message listener
    window.addEventListener('message', handleMessage);

    // Poll for popup closure (abandon detection)
    // Cross-origin popups do not fire a 'close' event on the opener
    pollTimerRef.current = setInterval(() => {
      if (popupRef.current?.closed) {
        // Only treat as abandoned if we haven't already resolved
        setStatus(prev => {
          if (prev === 'pending') {
            cleanup(false);
            window.removeEventListener('message', handleMessage);
            onErrorRef.current?.('abandoned');
            return 'abandoned';
          }
          return prev;
        });
      }
    }, POLL_INTERVAL_MS);

    // Hard timeout
    timeoutTimerRef.current = setTimeout(() => {
      setStatus(prev => {
        if (prev === 'pending') {
          cleanup(true);
          window.removeEventListener('message', handleMessage);
          onErrorRef.current?.('timeout');
          return 'timeout';
        }
        return prev;
      });
    }, POPUP_TIMEOUT_MS);
  }, [status, authUrl, provider, handleMessage, cleanup]);

  const cancel = useCallback(() => {
    cleanup(true);
    window.removeEventListener('message', handleMessage);
    setStatus('idle');
  }, [cleanup, handleMessage]);

  // Cleanup on unmount — prevents ghost callbacks and memory leaks
  useEffect(() => {
    return () => {
      window.removeEventListener('message', handleMessage);
      cleanup(true);
    };
  }, [handleMessage, cleanup]);

  return { status, startAuth, cancel };
}
```

### The Component

```tsx
// OAuthButton.tsx
import { useOAuthPopup } from './useOAuthPopup';
import type { OAuthProvider, OAuthStatus } from './oauth.types';

interface OAuthButtonProps {
  provider: OAuthProvider;
  authUrl: string;
  allowedOrigin: string;
  onSuccess: (code: string, state: string) => void;
}

const PROVIDER_LABEL: Record<OAuthProvider, string> = {
  github: 'GitHub',
  google: 'Google',
};

const STATUS_ANNOUNCEMENT: Record<OAuthStatus, string> = {
  idle:      '',
  pending:   'Sign-in popup opened. Complete sign in to continue.',
  blocked:   'Popup was blocked. Please allow popups and try again.',
  abandoned: 'Sign-in cancelled.',
  timeout:   'Sign-in timed out. Please try again.',
  success:   'Sign-in successful.',
  error:     'Sign-in failed. Please try again.',
};

export function OAuthButton({ provider, authUrl, allowedOrigin, onSuccess }: OAuthButtonProps) {
  const { status, startAuth, cancel } = useOAuthPopup({
    provider,
    authUrl,
    allowedOrigin,
    onSuccess,
  });

  const isPending = status === 'pending';
  const label     = PROVIDER_LABEL[provider];

  return (
    <div>
      <button
        type="button"
        className="oauth-btn"
        data-provider={provider}
        aria-busy={isPending}
        aria-label={isPending ? `Signing in with ${label}...` : `Sign in with ${label}`}
        onClick={isPending ? cancel : startAuth}
        aria-describedby="oauth-status"
      >
        {/* Icon would go here */}
        <span className="oauth-btn__label">
          {isPending ? `Signing in with ${label}...` : `Sign in with ${label}`}
        </span>
        <span className="oauth-btn__spinner" aria-hidden="true" />
      </button>

      {/* Live region announces state changes to screen readers */}
      <div
        id="oauth-status"
        className="sr-only"
        role="status"
        aria-live="polite"
        aria-atomic="true"
      >
        {STATUS_ANNOUNCEMENT[status]}
      </div>

      {/* Blocked popup fallback */}
      {status === 'blocked' && (
        <div className="oauth-popup-warning" role="alert">
          <strong>Popup blocked.</strong> Your browser prevented the sign-in popup.
          Please allow popups for this site and{' '}
          <button type="button" onClick={startAuth} style={{ textDecoration: 'underline', background: 'none', border: 'none', cursor: 'pointer', padding: 0 }}>
            try again
          </button>.
        </div>
      )}
    </div>
  );
}
```

### The Popup Callback Page

```tsx
// OAuthCallback.tsx — rendered at your redirect_uri route, e.g. /auth/callback
// This page runs inside the popup window.
import { useEffect } from 'react';
import type { OAuthMessage } from './oauth.types';

export function OAuthCallback() {
  useEffect(() => {
    // Parse the code and state from the URL the provider redirected to
    const params = new URLSearchParams(window.location.search);
    const code   = params.get('code');
    const state  = params.get('state');
    const error  = params.get('error');

    if (!window.opener) {
      // Opened directly (not via popup flow) — redirect to main app
      window.location.replace('/');
      return;
    }

    const message: OAuthMessage = error || !code || !state
      ? { type: 'OAUTH_ERROR', error: error ?? 'missing_params' }
      : { type: 'OAUTH_SUCCESS', code, state };

    // postMessage to the opener
    // Target origin MUST be your app's origin — never use '*'
    window.opener.postMessage(message, window.location.origin);

    // Close the popup after posting
    window.close();
  }, []);

  return (
    <div style={{ display: 'flex', alignItems: 'center', justifyContent: 'center', height: '100vh', fontFamily: 'sans-serif' }}>
      <p>Completing sign-in...</p>
    </div>
  );
}
```

---

## Angular Implementation

### Service

```typescript
// oauth-popup.service.ts
import { Injectable, OnDestroy, NgZone } from '@angular/core';
import { signal, computed } from '@angular/core';

export type OAuthStatus =
  | 'idle' | 'pending' | 'blocked' | 'abandoned' | 'timeout' | 'success' | 'error';

export interface OAuthResult {
  code: string;
  state: string;
}

const POPUP_WIDTH      = 600;
const POPUP_HEIGHT     = 700;
const POLL_INTERVAL_MS = 500;
const POPUP_TIMEOUT_MS = 5 * 60 * 1000;

function generateState(): string {
  const arr = new Uint8Array(16);
  crypto.getRandomValues(arr);
  return Array.from(arr, b => b.toString(16).padStart(2, '0')).join('');
}

function popupFeatures(w: number, h: number): string {
  const left = window.screenX + (window.outerWidth  - w) / 2;
  const top  = window.screenY + (window.outerHeight - h) / 2;
  return `width=${w},height=${h},left=${Math.max(0, left)},top=${Math.max(0, top)},toolbar=no,menubar=no,scrollbars=yes,resizable=no,status=no`;
}

@Injectable({ providedIn: 'root' })
export class OAuthPopupService implements OnDestroy {
  private readonly _status  = signal<OAuthStatus>('idle');
  readonly status           = this._status.asReadonly();
  readonly isPending        = computed(() => this._status() === 'pending');

  private popup:        Window | null          = null;
  private stateNonce:   string                 = '';
  private pollTimer:    ReturnType<typeof setInterval>  | null = null;
  private timeoutTimer: ReturnType<typeof setTimeout>   | null = null;
  private messageHandler: ((e: MessageEvent) => void)   | null = null;

  constructor(private ngZone: NgZone) {}

  open(authUrl: string, allowedOrigin: string): Promise<OAuthResult> {
    return new Promise((resolve, reject) => {
      if (this._status() === 'pending') {
        reject(new Error('Auth already in progress'));
        return;
      }

      this.stateNonce = generateState();
      const url = new URL(authUrl);
      url.searchParams.set('state', this.stateNonce);

      this.popup = window.open(url.toString(), 'oauth_popup', popupFeatures(POPUP_WIDTH, POPUP_HEIGHT));

      if (!this.popup) {
        this._status.set('blocked');
        reject(new Error('blocked'));
        return;
      }

      this._status.set('pending');

      // Message handler — runs outside Angular zone, use ngZone.run to re-enter
      this.messageHandler = (event: MessageEvent) => {
        if (event.origin !== allowedOrigin) return;
        if (event.source !== this.popup)    return;

        const data = event.data;

        this.ngZone.run(() => {
          if (data?.type === 'OAUTH_SUCCESS') {
            if (data.state !== this.stateNonce) {
              this.cleanup(true);
              this._status.set('error');
              reject(new Error('state_mismatch'));
              return;
            }
            this.cleanup(true);
            this._status.set('success');
            resolve({ code: data.code, state: data.state });
          } else if (data?.type === 'OAUTH_ERROR') {
            this.cleanup(true);
            this._status.set('error');
            reject(new Error(data.error ?? 'oauth_error'));
          }
        });
      };

      window.addEventListener('message', this.messageHandler);

      // Abandon detection via polling
      this.pollTimer = setInterval(() => {
        if (this.popup?.closed && this._status() === 'pending') {
          this.ngZone.run(() => {
            this.cleanup(false);
            this._status.set('abandoned');
            reject(new Error('abandoned'));
          });
        }
      }, POLL_INTERVAL_MS);

      // Hard timeout
      this.timeoutTimer = setTimeout(() => {
        if (this._status() === 'pending') {
          this.ngZone.run(() => {
            this.cleanup(true);
            this._status.set('timeout');
            reject(new Error('timeout'));
          });
        }
      }, POPUP_TIMEOUT_MS);
    });
  }

  cancel(): void {
    this.cleanup(true);
    this._status.set('idle');
  }

  reset(): void {
    this._status.set('idle');
  }

  private cleanup(closePopup: boolean): void {
    if (this.pollTimer)    clearInterval(this.pollTimer);
    if (this.timeoutTimer) clearTimeout(this.timeoutTimer);
    this.pollTimer    = null;
    this.timeoutTimer = null;

    if (this.messageHandler) {
      window.removeEventListener('message', this.messageHandler);
      this.messageHandler = null;
    }

    if (closePopup && this.popup && !this.popup.closed) {
      this.popup.close();
    }
    this.popup = null;
  }

  ngOnDestroy(): void {
    this.cleanup(true);
  }
}
```

### Standalone Component

```typescript
// oauth-button.component.ts
import { Component, Input, Output, EventEmitter, inject, OnDestroy } from '@angular/core';
import { NgIf } from '@angular/common';
import { OAuthPopupService, OAuthResult } from './oauth-popup.service';

@Component({
  selector: 'app-oauth-button',
  standalone: true,
  imports: [NgIf],
  template: `
    <div>
      <button
        type="button"
        class="oauth-btn"
        [attr.data-provider]="provider"
        [attr.aria-busy]="oauthService.isPending()"
        [attr.aria-label]="oauthService.isPending()
          ? 'Signing in with ' + providerLabel + '...'
          : 'Sign in with ' + providerLabel"
        aria-describedby="oauth-status"
        (click)="oauthService.isPending() ? oauthService.cancel() : startAuth()"
      >
        <span class="oauth-btn__label">
          {{ oauthService.isPending() ? 'Signing in with ' + providerLabel + '...' : 'Sign in with ' + providerLabel }}
        </span>
        <span class="oauth-btn__spinner" aria-hidden="true"></span>
      </button>

      <!-- Live region for screen readers -->
      <div
        id="oauth-status"
        class="sr-only"
        role="status"
        aria-live="polite"
        aria-atomic="true"
      >{{ statusAnnouncement }}</div>

      <!-- Blocked popup warning -->
      <div
        *ngIf="oauthService.status() === 'blocked'"
        class="oauth-popup-warning"
        role="alert"
      >
        <strong>Popup blocked.</strong> Allow popups for this site and
        <button type="button" (click)="startAuth()"
          style="text-decoration:underline;background:none;border:none;cursor:pointer;padding:0">
          try again
        </button>.
      </div>
    </div>
  `,
})
export class OAuthButtonComponent implements OnDestroy {
  @Input() provider: string  = 'github';
  @Input() authUrl!: string;
  @Input() allowedOrigin!: string;
  @Output() success = new EventEmitter<OAuthResult>();
  @Output() failure = new EventEmitter<string>();

  oauthService = inject(OAuthPopupService);

  get providerLabel(): string {
    const labels: Record<string, string> = { github: 'GitHub', google: 'Google' };
    return labels[this.provider] ?? this.provider;
  }

  get statusAnnouncement(): string {
    const map: Record<string, string> = {
      idle:      '',
      pending:   `Sign-in popup opened. Complete sign in to continue.`,
      blocked:   'Popup was blocked. Please allow popups and try again.',
      abandoned: 'Sign-in cancelled.',
      timeout:   'Sign-in timed out. Please try again.',
      success:   'Sign-in successful.',
      error:     'Sign-in failed. Please try again.',
    };
    return map[this.oauthService.status()] ?? '';
  }

  async startAuth(): Promise<void> {
    try {
      const result = await this.oauthService.open(this.authUrl, this.allowedOrigin);
      this.success.emit(result);
    } catch (err: unknown) {
      const reason = err instanceof Error ? err.message : 'error';
      this.failure.emit(reason);
    }
  }

  ngOnDestroy(): void {
    // Service handles its own cleanup via ngOnDestroy,
    // but cancel here so status resets if component is removed mid-flow
    this.oauthService.cancel();
  }
}
```

### OAuth Callback Component (Angular)

```typescript
// oauth-callback.component.ts — rendered at the redirect_uri route inside the popup
import { Component, OnInit } from '@angular/core';

@Component({
  selector: 'app-oauth-callback',
  standalone: true,
  template: `<p style="font-family:sans-serif;text-align:center;padding:2rem">Completing sign-in...</p>`,
})
export class OAuthCallbackComponent implements OnInit {
  ngOnInit(): void {
    const params = new URLSearchParams(window.location.search);
    const code   = params.get('code');
    const state  = params.get('state');
    const error  = params.get('error');

    if (!window.opener) {
      window.location.replace('/');
      return;
    }

    const message = error || !code || !state
      ? { type: 'OAUTH_ERROR',   error: error ?? 'missing_params' }
      : { type: 'OAUTH_SUCCESS', code, state };

    // Target origin = your app's own origin, never '*'
    window.opener.postMessage(message, window.location.origin);
    window.close();
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Button communicates loading state | `aria-busy="true"` set on button while popup is open |
| Loading state announced to screen readers | `aria-live="polite"` status region updates with a human-readable sentence |
| Blocked popup error announced immediately | `role="alert"` on the blocked warning (assertive, not polite) |
| Button label changes when pending | `aria-label` updated to "Signing in..." to match visible text |
| Cancel is reachable via keyboard | Clicking the same button while `aria-busy="true"` calls `cancel()` |
| Spinner is purely decorative | `aria-hidden="true"` on spinner element |
| Provider icon is decorative | `aria-hidden="true"` and `focusable="false"` on SVG |
| State changes after popup closes are announced | Live region text updates on `abandoned`, `timeout`, `error`, and `success` |
| Minimum tap target on button | CSS enforces `height: 44px` |
| Reduced-motion preference respected | `@media (prefers-reduced-motion: reduce)` disables spinner animation |

---

## Production Pitfalls

**1. Never use `'*'` as the `targetOrigin` in `postMessage`**
The popup page calls `window.opener.postMessage(message, targetOrigin)`. If you pass `'*'`, any page that hijacked the opener reference can intercept the auth code. Always pass the exact origin string: `window.opener.postMessage(message, 'https://yourapp.com')`. On the receiving side, always check `event.origin` before reading `event.data`.

**2. `window.open` returns `null` silently — it does not throw**
Browsers block popups opened outside a direct user gesture (e.g., inside a `setTimeout` or a `fetch` callback). Your code must check the return value immediately and branch. Never assume the popup opened successfully. Always show a fallback with instructions to allow popups.

**3. Popup blockers treat programmatic `window.open` inside async callbacks as non-user-gesture**
If you call `window.open` inside an `async` function after an `await`, the browser may block it because the user gesture (click) has expired by the time the async call resolves. Fix: call `window.open` synchronously at the start of the click handler, before any `await`. You can pass a placeholder URL and then navigate the popup: `popup.location.href = resolvedUrl`.

**4. Cross-origin popups do not fire a `close` event on the opener**
`popup.addEventListener('close', ...)` does not work across origins. The only reliable abandon detection is `setInterval` polling `popup.closed`. Keep the poll interval at 300–500ms — anything shorter is wasteful, anything longer feels unresponsive.

**5. The `message` event listener must be removed on every exit path**
Success, error, abandonment, timeout, cancel, and component unmount are all exit paths. If you forget to call `removeEventListener` on any of them, a ghost listener remains active, may fire for a future OAuth flow, and can double-resolve or corrupt state. Use a single `cleanup` function called on all paths.

**6. CSRF via state nonce is mandatory, not optional**
Without a state nonce, an attacker can craft a URL with a valid `code` parameter and trick your app into exchanging it for a token on their behalf (authorization code injection). Generate the nonce with `crypto.getRandomValues`, store it, and validate it in the `OAUTH_SUCCESS` handler before calling `onSuccess`. If they do not match, abort and treat it as an error.

**7. Pop-unders on mobile — prefer redirect flow for mobile browsers**
Mobile browsers (iOS Safari in particular) may not support popups at all, or may open them as new full tabs that the user never returns from. Detect mobile with a `matchMedia` check or `navigator.userAgent` and fall back to a full-page redirect OAuth flow. Store the current URL in `sessionStorage` before redirecting so you can return to it after the callback.

---

## Interview Angle

**Q: "How would you implement an OAuth login with a popup so the main page doesn't navigate away?"**

A strong answer covers five points: (1) Use `window.open` with explicit `width`, `height`, `left`, and `top` features to open a centered popup, checking the return value for `null` to detect a blocked popup. (2) The popup page at the `redirect_uri` calls `window.opener.postMessage({ type, code, state }, exactOrigin)` and then `window.close()`. (3) The opener registers a `message` listener that validates `event.origin === allowedOrigin` and `event.source === popup` before reading the payload — this prevents XSS via spoofed messages. (4) Abandon detection uses `setInterval` polling `popup.closed` because no `close` event fires cross-origin. A hard `setTimeout` guards against the user leaving the popup open indefinitely. (5) All listeners, intervals, and timeouts must be torn down in every exit path including component unmount.

**Follow-up: "What goes wrong in production that your happy-path demo hides?"**

Three failure modes dominate. First, `window.open` inside an `async` callback after `await` gets blocked because the user gesture has expired — open the popup synchronously before any async work. Second, iOS Safari often refuses popups entirely; detect mobile and swap to a redirect-based flow. Third, forgetting to call `removeEventListener` on the `message` event when the component unmounts leaves a ghost callback that can fire during the next user's OAuth flow in a SPA where the component re-mounts.
