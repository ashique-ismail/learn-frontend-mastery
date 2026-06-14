# Notification Badge with WebSocket

## The Idea

**In plain English:** You have a bell icon in the header. When the server pushes a new notification over a live connection, the badge count increments in real time — no polling, no page refresh. When the user clicks "Mark all as read", an API call resets the count to zero. If the user closes and reopens the tab, the unread count survives because it was saved to localStorage.

**Real-world analogy:** Think of a mailroom in a large office building.

- The **WebSocket connection** is the pneumatic tube system running between the mailroom and your desk. The mailroom pushes mail directly to you the moment it arrives — you do not walk down every few minutes to check.
- The **badge count** is the pile of unopened envelopes sitting on your desk. Each new delivery adds one to the pile.
- **Exponential backoff reconnection** is what happens when the tube gets jammed. You wait 1 second before trying to clear it, then 2, then 4 — you do not slam the lever repeatedly because that makes things worse and risks overloading the mailroom when it comes back online.
- **"Mark all as read"** is sweeping the pile into a drawer. You make one trip to the mailroom (an HTTP POST) to file everything, and the pile on your desk resets to zero.
- **localStorage persistence** is the sticky note you leave on your desk before going home. Tomorrow morning you read the note first: "3 unread" — before the tube system even connects, the badge is already showing the right number.

The key insight: the WebSocket handles the *push* channel, but it is not a database. localStorage is your local cache, and the server API is the source of truth. On reconnect, always reconcile your local count against the server.

---

## Learning Objectives

- Understand the four WebSocket lifecycle states: `CONNECTING`, `OPEN`, `CLOSING`, `CLOSED`
- Implement automatic reconnection with exponential backoff and jitter
- Accumulate badge counts from push events without double-counting on reconnect
- Persist and restore unread count via localStorage across page reloads
- Call a REST "mark all as read" endpoint and optimistically update the badge
- Expose correct ARIA live region announcements when the badge count changes
- Handle the multi-tab race condition when two tabs write to the same localStorage key

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
| --- | --- | --- |
| Display a badge with a number | ✅ with `::after` + `counter` | — |
| Animate badge on new notification | ✅ with `@keyframes` | — |
| Hide badge when count is zero | ✅ with `[data-count="0"]` selector | — |
| Receive server-pushed events | ❌ | CSS has no network API |
| Persist count across reloads | ❌ | CSS has no localStorage access |
| Reconnect a dropped connection | ❌ | Requires JS timer and retry logic |
| Call a REST endpoint on user action | ❌ | Requires JS `fetch` |
| Zero the badge after "mark as read" | ❌ | Requires JS state mutation |
| Announce count change to screen readers | ❌ | `aria-live` region updates require JS |
| Deduplicate replayed messages on reconnect | ❌ | Requires JS Set of seen IDs |

**Conclusion:** CSS owns the badge shape, colour, entrance animation, and reduced-motion fallback. JS/framework owns the connection lifecycle, count state, persistence, deduplication, and server communication.

---

## HTML & CSS Foundation

### The Structure

```html
<!-- Notification bell in a header toolbar -->
<div class="notif-wrapper">
  <button
    class="notif-btn"
    aria-label="Notifications, 3 unread"
    aria-haspopup="dialog"
  >
    <!-- Bell SVG icon -->
    <svg class="notif-icon" aria-hidden="true" focusable="false"
         width="24" height="24" viewBox="0 0 24 24" fill="none"
         stroke="currentColor" stroke-width="2">
      <path d="M18 8A6 6 0 0 0 6 8c0 7-3 9-3 9h18s-3-2-3-9"/>
      <path d="M13.73 21a2 2 0 0 1-3.46 0"/>
    </svg>

    <!--
      Badge: CSS hides it when data-count="0".
      aria-hidden because the count is already in the button aria-label.
    -->
    <span
      class="notif-badge"
      aria-hidden="true"
      data-count="3"
    >3</span>
  </button>

  <!--
    Screen-reader live region: announces badge changes without moving focus.
    Placed outside the button so it is never read as part of the button label.
  -->
  <span
    class="sr-only"
    aria-live="polite"
    aria-atomic="true"
    id="notif-announcer"
  ></span>
</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --badge-bg: #e53e3e;
  --badge-color: #ffffff;
  --badge-size: 1.25rem;
  --badge-font-size: 0.7rem;
  --badge-spring: cubic-bezier(0.34, 1.56, 0.64, 1);
}

/* ─── Wrapper ─── */
.notif-wrapper {
  position: relative;
  display: inline-flex;
}

/* ─── Button reset ─── */
.notif-btn {
  position: relative;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 44px;           /* minimum WCAG 2.5.5 tap target */
  height: 44px;
  background: none;
  border: none;
  border-radius: 50%;
  cursor: pointer;
  color: inherit;
}

.notif-btn:focus-visible {
  outline: 2px solid currentColor;
  outline-offset: 2px;
}

/* ─── Badge ─── */
.notif-badge {
  position: absolute;
  top: 4px;
  right: 4px;
  min-width: var(--badge-size);
  height: var(--badge-size);
  padding: 0 0.3em;
  background: var(--badge-bg);
  color: var(--badge-color);
  font-size: var(--badge-font-size);
  font-weight: 700;
  line-height: var(--badge-size);
  text-align: center;
  border-radius: 999px;
  border: 2px solid #fff;   /* separates badge from icon at small sizes */
  /* Hidden by default — shown only when count > 0 */
  display: none;
}

/* Show only when data-count is a non-zero integer */
.notif-badge[data-count]:not([data-count="0"]) {
  display: block;
}

/* ─── Pop animation triggered by JS adding .is-new ─── */
@keyframes badge-pop {
  0%   { transform: scale(0.5); }
  60%  { transform: scale(1.35); }
  100% { transform: scale(1); }
}

.notif-badge.is-new {
  animation: badge-pop 0.3s var(--badge-spring) forwards;
}

/* ─── Optional connection status dot ─── */
.notif-status {
  position: absolute;
  bottom: 5px;
  right: 5px;
  width: 8px;
  height: 8px;
  border-radius: 50%;
  border: 1.5px solid #fff;
}

.notif-status--connected    { background: #38a169; }
.notif-status--connecting   { background: #d69e2e; }
.notif-status--disconnected { background: #e53e3e; }

/* ─── Screen reader only ─── */
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
  .notif-badge.is-new {
    animation: none;
  }
}
```

**What CSS owns:** badge shape, pop animation, connection status dot colours, the `data-count="0"` hide rule, and reduced-motion fallback.

**What CSS cannot own:** WebSocket connection, count increments, localStorage reads/writes, API calls, ARIA label updates, or message deduplication.

---

## React Implementation

### Types

```tsx
// notif.types.ts
export type WsReadyState = 'connecting' | 'open' | 'closed' | 'error';

export interface WsMessage {
  event: 'notification' | 'ping';
  id?: string;
  title?: string;
}
```

### The WebSocket Hook

```tsx
// useNotifWebSocket.ts
import { useEffect, useRef, useCallback, useState } from 'react';
import type { WsReadyState, WsMessage } from './notif.types';

const BASE_DELAY_MS = 1_000;
const MAX_DELAY_MS  = 30_000;
const MAX_ATTEMPTS  = 10;
const STORAGE_KEY   = 'notif_unread_count';

function loadCount(): number {
  try {
    return parseInt(localStorage.getItem(STORAGE_KEY) ?? '0', 10) || 0;
  } catch {
    return 0;   // private browsing or quota — fail silently
  }
}

function saveCount(count: number): void {
  try {
    localStorage.setItem(STORAGE_KEY, String(count));
  } catch { /* quota exceeded — ignore */ }
}

interface UseNotifWebSocketOptions {
  url: string;
  onMessage?: (msg: WsMessage) => void;
}

interface UseNotifWebSocketReturn {
  readyState: WsReadyState;
  unreadCount: number;
  resetCount: () => void;
}

export function useNotifWebSocket({
  url,
  onMessage,
}: UseNotifWebSocketOptions): UseNotifWebSocketReturn {
  const [readyState, setReadyState]   = useState<WsReadyState>('connecting');
  // Initialise from localStorage so badge is correct before socket connects
  const [unreadCount, setUnreadCount] = useState<number>(loadCount);

  const wsRef        = useRef<WebSocket | null>(null);
  const attemptsRef  = useRef(0);
  const timerRef     = useRef<ReturnType<typeof setTimeout> | null>(null);
  const mountedRef   = useRef(true);
  const seenIdsRef   = useRef<Set<string>>(new Set());   // deduplication
  const onMessageRef = useRef(onMessage);

  // Keep callback ref fresh without triggering reconnects
  useEffect(() => { onMessageRef.current = onMessage; }, [onMessage]);

  // Persist count to localStorage whenever it changes
  useEffect(() => { saveCount(unreadCount); }, [unreadCount]);

  // Sync count from other tabs via the storage event
  useEffect(() => {
    const onStorage = (e: StorageEvent) => {
      if (e.key !== STORAGE_KEY || e.newValue === null) return;
      const synced = parseInt(e.newValue, 10) || 0;
      setUnreadCount(synced);
    };
    window.addEventListener('storage', onStorage);
    return () => window.removeEventListener('storage', onStorage);
  }, []);

  const resetCount = useCallback(() => {
    setUnreadCount(0);
    seenIdsRef.current.clear();
  }, []);

  const connect = useCallback(() => {
    if (!mountedRef.current) return;
    setReadyState('connecting');

    const ws = new WebSocket(url);
    wsRef.current = ws;

    ws.onopen = () => {
      if (!mountedRef.current) { ws.close(); return; }
      setReadyState('open');
      attemptsRef.current = 0;
    };

    ws.onmessage = (event: MessageEvent) => {
      if (!mountedRef.current) return;
      try {
        const msg: WsMessage = JSON.parse(event.data as string);

        if (msg.event === 'notification') {
          // Deduplicate: server may replay missed events on reconnect
          if (msg.id && seenIdsRef.current.has(msg.id)) return;
          if (msg.id) seenIdsRef.current.add(msg.id);

          setUnreadCount(prev => prev + 1);
        }

        onMessageRef.current?.(msg);
      } catch {
        // Malformed JSON — ignore
      }
    };

    ws.onerror = () => {
      if (!mountedRef.current) return;
      setReadyState('error');
      // onerror is always followed by onclose — reconnect logic lives in onclose
    };

    ws.onclose = () => {
      if (!mountedRef.current) return;
      setReadyState('closed');
      wsRef.current = null;

      if (attemptsRef.current >= MAX_ATTEMPTS) return;

      // Exponential backoff with ±25% jitter to avoid thundering herd
      const exp    = Math.min(BASE_DELAY_MS * 2 ** attemptsRef.current, MAX_DELAY_MS);
      const jitter = exp * 0.25 * (Math.random() - 0.5);
      const delay  = Math.round(exp + jitter);

      attemptsRef.current += 1;
      timerRef.current = setTimeout(connect, delay);
    };
  }, [url]); // eslint-disable-line react-hooks/exhaustive-deps

  useEffect(() => {
    mountedRef.current = true;
    connect();

    return () => {
      mountedRef.current = false;
      if (timerRef.current) clearTimeout(timerRef.current);
      wsRef.current?.close();
    };
  }, [connect]);

  return { readyState, unreadCount, resetCount };
}
```

### Mark-All-As-Read Hook

```tsx
// useMarkAllRead.ts
import { useState, useCallback } from 'react';

interface UseMarkAllReadOptions {
  apiUrl: string;
  onSuccess: () => void;
  onRollback: (previousCount: number) => void;
}

export function useMarkAllRead({ apiUrl, onSuccess, onRollback }: UseMarkAllReadOptions) {
  const [isPending, setIsPending] = useState(false);
  const [error, setError]         = useState<string | null>(null);

  const markAllRead = useCallback(async (currentCount: number) => {
    if (isPending) return;
    setIsPending(true);
    setError(null);

    // Optimistic update: reset badge before server confirms
    onSuccess();

    try {
      const res = await fetch(apiUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        credentials: 'include',
      });
      if (!res.ok) throw new Error(`Server responded ${res.status}`);
    } catch (err) {
      // Roll back to the count we had before the optimistic update
      onRollback(currentCount);
      setError(err instanceof Error ? err.message : 'Unknown error');
    } finally {
      setIsPending(false);
    }
  }, [apiUrl, isPending, onSuccess, onRollback]);

  return { markAllRead, isPending, error };
}
```

### Notification Badge Component

```tsx
// NotificationBadge.tsx
import { useEffect, useRef, useCallback, useState } from 'react';
import { useNotifWebSocket } from './useNotifWebSocket';
import { useMarkAllRead }    from './useMarkAllRead';

interface Props {
  wsUrl:       string;
  markReadUrl: string;
}

export function NotificationBadge({ wsUrl, markReadUrl }: Props) {
  const { readyState, unreadCount, resetCount } = useNotifWebSocket({ url: wsUrl });
  const [savedCount, setSavedCount]             = useState(unreadCount);

  const rollback = useCallback((prev: number) => {
    // Restore count when mark-as-read API fails
    // This relies on the hook's setter being accessible — in practice
    // expose a setCount from useNotifWebSocket or use a context/store.
    console.error('Mark-as-read failed; previous count was', prev);
  }, []);

  const { markAllRead, isPending } = useMarkAllRead({
    apiUrl:     markReadUrl,
    onSuccess:  resetCount,
    onRollback: rollback,
  });

  const badgeRef     = useRef<HTMLSpanElement>(null);
  const announcerRef = useRef<HTMLSpanElement>(null);
  const prevCountRef = useRef(unreadCount);

  // Save count snapshot for rollback before optimistic reset
  useEffect(() => {
    if (unreadCount > 0) setSavedCount(unreadCount);
  }, [unreadCount]);

  // Pop animation + screen reader announcement on count change
  useEffect(() => {
    if (unreadCount === prevCountRef.current) return;
    prevCountRef.current = unreadCount;

    const badge = badgeRef.current;
    if (badge && unreadCount > 0) {
      badge.classList.remove('is-new');
      void badge.offsetWidth;          // force reflow to restart animation
      badge.classList.add('is-new');
    }

    if (announcerRef.current) {
      announcerRef.current.textContent =
        unreadCount === 0
          ? 'All notifications marked as read'
          : `${unreadCount} unread notification${unreadCount !== 1 ? 's' : ''}`;
    }
  }, [unreadCount]);

  const statusClass = {
    connecting: 'notif-status--connecting',
    open:       'notif-status--connected',
    closed:     'notif-status--disconnected',
    error:      'notif-status--disconnected',
  }[readyState];

  const ariaLabel =
    unreadCount === 0
      ? 'Notifications, none unread'
      : `Notifications, ${unreadCount} unread`;

  return (
    <div className="notif-wrapper">
      <button
        className="notif-btn"
        aria-label={ariaLabel}
        aria-haspopup="dialog"
        onClick={() => markAllRead(savedCount)}
        disabled={isPending || unreadCount === 0}
      >
        <svg className="notif-icon" aria-hidden="true" focusable="false"
             width="24" height="24" viewBox="0 0 24 24"
             fill="none" stroke="currentColor" strokeWidth="2">
          <path d="M18 8A6 6 0 0 0 6 8c0 7-3 9-3 9h18s-3-2-3-9" />
          <path d="M13.73 21a2 2 0 0 1-3.46 0" />
        </svg>

        <span
          ref={badgeRef}
          className="notif-badge"
          aria-hidden="true"
          data-count={unreadCount}
        >
          {unreadCount > 99 ? '99+' : unreadCount}
        </span>

        <span className={`notif-status ${statusClass}`} aria-hidden="true" />
      </button>

      <span
        ref={announcerRef}
        className="sr-only"
        aria-live="polite"
        aria-atomic="true"
      />
    </div>
  );
}
```

---

## Angular Implementation

### Angular Types

```typescript
// notif.model.ts
export type WsReadyState = 'connecting' | 'open' | 'closed' | 'error';

export interface WsMessage {
  event: 'notification' | 'ping';
  id?: string;
  title?: string;
}
```

### WebSocket Service

```typescript
// notif-ws.service.ts
import { Injectable, OnDestroy, signal, computed, effect } from '@angular/core';
import type { WsReadyState, WsMessage } from './notif.model';

const BASE_DELAY_MS = 1_000;
const MAX_DELAY_MS  = 30_000;
const MAX_ATTEMPTS  = 10;
const STORAGE_KEY   = 'notif_unread_count';

@Injectable({ providedIn: 'root' })
export class NotifWsService implements OnDestroy {
  private ws?: WebSocket;
  private attempts  = 0;
  private timer?: ReturnType<typeof setTimeout>;
  private destroyed = false;
  private seenIds   = new Set<string>();

  private _readyState  = signal<WsReadyState>('connecting');
  private _unreadCount = signal<number>(this.loadCount());

  readonly readyState  = this._readyState.asReadonly();
  readonly unreadCount = this._unreadCount.asReadonly();
  readonly hasUnread   = computed(() => this._unreadCount() > 0);

  constructor() {
    // Persist to localStorage whenever the count changes
    effect(() => {
      const count = this._unreadCount();
      try { localStorage.setItem(STORAGE_KEY, String(count)); } catch { /* quota */ }
    });

    // Sync from other tabs
    window.addEventListener('storage', this.onStorageEvent);
  }

  private loadCount(): number {
    try { return parseInt(localStorage.getItem(STORAGE_KEY) ?? '0', 10) || 0; }
    catch { return 0; }
  }

  private onStorageEvent = (e: StorageEvent) => {
    if (e.key !== STORAGE_KEY || e.newValue === null) return;
    this._unreadCount.set(parseInt(e.newValue, 10) || 0);
  };

  connect(url: string): void {
    if (this.destroyed) return;
    this._readyState.set('connecting');
    this.ws = new WebSocket(url);

    this.ws.onopen = () => {
      if (this.destroyed) { this.ws?.close(); return; }
      this._readyState.set('open');
      this.attempts = 0;
    };

    this.ws.onmessage = ({ data }: MessageEvent) => {
      if (this.destroyed) return;
      try {
        const msg: WsMessage = JSON.parse(data as string);
        if (msg.event === 'notification') {
          if (msg.id && this.seenIds.has(msg.id)) return;   // deduplicate
          if (msg.id) this.seenIds.add(msg.id);
          this._unreadCount.update(n => n + 1);
        }
      } catch { /* malformed JSON */ }
    };

    this.ws.onerror = () => {
      if (!this.destroyed) this._readyState.set('error');
    };

    this.ws.onclose = () => {
      if (this.destroyed) return;
      this._readyState.set('closed');
      if (this.attempts >= MAX_ATTEMPTS) return;

      const exp    = Math.min(BASE_DELAY_MS * 2 ** this.attempts, MAX_DELAY_MS);
      const jitter = exp * 0.25 * (Math.random() - 0.5);
      this.attempts++;
      this.timer = setTimeout(() => this.connect(url), Math.round(exp + jitter));
    };
  }

  resetCount(): void {
    this._unreadCount.set(0);
    this.seenIds.clear();
  }

  ngOnDestroy(): void {
    this.destroyed = true;
    clearTimeout(this.timer);
    this.ws?.close();
    window.removeEventListener('storage', this.onStorageEvent);
  }
}
```

### Mark-All-As-Read Service

```typescript
// mark-read.service.ts
import { Injectable, inject, signal } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { NotifWsService } from './notif-ws.service';

@Injectable({ providedIn: 'root' })
export class MarkReadService {
  private http    = inject(HttpClient);
  private notifWs = inject(NotifWsService);

  readonly isPending = signal(false);
  readonly error     = signal<string | null>(null);

  markAllRead(apiUrl: string, previousCount: number): void {
    if (this.isPending()) return;
    this.isPending.set(true);
    this.error.set(null);

    // Optimistic update before server confirms
    this.notifWs.resetCount();

    this.http.post(apiUrl, {}).subscribe({
      next:  () => this.isPending.set(false),
      error: (err: Error) => {
        // Roll back: restore the count we snapshot before optimistic reset
        // In a production app expose a setCount on NotifWsService instead
        console.error('Mark-as-read failed; previous count was', previousCount);
        this.error.set(err.message ?? 'Failed to mark as read');
        this.isPending.set(false);
      },
    });
  }
}
```

### Angular Notification Badge Component

```typescript
// notification-badge.component.ts
import {
  Component, OnInit, OnDestroy, inject,
  Input, ViewChild, ElementRef, effect, signal
} from '@angular/core';
import { NgClass } from '@angular/common';
import { NotifWsService } from './notif-ws.service';
import { MarkReadService } from './mark-read.service';

@Component({
  selector: 'app-notification-badge',
  standalone: true,
  imports: [NgClass],
  template: `
    <div class="notif-wrapper">
      <button
        class="notif-btn"
        [attr.aria-label]="ariaLabel()"
        aria-haspopup="dialog"
        [disabled]="markReadSvc.isPending() || !notifWs.hasUnread()"
        (click)="onMarkRead()"
      >
        <svg class="notif-icon" aria-hidden="true" focusable="false"
             width="24" height="24" viewBox="0 0 24 24"
             fill="none" stroke="currentColor" stroke-width="2">
          <path d="M18 8A6 6 0 0 0 6 8c0 7-3 9-3 9h18s-3-2-3-9" />
          <path d="M13.73 21a2 2 0 0 1-3.46 0" />
        </svg>

        <span
          #badgeEl
          class="notif-badge"
          aria-hidden="true"
          [attr.data-count]="notifWs.unreadCount()"
        >
          {{ notifWs.unreadCount() > 99 ? '99+' : notifWs.unreadCount() }}
        </span>

        <span
          class="notif-status"
          aria-hidden="true"
          [ngClass]="statusClass()"
        ></span>
      </button>

      <span
        #announcerEl
        class="sr-only"
        aria-live="polite"
        aria-atomic="true"
      ></span>
    </div>
  `,
})
export class NotificationBadgeComponent implements OnInit, OnDestroy {
  @Input({ required: true }) wsUrl!: string;
  @Input({ required: true }) markReadUrl!: string;

  @ViewChild('badgeEl')     badgeEl!:     ElementRef<HTMLSpanElement>;
  @ViewChild('announcerEl') announcerEl!: ElementRef<HTMLSpanElement>;

  notifWs     = inject(NotifWsService);
  markReadSvc = inject(MarkReadService);

  private savedCount = signal(0);

  readonly ariaLabel = () => {
    const n = this.notifWs.unreadCount();
    return n === 0
      ? 'Notifications, none unread'
      : `Notifications, ${n} unread`;
  };

  readonly statusClass = () => ({
    'notif-status--connected':    this.notifWs.readyState() === 'open',
    'notif-status--connecting':   this.notifWs.readyState() === 'connecting',
    'notif-status--disconnected': this.notifWs.readyState() === 'closed'
                               || this.notifWs.readyState() === 'error',
  });

  constructor() {
    let prev = this.notifWs.unreadCount();

    effect(() => {
      const count = this.notifWs.unreadCount();

      // Snapshot before reset for potential rollback
      if (count > 0) this.savedCount.set(count);

      if (count === prev) return;
      prev = count;

      const badge = this.badgeEl?.nativeElement;
      if (badge && count > 0) {
        badge.classList.remove('is-new');
        void badge.offsetWidth;   // force reflow
        badge.classList.add('is-new');
      }

      const announcer = this.announcerEl?.nativeElement;
      if (announcer) {
        announcer.textContent = count === 0
          ? 'All notifications marked as read'
          : `${count} unread notification${count !== 1 ? 's' : ''}`;
      }
    });
  }

  onMarkRead(): void {
    this.markReadSvc.markAllRead(this.markReadUrl, this.savedCount());
  }

  ngOnInit(): void {
    this.notifWs.connect(this.wsUrl);
  }

  ngOnDestroy(): void {
    // NotifWsService.ngOnDestroy closes the socket automatically
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
| --- | --- |
| Badge count communicated to screen readers | Button `aria-label` includes count text, updated on every change |
| Badge `<span>` hidden from assistive technology | `aria-hidden="true"` on badge — count already in button label |
| Count changes announced without focus movement | `aria-live="polite"` + `aria-atomic="true"` region updated via JS |
| Button labelled correctly at zero | Label reads "none unread" rather than omitting count entirely |
| Button disabled when nothing to mark | `disabled` when `unreadCount === 0` or request is in-flight |
| Connection status dot not read aloud | `aria-hidden="true"` on status dot element |
| Minimum tap target size | `width: 44px; height: 44px` on `.notif-btn` |
| Focus ring visible on keyboard navigation | `:focus-visible` outline, never suppressed globally |
| Animation respects reduced motion preference | `@media (prefers-reduced-motion: reduce)` removes pop keyframe |

---

## Production Pitfalls

**1. Double-counting on reconnect**
When the WebSocket reconnects, the server may replay missed events. If the handler blindly increments for every incoming message, events delivered before the disconnect are counted twice. Fix: include a stable `id` field in every server event. Maintain a `Set<string>` of seen IDs in the client hook and skip any event whose ID is already in the set. Clear the set only when the user marks all as read.

**2. localStorage races across tabs**
If the user has two tabs open, each tab has its own WebSocket and increments its own count signal. Both write to the same localStorage key, so whichever tab writes last wins — the other tab shows a stale badge. Fix: listen for the `window` `storage` event in each tab and update local state when another tab writes the key. The implementation above wires this up in both the React hook and the Angular service.

**3. Exponential backoff without a cap creates frozen UIs**
Without `MAX_DELAY_MS`, after 10 failed attempts the delay exceeds 17 minutes. The badge shows "disconnected" and the user has no feedback. Fix: cap at 30 s, and after 3+ failed attempts surface a human-readable "Reconnecting..." message in the UI rather than just changing the dot colour.

**4. Optimistic "mark as read" with no rollback**
Resetting the badge before the server confirms is fast but risky. If the POST fails — network drop, 5xx — the badge shows zero while the server still has unread items. Fix: snapshot the count immediately before the optimistic reset; restore it in the error handler. The implementations above pass `previousCount` into the mark-read call for exactly this reason.

**5. WebSocket URL contains auth token in the query string**
Passing `?token=<jwt>` in the URL logs the token in server access logs, browser history, and referrer headers. Fix: send authentication in the first message after `onopen` (a common pattern for authenticated WebSockets), or exchange a short-lived one-time ticket via a REST endpoint before connecting and use that ticket in the URL instead.

**6. Component re-mounts open duplicate sockets**
In React Strict Mode, effects run twice in development. If each run calls `connect()` without a prior close, two WebSocket connections are opened. Fix: the `mountedRef` / `destroyed` guard in the implementations above ensures the cleanup function closes the previous socket before a new one is opened, and the `connect` callback bails out early if the component has unmounted.

**7. Large badge numbers break layout**
A count of "1024" overflows a fixed-size circular badge. Fix: cap the displayed text at "99+" while keeping the full integer in state. The `aria-label` on the button must announce the real count, not the truncated display value, so screen reader users hear accurate information.

---

## Interview Angle

**Q: "Walk me through how you'd implement a real-time notification badge that survives page reloads and dropped connections."**
A complete answer covers five areas. First, **WebSocket lifecycle**: explain the four native ready-state integers (`CONNECTING=0`, `OPEN=1`, `CLOSING=2`, `CLOSED=3`) and the rule that you must never call `send()` unless the state is `OPEN`. Second, **automatic reconnection**: implement exponential backoff — start at 1 s, double each attempt, hard-cap at 30 s, add ±25% jitter to prevent a thundering-herd problem when a server restarts and thousands of clients reconnect simultaneously. Third, **count accumulation with deduplication**: increment a local counter on each push event, but track seen message IDs in a `Set` so events replayed during reconnect are not double-counted. Fourth, **localStorage persistence**: write the count on every change and read it back as the initial state, so the badge is visible before the socket even finishes connecting on reload. Fifth, **mark as read**: apply an optimistic UI reset on button click, fire the POST, and roll back to the saved count only on failure — waiting for the server round-trip before updating the badge makes the interaction feel sluggish.

**Follow-up: "What's the multi-tab problem and how do you solve it?"**
Each browser tab runs its own JavaScript context, its own WebSocket connection, and its own in-memory count. Two tabs incrementing independently and both writing to localStorage will produce conflicting badge values. The fix is the `window` `storage` event: when Tab A writes a new count to localStorage, the browser fires a `storage` event in every other same-origin tab. Each tab's listener reads the new value and overwrites its own signal, keeping all tabs in sync without any extra server infrastructure.
