# Live Feed with New-Item Banner

## The Idea

**In plain English:** You have a real-time feed — news, social posts, stock ticks, activity logs. New items arrive continuously over a WebSocket or Server-Sent Events connection. If the user is at the top of the feed, prepend new items immediately. If the user has scrolled down, don't yank them back to the top — instead show a sticky banner saying "12 new items" and let them click it to scroll up and flush the queue. When the connection drops, reconnect automatically using exponential backoff so the user never stares at a broken feed.

**Real-world analogy:** Think of a live news ticker board in an airport departure lounge.

- The **board itself** is the feed — it always shows the latest flights at the top.
- If you are standing **right in front of the board** (scrolled to top), new flights slide in at the top as they are announced — you see them immediately.
- If you have **walked to a coffee shop nearby** (scrolled down, reading older items), the board keeps updating but you don't see it. A **gate agent taps you on the shoulder** (the banner) and says "There are 8 new updates — come back to the board." You decide when to walk back.
- The banner text is not "click here to reload the page" — it is "there is already a queue of ready items waiting for you; one tap reveals them."
- If the airport's **announcement system goes silent** (WebSocket disconnect), there is a procedure: wait 1 second, try again. If that fails, wait 2 seconds. Then 4. Then 8. This is **exponential backoff** — avoiding thundering-herd reconnects when a server restarts.

The key insight: prepend vs. append is not a CSS decision. It is a product decision driven by scroll position, and it has direct consequences for DOM performance, user focus, and connection resilience.

---

## Learning Objectives

- Understand when to prepend vs. buffer new items based on scroll position
- Implement a sticky "N new items" banner that appears only when the user has scrolled down
- Wire a WebSocket connection with typed messages and full reconnect logic using exponential backoff
- Use Server-Sent Events (SSE) as an HTTP-only alternative with the same backoff pattern
- Avoid layout shift (CLS) when prepending items above the current scroll position
- Handle connection lifecycle (open, message, close, error, intentional close) correctly in both React and Angular
- Make the banner and live region accessible without screaming at screen readers on every update

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Style the sticky banner position | Yes | — |
| Show/hide banner based on scroll position | No | CSS cannot read `scrollTop` |
| Count unread items | No | CSS has no counters that react to data |
| Prepend vs. append decision | No | Requires comparing `scrollTop` to a threshold |
| Maintain scroll position when prepending above viewport | No | Requires `scrollTop` adjustment after DOM mutation |
| Open/manage WebSocket or SSE connection | No | Network I/O is JavaScript only |
| Exponential backoff on disconnect | No | Requires timers and retry state |
| Announce new item count to screen reader without spamming | No | Requires controlled `aria-live` debouncing |

**Conclusion:** CSS owns the banner shape, sticky positioning, and transition. Everything else — scroll detection, connection management, item queuing, and scroll restoration — belongs to JavaScript.

---

## HTML & CSS Foundation

### The Structure

```html
<!-- Outer container — position:relative is the stacking context for the banner -->
<div class="feed-container" id="feed-container">

  <!-- Sticky banner: hidden until JS adds .is-visible -->
  <button
    class="feed-new-banner"
    aria-live="polite"
    aria-atomic="true"
    hidden
  >
    <span class="feed-new-banner__icon" aria-hidden="true">↑</span>
    <span class="feed-new-banner__text">12 new items</span>
  </button>

  <!-- The feed list — new items prepend at the top -->
  <ul class="feed-list" id="feed-list" role="list" aria-label="Live feed">
    <li class="feed-item" data-id="123">
      <span class="feed-item__meta">2 min ago</span>
      <p class="feed-item__body">BTC/USD crossed $70,000</p>
    </li>
    <!-- more items... -->
  </ul>

  <!-- Connection status bar -->
  <div
    class="feed-status"
    role="status"
    aria-live="polite"
    aria-label="Connection status"
  >
    <!-- JS injects: Connected / Reconnecting (attempt 2)… / Disconnected -->
  </div>

</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --banner-bg: #1a73e8;
  --banner-text: #fff;
  --banner-radius: 9999px;
  --banner-shadow: 0 4px 16px rgba(0, 0, 0, 0.18);
  --feed-gap: 0.75rem;
  --item-enter-duration: 0.25s;
  --status-connected: #16a34a;
  --status-reconnecting: #d97706;
  --status-disconnected: #dc2626;
}

/* ─── Feed container ─── */
.feed-container {
  position: relative;       /* banner is positioned inside this */
  max-width: 680px;
  margin: 0 auto;
}

/* ─── New-item sticky banner ─── */
.feed-new-banner {
  position: sticky;
  top: 1rem;
  z-index: 10;
  display: flex;
  align-items: center;
  gap: 0.4rem;
  padding: 0.5rem 1.25rem;
  margin: 0 auto 0.5rem;
  width: fit-content;
  background: var(--banner-bg);
  color: var(--banner-text);
  border: none;
  border-radius: var(--banner-radius);
  box-shadow: var(--banner-shadow);
  font-size: 0.9rem;
  font-weight: 600;
  cursor: pointer;
  /* Start off-screen above viewport */
  transform: translateY(-3rem);
  opacity: 0;
  transition:
    transform 0.25s cubic-bezier(0.34, 1.56, 0.64, 1),
    opacity 0.2s ease;
}

.feed-new-banner.is-visible {
  transform: translateY(0);
  opacity: 1;
}

.feed-new-banner:hover {
  filter: brightness(1.1);
}

/* ─── Feed list ─── */
.feed-list {
  display: flex;
  flex-direction: column;
  gap: var(--feed-gap);
  list-style: none;
  padding: 0;
  margin: 0;
}

/* ─── Feed item ─── */
.feed-item {
  background: #fff;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  padding: 1rem;
  animation: none;
}

.feed-item.is-new {
  animation: item-enter var(--item-enter-duration) ease both;
}

@keyframes item-enter {
  from {
    opacity: 0;
    transform: translateY(-12px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.feed-item__meta {
  font-size: 0.75rem;
  color: #6b7280;
  margin-bottom: 0.25rem;
  display: block;
}

.feed-item__body {
  margin: 0;
  line-height: 1.5;
}

/* ─── Connection status bar ─── */
.feed-status {
  display: flex;
  align-items: center;
  gap: 0.4rem;
  font-size: 0.75rem;
  padding: 0.35rem 0;
  color: #6b7280;
}

.feed-status::before {
  content: '';
  display: inline-block;
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: var(--status-disconnected);
}

.feed-status[data-state="connected"]::before    { background: var(--status-connected);    }
.feed-status[data-state="reconnecting"]::before { background: var(--status-reconnecting); }
.feed-status[data-state="disconnected"]::before { background: var(--status-disconnected); }

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .feed-new-banner,
  .feed-item.is-new {
    transition: none;
    animation: none;
  }
}
```

**What CSS owns:** sticky banner position and bounce-in animation, item enter animation, connection status dot color, reduced-motion fallback.

**What CSS cannot own:** scroll position tracking, item count, WebSocket lifecycle, backoff timers, prepend-vs-buffer decision.

---

## React Implementation

### Types

```tsx
// feed.types.ts
export interface FeedItem {
  id: string;
  timestamp: string;   // ISO-8601
  body: string;
  author?: string;
}

export type ConnectionState = 'connecting' | 'connected' | 'reconnecting' | 'disconnected';
```

### WebSocket Hook with Exponential Backoff

```tsx
// useWebSocketFeed.ts
import { useEffect, useRef, useCallback, useState } from 'react';
import type { FeedItem, ConnectionState } from './feed.types';

const BASE_DELAY_MS  = 1_000;
const MAX_DELAY_MS   = 30_000;
const MAX_ATTEMPTS   = 8;

interface Options {
  url: string;
  onMessage: (item: FeedItem) => void;
}

export function useWebSocketFeed({ url, onMessage }: Options) {
  const [state, setState]   = useState<ConnectionState>('connecting');
  const [attempt, setAttempt] = useState(0);

  const wsRef        = useRef<WebSocket | null>(null);
  const timerRef     = useRef<ReturnType<typeof setTimeout> | null>(null);
  const intentional  = useRef(false);         // set true on explicit teardown
  const onMessageRef = useRef(onMessage);
  onMessageRef.current = onMessage;           // always fresh, no re-subscription

  const connect = useCallback((attemptNum: number) => {
    if (intentional.current) return;

    setState(attemptNum === 0 ? 'connecting' : 'reconnecting');
    setAttempt(attemptNum);

    const ws = new WebSocket(url);
    wsRef.current = ws;

    ws.onopen = () => {
      setState('connected');
      setAttempt(0);
    };

    ws.onmessage = (event: MessageEvent) => {
      try {
        const item: FeedItem = JSON.parse(event.data as string);
        onMessageRef.current(item);
      } catch {
        console.warn('[feed] unparseable message', event.data);
      }
    };

    ws.onclose = () => {
      wsRef.current = null;
      if (intentional.current) {
        setState('disconnected');
        return;
      }
      const nextAttempt = attemptNum + 1;
      if (nextAttempt > MAX_ATTEMPTS) {
        setState('disconnected');
        return;
      }
      // Exponential backoff: 1s, 2s, 4s, 8s … capped at 30s
      const delay = Math.min(BASE_DELAY_MS * 2 ** attemptNum, MAX_DELAY_MS);
      setState('reconnecting');
      timerRef.current = setTimeout(() => connect(nextAttempt), delay);
    };

    ws.onerror = () => {
      // onerror always precedes onclose; let onclose drive reconnect logic
      console.warn('[feed] WebSocket error');
    };
  }, [url]);

  useEffect(() => {
    intentional.current = false;
    connect(0);

    return () => {
      intentional.current = true;
      if (timerRef.current) clearTimeout(timerRef.current);
      wsRef.current?.close(1000, 'component unmount');
    };
  }, [connect]);

  return { connectionState: state, reconnectAttempt: attempt };
}
```

### SSE Alternative Hook

```tsx
// useSSEFeed.ts — drop-in replacement for environments without WebSocket
import { useEffect, useRef, useState } from 'react';
import type { FeedItem, ConnectionState } from './feed.types';

const BASE_DELAY_MS = 1_000;
const MAX_DELAY_MS  = 30_000;

export function useSSEFeed(url: string, onMessage: (item: FeedItem) => void) {
  const [state, setState] = useState<ConnectionState>('connecting');
  const onMessageRef = useRef(onMessage);
  onMessageRef.current = onMessage;
  const intentional = useRef(false);
  const timerRef    = useRef<ReturnType<typeof setTimeout> | null>(null);

  useEffect(() => {
    let attempt = 0;

    function connect() {
      if (intentional.current) return;
      setState(attempt === 0 ? 'connecting' : 'reconnecting');

      const es = new EventSource(url);

      es.onopen = () => {
        setState('connected');
        attempt = 0;
      };

      es.addEventListener('feed-item', (e: MessageEvent) => {
        try {
          const item: FeedItem = JSON.parse(e.data as string);
          onMessageRef.current(item);
        } catch {
          console.warn('[sse] unparseable event', e.data);
        }
      });

      es.onerror = () => {
        // Always close before scheduling own retry — prevents double reconnect
        es.close();
        if (intentional.current) { setState('disconnected'); return; }
        attempt++;
        const delay = Math.min(BASE_DELAY_MS * 2 ** (attempt - 1), MAX_DELAY_MS);
        timerRef.current = setTimeout(connect, delay);
      };
    }

    connect();

    return () => {
      intentional.current = true;
      if (timerRef.current) clearTimeout(timerRef.current);
    };
  }, [url]);

  return { connectionState: state };
}
```

### Scroll-Aware Feed Hook

```tsx
// useFeedScroll.ts
import { useRef, useState, useCallback, useEffect } from 'react';
import type { FeedItem } from './feed.types';

const SCROLL_THRESHOLD = 120;   // px from top — within this distance, auto-prepend

export function useFeedScroll() {
  const listRef                     = useRef<HTMLUListElement>(null);
  const [queue, setQueue]           = useState<FeedItem[]>([]);
  const [visible, setVisible]       = useState<FeedItem[]>([]);
  const [lastNewId, setLastNewId]   = useState<string | null>(null);
  const isAtTop                     = useRef(true);

  // Track whether user is near the top of the page
  useEffect(() => {
    const onScroll = () => {
      isAtTop.current = window.scrollY <= SCROLL_THRESHOLD;
    };
    window.addEventListener('scroll', onScroll, { passive: true });
    return () => window.removeEventListener('scroll', onScroll);
  }, []);

  // Called by the WebSocket/SSE hook on each new item
  const receiveItem = useCallback((item: FeedItem) => {
    if (isAtTop.current) {
      setLastNewId(item.id);
      setVisible(prev => [item, ...prev]);
    } else {
      setQueue(prev => [item, ...prev]);
    }
  }, []);

  // Called when user clicks the banner
  const flushQueue = useCallback(() => {
    if (queue.length === 0) return;
    setVisible(prev => [...queue, ...prev]);
    setQueue([]);
    window.scrollTo({ top: 0, behavior: 'smooth' });
  }, [queue]);

  return { listRef, visible, queue, lastNewId, receiveItem, flushQueue };
}
```

### Full LiveFeed Component

```tsx
// LiveFeed.tsx
import { useCallback } from 'react';
import { useWebSocketFeed } from './useWebSocketFeed';
import { useFeedScroll }    from './useFeedScroll';
import type { FeedItem }    from './feed.types';

interface Props {
  wsUrl: string;
}

export function LiveFeed({ wsUrl }: Props) {
  const { listRef, visible, queue, lastNewId, receiveItem, flushQueue } =
    useFeedScroll();

  const handleMessage = useCallback((item: FeedItem) => {
    receiveItem(item);
  }, [receiveItem]);

  const { connectionState, reconnectAttempt } = useWebSocketFeed({
    url: wsUrl,
    onMessage: handleMessage,
  });

  const statusLabel =
    connectionState === 'connected'    ? 'Connected' :
    connectionState === 'reconnecting' ? `Reconnecting… (attempt ${reconnectAttempt})` :
    connectionState === 'connecting'   ? 'Connecting…' :
    'Disconnected';

  return (
    <div className="feed-container">

      {/* Banner — only visible when items are queued */}
      <button
        className={`feed-new-banner ${queue.length > 0 ? 'is-visible' : ''}`}
        aria-hidden={queue.length === 0}
        tabIndex={queue.length === 0 ? -1 : 0}
        onClick={flushQueue}
      >
        <span className="feed-new-banner__icon" aria-hidden="true">↑</span>
        <span className="feed-new-banner__text">
          {queue.length} new {queue.length === 1 ? 'item' : 'items'}
        </span>
      </button>

      {/* Live feed list */}
      <ul
        ref={listRef}
        className="feed-list"
        role="list"
        aria-label="Live feed"
        aria-live="off"   /* we manage announcements manually — no per-item spam */
      >
        {visible.map((item) => (
          <li
            key={item.id}
            className={`feed-item ${item.id === lastNewId ? 'is-new' : ''}`}
            data-id={item.id}
          >
            <span className="feed-item__meta">
              {new Date(item.timestamp).toLocaleTimeString()}
              {item.author ? ` · ${item.author}` : ''}
            </span>
            <p className="feed-item__body">{item.body}</p>
          </li>
        ))}
      </ul>

      {/* Connection status */}
      <div
        className="feed-status"
        role="status"
        data-state={connectionState}
      >
        {statusLabel}
      </div>

      {/* Screen reader announcement for queued items — separate from the list */}
      <div className="sr-only" aria-live="polite" aria-atomic="true">
        {queue.length > 0
          ? `${queue.length} new ${queue.length === 1 ? 'item' : 'items'} available. Activate the banner to view.`
          : ''}
      </div>

    </div>
  );
}
```

---

## Angular Implementation

### Model and Service

```typescript
// feed.model.ts
export interface FeedItem {
  id: string;
  timestamp: string;
  body: string;
  author?: string;
}

export type ConnectionState = 'connecting' | 'connected' | 'reconnecting' | 'disconnected';
```

```typescript
// feed.service.ts
import { Injectable, signal, OnDestroy } from '@angular/core';
import { FeedItem, ConnectionState } from './feed.model';

const BASE_DELAY_MS = 1_000;
const MAX_DELAY_MS  = 30_000;
const MAX_ATTEMPTS  = 8;

@Injectable({ providedIn: 'root' })
export class FeedService implements OnDestroy {
  // ── Public signals ──────────────────────────────────────────────
  readonly connectionState  = signal<ConnectionState>('connecting');
  readonly reconnectAttempt = signal(0);

  // Raw item stream — components choose prepend vs. queue
  private readonly _incoming = signal<FeedItem | null>(null);
  readonly latestItem = this._incoming.asReadonly();

  // ── Private ─────────────────────────────────────────────────────
  private ws: WebSocket | null = null;
  private timer: ReturnType<typeof setTimeout> | null = null;
  private intentional = false;
  private url = '';

  connect(url: string): void {
    this.url = url;
    this.intentional = false;
    this._open(0);
  }

  private _open(attempt: number): void {
    if (this.intentional) return;

    this.connectionState.set(attempt === 0 ? 'connecting' : 'reconnecting');
    this.reconnectAttempt.set(attempt);

    const ws = new WebSocket(this.url);
    this.ws = ws;

    ws.onopen = () => {
      this.connectionState.set('connected');
      this.reconnectAttempt.set(0);
    };

    ws.onmessage = (event: MessageEvent) => {
      try {
        const item: FeedItem = JSON.parse(event.data as string);
        this._incoming.set(item);
      } catch {
        console.warn('[feed] unparseable message', event.data);
      }
    };

    ws.onclose = () => {
      this.ws = null;
      if (this.intentional) { this.connectionState.set('disconnected'); return; }
      const next = attempt + 1;
      if (next > MAX_ATTEMPTS) { this.connectionState.set('disconnected'); return; }
      const delay = Math.min(BASE_DELAY_MS * 2 ** attempt, MAX_DELAY_MS);
      this.connectionState.set('reconnecting');
      this.timer = setTimeout(() => this._open(next), delay);
    };

    ws.onerror = () => console.warn('[feed] WebSocket error');
  }

  disconnect(): void {
    this.intentional = true;
    if (this.timer) clearTimeout(this.timer);
    this.ws?.close(1000, 'intentional');
  }

  ngOnDestroy(): void {
    this.disconnect();
  }
}
```

### Live Feed Component

```typescript
// live-feed.component.ts
import {
  Component, OnInit, OnDestroy, inject,
  signal, computed, effect
} from '@angular/core';
import { NgFor, NgIf, DatePipe } from '@angular/common';
import { FeedService } from './feed.service';
import { FeedItem } from './feed.model';

const SCROLL_THRESHOLD = 120;

@Component({
  selector: 'app-live-feed',
  standalone: true,
  imports: [NgFor, NgIf, DatePipe],
  template: `
    <div class="feed-container">

      <!-- New-item banner -->
      <button
        class="feed-new-banner"
        [class.is-visible]="queue().length > 0"
        [attr.aria-hidden]="queue().length === 0 ? 'true' : null"
        [tabIndex]="queue().length === 0 ? -1 : 0"
        (click)="flushQueue()"
      >
        <span class="feed-new-banner__icon" aria-hidden="true">↑</span>
        <span class="feed-new-banner__text">
          {{ queue().length }} new {{ queue().length === 1 ? 'item' : 'items' }}
        </span>
      </button>

      <!-- Feed list -->
      <ul
        class="feed-list"
        role="list"
        aria-label="Live feed"
        aria-live="off"
      >
        <li
          *ngFor="let item of visible(); trackBy: trackById"
          class="feed-item"
          [class.is-new]="item.id === lastNewId()"
          [attr.data-id]="item.id"
        >
          <span class="feed-item__meta">
            {{ item.timestamp | date:'shortTime' }}
            <ng-container *ngIf="item.author"> · {{ item.author }}</ng-container>
          </span>
          <p class="feed-item__body">{{ item.body }}</p>
        </li>
      </ul>

      <!-- Connection status -->
      <div
        class="feed-status"
        role="status"
        [attr.data-state]="feedService.connectionState()"
      >
        {{ statusLabel() }}
      </div>

      <!-- SR announcement -->
      <div class="sr-only" aria-live="polite" aria-atomic="true">
        {{ srAnnouncement() }}
      </div>

    </div>
  `,
})
export class LiveFeedComponent implements OnInit, OnDestroy {
  feedService = inject(FeedService);

  // ── State ─────────────────────────────────────────────────────────
  readonly visible    = signal<FeedItem[]>([]);
  readonly queue      = signal<FeedItem[]>([]);
  readonly lastNewId  = signal<string | null>(null);
  private isAtTop     = true;
  private scrollFn?: () => void;

  readonly statusLabel = computed(() => {
    const s = this.feedService.connectionState();
    const a = this.feedService.reconnectAttempt();
    if (s === 'connected')    return 'Connected';
    if (s === 'reconnecting') return `Reconnecting… (attempt ${a})`;
    if (s === 'connecting')   return 'Connecting…';
    return 'Disconnected';
  });

  readonly srAnnouncement = computed(() => {
    const n = this.queue().length;
    return n > 0
      ? `${n} new ${n === 1 ? 'item' : 'items'} available. Activate the banner to view.`
      : '';
  });

  constructor() {
    // React to each new WebSocket item arriving via the service signal
    effect(() => {
      const item = this.feedService.latestItem();
      if (!item) return;
      if (this.isAtTop) {
        this.lastNewId.set(item.id);
        this.visible.update(v => [item, ...v]);
      } else {
        this.queue.update(q => [item, ...q]);
      }
    });
  }

  ngOnInit(): void {
    this.feedService.connect('wss://your-feed-endpoint/stream');

    this.scrollFn = () => {
      this.isAtTop = window.scrollY <= SCROLL_THRESHOLD;
    };
    window.addEventListener('scroll', this.scrollFn, { passive: true });
  }

  flushQueue(): void {
    this.visible.update(v => [...this.queue(), ...v]);
    this.queue.set([]);
    window.scrollTo({ top: 0, behavior: 'smooth' });
  }

  trackById(_: number, item: FeedItem): string {
    return item.id;
  }

  ngOnDestroy(): void {
    this.feedService.disconnect();
    if (this.scrollFn) window.removeEventListener('scroll', this.scrollFn);
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| New items do not spam screen readers on every arrival | `aria-live="off"` on the list; announcements go through a separate controlled region |
| SR announcement for queued items is meaningful | Dedicated `sr-only` div with `aria-live="polite"` and `aria-atomic="true"` reads "N new items available" |
| Banner is keyboard-reachable when visible | `tabIndex={0}` when `queue.length > 0`; `-1` otherwise so it is skipped when hidden |
| Banner is hidden from AT when queue is empty | `aria-hidden="true"` added when count is 0 |
| Connection status is announced on change | `role="status"` with implicit `aria-live="polite"` semantics |
| "Reconnecting" state does not flood SR | `role="status"` coalesces rapid updates; AT announces after a brief pause |
| After flush, keyboard focus is not lost | Banner click keeps focus on the button; screen reader re-reads the live region |
| Item timestamps are human-readable | ISO strings formatted via `toLocaleTimeString()` / Angular `DatePipe`, not raw epoch values |
| Feed items have sufficient color contrast | `feed-item__meta` color `#6b7280` meets 4.5:1 on white backgrounds |
| Minimum tap target for banner | Padding produces >= 44px height; enforced in CSS |

---

## Production Pitfalls

**1. Layout shift (CLS) when prepending above the viewport**
When the user has scrolled down and you programmatically prepend items, the browser shifts existing content down, breaking the reading position. Fix: track `scrollTop` with a passive scroll listener and only prepend immediately if the user is within a threshold (120px) of the top. If they are further down, buffer the item instead — no DOM mutation occurs, so no shift.

**2. Memory leak from an unbounded item array**
A feed running for hours accumulates thousands of items, exhausting memory and making every render cycle slower. Fix: cap `visible` at a maximum (e.g., 200 items). When trimming, remove from the tail (oldest). Users rarely scroll that far down; provide a "load older" button or virtualized scroll for historical access.

**3. Exponential backoff must include jitter**
Without jitter, every client that disconnects at the same moment (server restart, deploy) will reconnect at exactly `2^n` seconds, creating synchronized storms that overwhelm the revived server. Fix: add random jitter to the delay: `delay = BASE * 2^n + Math.random() * 1000`. This spreads reconnect attempts across a window instead of piling them up simultaneously.

**4. WebSocket `onmessage` fires after component unmount**
If the component unmounts mid-flight and `onmessage` fires before the socket closes, it calls `setState` or writes to a signal on a torn-down component tree. Fix: the `intentional` flag in both implementations guards all state writes inside `onmessage` and `onclose`. Always check the flag before any signal or state mutation.

**5. `position: sticky` ignored inside `overflow: hidden` containers**
If the feed container or any ancestor has `overflow: hidden` or `overflow: auto`, `position: sticky` inside it behaves as `position: relative` and the banner never sticks to the viewport. Fix: ensure the scroll happens on the `<body>` or an ancestor that does NOT have `overflow: hidden`. Alternatively, move the banner outside the scroll container and use `position: fixed` with `left: 50%; transform: translateX(-50%)`.

**6. SSE does not support custom request headers**
SSE uses a plain HTTP GET — you cannot attach an `Authorization: Bearer` header from the browser (cookies work, but custom headers do not). Fix: pass a short-lived token in the query string (`?token=...`) obtained from an authenticated endpoint, or switch to WebSocket if you need header-based auth or bidirectional messaging.

**7. Double reconnect when using SSE with your own backoff**
The browser's `EventSource` has built-in reconnect logic triggered by a `retry:` field in the SSE stream. If you also implement your own `onerror` retry without calling `es.close()` first, both mechanisms fire simultaneously. Fix: always call `es.close()` synchronously inside `onerror` before scheduling your own timer, as shown in the `useSSEFeed` hook above.

---

## Interview Angle

### Q: "A live feed prepends new items. How do you prevent content from jumping when the user has scrolled down to read older posts?"

There are two distinct problems. First, the decision of whether to prepend at all: track `scrollTop` (or `window.scrollY`) with a passive scroll listener and compare it against a threshold — 120px is a reasonable value. If the user is within that threshold, prepend immediately. If they are further down, push the item into a separate buffer array instead. No DOM mutation occurs, so there is no layout shift.

Second, even when the user is at the top and you do prepend, older browsers can still cause a flash. The CSS property `overflow-anchor: auto` (the browser default on most scroll containers) should pin the scroll position to the content below the insertion point, but its cross-browser reliability is inconsistent. The safe fallback is the `scrollHeight - scrollTop` restoration pattern: capture the value before the state update, let React or Angular update the DOM, then immediately write `scrollTop = newScrollHeight - capturedValue` inside a `requestAnimationFrame`.

### Follow-up: "How does your exponential backoff behave if 10,000 clients all disconnect simultaneously during a deploy?"

Without jitter, all 10,000 clients compute the same backoff value (`1s`, `2s`, `4s`...) and reconnect in perfectly synchronized waves, which can crash a freshly restarted server. The fix is to add random jitter: `delay = BASE_DELAY * 2^attempt + Math.random() * 1000`. This spreads the reconnect attempts across a one-second window per wave, reducing peak concurrency by roughly the jitter factor. The `intentional` flag ensures clients that were explicitly closed (component unmount, user logout) never enter the reconnect loop at all — only unintentional disconnects trigger backoff.
