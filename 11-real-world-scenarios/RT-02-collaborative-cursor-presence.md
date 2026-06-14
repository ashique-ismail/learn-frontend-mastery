# Real-Time Collaborative Cursor Presence

## The Idea

**In plain English:** Every connected user sees a labelled, colour-coded cursor for every other user moving around the same shared canvas or document. When someone joins, their cursor appears. When they leave, it disappears. The cursor positions are broadcast over a WebSocket, throttled so the network is not flooded, and smoothed on the receiving end so movement looks fluid rather than teleporting.

**Real-world analogy:** Think of a glass whiteboard in a conference room where multiple people are drawing at once.

- Each person's **marker** = their cursor, with their name written on the cap.
- The **conference call camera** = the WebSocket connection — everyone watching the stream sees each other's markers move.
- If someone narrated every millimetre of marker movement over the phone, the call would be unusable. Instead, you glance up every second or two to see where everyone is — that throttle is the 50 ms update interval.
- When you blink between glances, a marker seems to jump. The **smooth sweep of your eye** filling in that gap = CSS `transition` interpolating between the last known position and the new one.
- When someone **leaves the room**, their marker vanishes — that is the `leave` event removing the cursor overlay.
- Each person gets a **unique pen colour** assigned when they join — deterministic so the same user always gets the same colour across reconnects.

The key insight: the WebSocket is the data pipe; throttling protects the server; CSS transitions protect the receiver's eye; the framework owns the cursor map state.

---

## Learning Objectives

- Understand why cursor presence cannot be built with CSS or polling alone
- Implement WebSocket broadcast of cursor positions with a shared room pattern
- Throttle outgoing position updates using both `requestAnimationFrame` and a 50 ms time gate
- Assign stable, deterministic colours to named users
- Render remote cursors as absolutely-positioned overlays with smooth CSS transition
- Handle join/leave events and clean up stale cursors
- Avoid production pitfalls: coordinate systems, resize invalidation, z-index, reconnect storms

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Position a cursor overlay at arbitrary x/y | ✅ `position: absolute; left: x; top: y` | — |
| Smooth movement between two known positions | ✅ `transition: left 0.1s, top 0.1s` | — |
| Receive another user's pointer coordinates | ❌ | CSS has no network layer |
| Maintain a map of connected users | ❌ | CSS has no data structures or state |
| Throttle outgoing events to 50 ms | ❌ | CSS cannot intercept or debounce DOM events |
| Remove a cursor when a user disconnects | ❌ | CSS cannot react to WebSocket `close` frames |
| Assign a colour based on a username hash | ❌ | CSS `attr()` cannot compute hashes |
| Handle viewport resize and recalculate coordinates | ❌ | CSS cannot re-broadcast corrected positions |

**Conclusion:** CSS owns exactly two things here: cursor position (`left`/`top`) and interpolation (`transition`). Every other concern — connection, state, throttle, colour assignment, join/leave — belongs to JS.

---

## HTML & CSS Foundation

### The Canvas Overlay Structure

```html
<!-- The shared canvas area — could be a doc editor, design tool, or board -->
<div class="collab-canvas" id="canvas">

  <!-- Your own content lives here -->
  <div class="canvas-content">...</div>

  <!-- Remote cursors are injected here by JS -->
  <div class="cursor-layer" aria-hidden="true">
    <!-- One .remote-cursor per connected peer, added/removed dynamically -->
    <div
      class="remote-cursor"
      data-user-id="u_42"
      style="--cx: 340px; --cy: 218px; --user-color: #e05c4b;"
    >
      <svg class="cursor-icon" viewBox="0 0 16 20" aria-hidden="true">
        <path d="M0 0 L0 20 L5 15 L9 20 L11 18 L7 13 L13 13 Z" />
      </svg>
      <span class="cursor-label">Alice</span>
    </div>
  </div>

</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --cursor-transition-duration: 80ms;
  --cursor-label-bg: rgba(0, 0, 0, 0.72);
  --cursor-label-radius: 4px;
}

/* ─── Canvas container ─── */
.collab-canvas {
  position: relative;   /* establishes the coordinate system for cursors */
  width: 100%;
  height: 100%;
  overflow: hidden;
  user-select: none;    /* prevents accidental text selection while tracking */
}

/* ─── Cursor layer sits above content, below modals ─── */
.cursor-layer {
  position: absolute;
  inset: 0;
  pointer-events: none;  /* clicks pass through to canvas content */
  z-index: 50;
}

/* ─── Individual remote cursor ─── */
.remote-cursor {
  position: absolute;
  /*
    CSS custom properties carry the server-authoritative position.
    JS only updates --cx and --cy; the browser handles the rest.
  */
  left: var(--cx, 0px);
  top:  var(--cy, 0px);

  /*
    transition on left/top provides the interpolation.
    linear easing matches pointer velocity better than ease-in-out.
    80ms is short enough to feel live, long enough to hide jitter.
  */
  transition:
    left  var(--cursor-transition-duration) linear,
    top   var(--cursor-transition-duration) linear;

  display: flex;
  align-items: flex-start;
  gap: 4px;
}

/* Remove transition on the very first placement to avoid sliding in from (0,0) */
.remote-cursor.no-transition {
  transition: none;
}

/* ─── Cursor SVG icon ─── */
.cursor-icon {
  width: 16px;
  height: 20px;
  fill: var(--user-color, #888);
  stroke: white;
  stroke-width: 1px;
  flex-shrink: 0;
  filter: drop-shadow(0 1px 2px rgba(0,0,0,0.4));
}

/* ─── Name label ─── */
.cursor-label {
  display: inline-block;
  margin-top: 16px;   /* aligns below the cursor tip */
  padding: 2px 6px;
  background: var(--user-color, #888);
  color: white;
  font-size: 11px;
  font-weight: 600;
  line-height: 1.4;
  border-radius: var(--cursor-label-radius);
  white-space: nowrap;
  box-shadow: 0 1px 3px rgba(0,0,0,0.3);
}

/* ─── Fade out stale / disconnecting cursor ─── */
.remote-cursor.is-leaving {
  opacity: 0;
  transition:
    left    var(--cursor-transition-duration) linear,
    top     var(--cursor-transition-duration) linear,
    opacity 0.4s ease;
}

/* ─── Reduced motion: disable position interpolation ─── */
@media (prefers-reduced-motion: reduce) {
  .remote-cursor {
    transition: none;
  }
}
```

**What CSS owns:** cursor position via custom properties, smooth interpolation via `transition`, name label styling, leave fade-out animation, reduced-motion opt-out.

**What CSS cannot own:** WebSocket connection, throttled event emission, cursor map state, colour assignment, join/leave lifecycle.

---

## React Implementation

### Types

```tsx
// types.ts
export interface RemoteCursor {
  userId: string;
  name: string;
  color: string;
  x: number;         // 0–100 percentage of canvas width
  y: number;         // 0–100 percentage of canvas height
  updatedAt: number; // Date.now() — used to GC stale cursors
}

export type ServerMessage =
  | { type: 'cursor_move';        userId: string; name: string; x: number; y: number }
  | { type: 'user_join';          userId: string; name: string; color: string }
  | { type: 'user_leave';         userId: string }
  | { type: 'presence_snapshot';  cursors: RemoteCursor[] }; // full state on connect
```

### Colour Assignment Utility

```tsx
// cursorColor.ts
const PALETTE = [
  '#e05c4b', '#e0944b', '#d4c44b', '#4be06a',
  '#4bb8e0', '#4b6de0', '#a04be0', '#e04ba8',
];

/**
 * Deterministic colour from userId — same user always gets the same colour.
 * Uses a DJB2-style hash so no server coordination is needed.
 */
export function colorForUser(userId: string): string {
  let hash = 5381;
  for (let i = 0; i < userId.length; i++) {
    hash = ((hash << 5) + hash) ^ userId.charCodeAt(i);
    hash = hash >>> 0; // coerce to uint32
  }
  return PALETTE[hash % PALETTE.length];
}
```

### The WebSocket Hook

```tsx
// useCollabSocket.ts
import { useEffect, useRef, useCallback } from 'react';

type MessageHandler = (msg: unknown) => void;

interface UseCollabSocketOptions {
  url: string;
  onMessage: MessageHandler;
  onOpen?:   () => void;
  onClose?:  () => void;
}

export function useCollabSocket({ url, onMessage, onOpen, onClose }: UseCollabSocketOptions) {
  const wsRef      = useRef<WebSocket | null>(null);
  const reconnectT = useRef<ReturnType<typeof setTimeout> | null>(null);
  const mountedRef = useRef(true);

  const connect = useCallback(() => {
    const ws = new WebSocket(url);
    wsRef.current = ws;

    ws.onopen = () => { onOpen?.(); };

    ws.onmessage = (evt) => {
      try { onMessage(JSON.parse(evt.data as string)); }
      catch { /* ignore malformed frames */ }
    };

    ws.onclose = () => {
      onClose?.();
      if (!mountedRef.current) return;
      // Exponential back-off capped at 8 s to avoid reconnect storm
      reconnectT.current = setTimeout(connect, Math.min(8000, 1000));
    };

    ws.onerror = () => ws.close();
  }, [url, onMessage, onOpen, onClose]);

  const send = useCallback((data: unknown) => {
    if (wsRef.current?.readyState === WebSocket.OPEN) {
      wsRef.current.send(JSON.stringify(data));
    }
  }, []);

  useEffect(() => {
    mountedRef.current = true;
    connect();
    return () => {
      mountedRef.current = false;
      if (reconnectT.current) clearTimeout(reconnectT.current);
      wsRef.current?.close();
    };
  }, [connect]);

  return { send };
}
```

### The Cursor Presence Hook

```tsx
// useCursorPresence.ts
import { useState, useCallback, useRef, useEffect, RefObject } from 'react';
import { useCollabSocket } from './useCollabSocket';
import { colorForUser } from './cursorColor';
import type { RemoteCursor, ServerMessage } from './types';

const THROTTLE_MS  = 50;
const STALE_TTL_MS = 10_000; // remove cursors silent for 10 s

interface Options {
  canvasRef:     RefObject<HTMLElement>;
  roomUrl:       string;
  localUserId:   string;
  localUserName: string;
}

export function useCursorPresence({ canvasRef, roomUrl, localUserId, localUserName }: Options) {
  const [cursors, setCursors] = useState<Map<string, RemoteCursor>>(new Map());

  // ── Throttled outgoing position broadcast ─────────────────────────────────
  const lastSendRef = useRef(0);
  const pendingRef  = useRef<{ x: number; y: number } | null>(null);
  const rafRef      = useRef<number>(0);

  const { send } = useCollabSocket({
    url: roomUrl,

    onMessage: useCallback((raw: unknown) => {
      const msg = raw as ServerMessage;

      setCursors(prev => {
        const next = new Map(prev);

        if (msg.type === 'user_join') {
          // Seed with zero position; first cursor_move event will update it
          next.set(msg.userId, {
            userId: msg.userId,
            name: msg.name,
            color: msg.color ?? colorForUser(msg.userId),
            x: 0, y: 0,
            updatedAt: Date.now(),
          });
        } else if (msg.type === 'user_leave') {
          next.delete(msg.userId);
        } else if (msg.type === 'cursor_move') {
          const existing = next.get(msg.userId);
          if (existing) {
            next.set(msg.userId, { ...existing, x: msg.x, y: msg.y, updatedAt: Date.now() });
          }
        } else if (msg.type === 'presence_snapshot') {
          for (const c of msg.cursors) {
            if (c.userId !== localUserId) next.set(c.userId, c);
          }
        }

        return next;
      });
    }, [localUserId]),

    onOpen: useCallback(() => {
      // Announce ourselves to the room
      send({ type: 'user_join', userId: localUserId, name: localUserName });
    }, [send, localUserId, localUserName]),
  });

  // ── Pointer move handler — RAF + 50 ms gate ───────────────────────────────
  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    const onMove = (e: PointerEvent) => {
      const rect = canvas.getBoundingClientRect();
      // Normalise to canvas-local percentage coordinates (not viewport pixels)
      const x = ((e.clientX - rect.left) / rect.width)  * 100;
      const y = ((e.clientY - rect.top)  / rect.height) * 100;

      pendingRef.current = { x, y };

      cancelAnimationFrame(rafRef.current);
      rafRef.current = requestAnimationFrame(() => {
        const now = Date.now();
        if (now - lastSendRef.current < THROTTLE_MS) return;
        if (!pendingRef.current) return;

        lastSendRef.current = now;
        send({ type: 'cursor_move', userId: localUserId, x: pendingRef.current.x, y: pendingRef.current.y });
        pendingRef.current = null;
      });
    };

    canvas.addEventListener('pointermove', onMove);
    return () => {
      canvas.removeEventListener('pointermove', onMove);
      cancelAnimationFrame(rafRef.current);
    };
  }, [canvasRef, send, localUserId]);

  // ── Stale cursor GC — runs every 5 s ─────────────────────────────────────
  useEffect(() => {
    const interval = setInterval(() => {
      const cutoff = Date.now() - STALE_TTL_MS;
      setCursors(prev => {
        const next = new Map(prev);
        let changed = false;
        for (const [id, c] of next) {
          if (c.updatedAt < cutoff) { next.delete(id); changed = true; }
        }
        return changed ? next : prev;
      });
    }, 5_000);
    return () => clearInterval(interval);
  }, []);

  return { cursors };
}
```

### The Canvas Component

```tsx
// CollabCanvas.tsx
import { useRef, useLayoutEffect } from 'react';
import { useCursorPresence } from './useCursorPresence';
import type { RemoteCursor } from './types';

interface Props {
  roomUrl:   string;
  userId:    string;
  userName:  string;
  children:  React.ReactNode;
}

export function CollabCanvas({ roomUrl, userId, userName, children }: Props) {
  const canvasRef = useRef<HTMLDivElement>(null);
  const { cursors } = useCursorPresence({
    canvasRef,
    roomUrl,
    localUserId:   userId,
    localUserName: userName,
  });

  return (
    <div ref={canvasRef} className="collab-canvas" aria-label="Collaborative canvas">
      <div className="canvas-content">{children}</div>

      {/* Remote cursor overlays — aria-hidden because they are decorative */}
      <div className="cursor-layer" aria-hidden="true">
        {[...cursors.values()].map(cursor => (
          <RemoteCursorOverlay key={cursor.userId} cursor={cursor} />
        ))}
      </div>
    </div>
  );
}

// ── Single remote cursor ─────────────────────────────────────────────────────

interface OverlayProps {
  cursor: RemoteCursor;
}

function RemoteCursorOverlay({ cursor }: OverlayProps) {
  const ref        = useRef<HTMLDivElement>(null);
  const isFirstRef = useRef(true);

  /*
    On the very first render for this userId suppress the transition so the
    cursor appears at the correct position instantly rather than sliding in
    from (0 %, 0 %).
  */
  useLayoutEffect(() => {
    if (!ref.current) return;
    if (isFirstRef.current) {
      ref.current.classList.add('no-transition');
      void ref.current.offsetHeight; // force reflow so class takes effect
      ref.current.classList.remove('no-transition');
      isFirstRef.current = false;
    }
  }, []);

  return (
    <div
      ref={ref}
      className="remote-cursor"
      data-user-id={cursor.userId}
      style={{
        '--cx':         `${cursor.x}%`,
        '--cy':         `${cursor.y}%`,
        '--user-color': cursor.color,
      } as React.CSSProperties}
    >
      <svg className="cursor-icon" viewBox="0 0 16 20" aria-hidden="true">
        <path d="M0 0 L0 20 L5 15 L9 20 L11 18 L7 13 L13 13 Z" />
      </svg>
      <span className="cursor-label">{cursor.name}</span>
    </div>
  );
}
```

---

## Angular Implementation

### Models

```typescript
// collab.model.ts
export interface RemoteCursor {
  userId:    string;
  name:      string;
  color:     string;
  x:         number;
  y:         number;
  updatedAt: number;
}

export type ServerMessage =
  | { type: 'cursor_move';        userId: string; name: string; x: number; y: number }
  | { type: 'user_join';          userId: string; name: string; color: string }
  | { type: 'user_leave';         userId: string }
  | { type: 'presence_snapshot';  cursors: RemoteCursor[] };
```

### Presence Service

```typescript
// cursor-presence.service.ts
import { Injectable, OnDestroy, signal, computed, inject, PLATFORM_ID } from '@angular/core';
import { isPlatformBrowser } from '@angular/common';
import type { RemoteCursor, ServerMessage } from './collab.model';

const THROTTLE_MS  = 50;
const STALE_TTL_MS = 10_000;

const PALETTE = ['#e05c4b','#e0944b','#d4c44b','#4be06a','#4bb8e0','#4b6de0','#a04be0','#e04ba8'];

function colorForUser(userId: string): string {
  let hash = 5381;
  for (let i = 0; i < userId.length; i++) {
    hash = ((hash << 5) + hash) ^ userId.charCodeAt(i);
    hash = hash >>> 0;
  }
  return PALETTE[hash % PALETTE.length];
}

@Injectable({ providedIn: 'root' })
export class CursorPresenceService implements OnDestroy {
  private platformId = inject(PLATFORM_ID);

  // ── State ─────────────────────────────────────────────────────────────────
  private _cursors = signal<Map<string, RemoteCursor>>(new Map());
  readonly cursors = computed(() => [...this._cursors().values()]);

  // ── WebSocket ─────────────────────────────────────────────────────────────
  private ws:             WebSocket | null = null;
  private reconnectTimer: ReturnType<typeof setTimeout> | null = null;
  private destroyed =     false;

  private localUserId   = '';
  private localUserName = '';
  private roomUrl       = '';

  // ── Throttle refs ─────────────────────────────────────────────────────────
  private lastSendTime = 0;
  private pendingPos:  { x: number; y: number } | null = null;
  private rafId = 0;

  // ── Stale cursor GC ───────────────────────────────────────────────────────
  private gcInterval: ReturnType<typeof setInterval> | null = null;

  connect(roomUrl: string, userId: string, userName: string): void {
    if (!isPlatformBrowser(this.platformId)) return;
    this.roomUrl       = roomUrl;
    this.localUserId   = userId;
    this.localUserName = userName;
    this.openSocket();
    this.startGC();
  }

  private openSocket(): void {
    const ws = new WebSocket(this.roomUrl);
    this.ws = ws;

    ws.onopen = () => {
      this.send({ type: 'user_join', userId: this.localUserId, name: this.localUserName });
    };

    ws.onmessage = (evt) => {
      try { this.handleMessage(JSON.parse(evt.data as string)); }
      catch { /* ignore malformed frames */ }
    };

    ws.onclose = () => {
      if (this.destroyed) return;
      // Jittered reconnect — avoids thundering-herd on mass disconnect
      const delay = Math.min(8000, 1000) + Math.random() * 2000;
      this.reconnectTimer = setTimeout(() => this.openSocket(), delay);
    };

    ws.onerror = () => ws.close();
  }

  private handleMessage(msg: ServerMessage): void {
    this._cursors.update(prev => {
      const next = new Map(prev);
      switch (msg.type) {
        case 'user_join':
          next.set(msg.userId, {
            userId: msg.userId, name: msg.name,
            color: msg.color ?? colorForUser(msg.userId),
            x: 0, y: 0, updatedAt: Date.now(),
          });
          break;
        case 'user_leave':
          next.delete(msg.userId);
          break;
        case 'cursor_move': {
          const c = next.get(msg.userId);
          if (c) next.set(msg.userId, { ...c, x: msg.x, y: msg.y, updatedAt: Date.now() });
          break;
        }
        case 'presence_snapshot':
          for (const c of msg.cursors) {
            if (c.userId !== this.localUserId) next.set(c.userId, c);
          }
          break;
      }
      return next;
    });
  }

  broadcastMove(canvasEl: HTMLElement, clientX: number, clientY: number): void {
    const rect = canvasEl.getBoundingClientRect();
    const x = ((clientX - rect.left) / rect.width)  * 100;
    const y = ((clientY - rect.top)  / rect.height) * 100;
    this.pendingPos = { x, y };

    cancelAnimationFrame(this.rafId);
    this.rafId = requestAnimationFrame(() => {
      const now = Date.now();
      if (now - this.lastSendTime < THROTTLE_MS || !this.pendingPos) return;
      this.lastSendTime = now;
      this.send({ type: 'cursor_move', userId: this.localUserId, ...this.pendingPos });
      this.pendingPos = null;
    });
  }

  private send(data: unknown): void {
    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(data));
    }
  }

  private startGC(): void {
    this.gcInterval = setInterval(() => {
      const cutoff = Date.now() - STALE_TTL_MS;
      this._cursors.update(prev => {
        const next = new Map(prev);
        let changed = false;
        for (const [id, c] of next) {
          if (c.updatedAt < cutoff) { next.delete(id); changed = true; }
        }
        return changed ? next : prev;
      });
    }, 5_000);
  }

  ngOnDestroy(): void {
    this.destroyed = true;
    if (this.reconnectTimer) clearTimeout(this.reconnectTimer);
    if (this.gcInterval)     clearInterval(this.gcInterval);
    cancelAnimationFrame(this.rafId);
    this.ws?.close();
  }
}
```

### Canvas Component

```typescript
// collab-canvas.component.ts
import {
  Component, OnInit, ElementRef, Input, inject, AfterViewInit
} from '@angular/core';
import { NgFor } from '@angular/common';
import { CursorPresenceService } from './cursor-presence.service';
import type { RemoteCursor } from './collab.model';

@Component({
  selector: 'app-collab-canvas',
  standalone: true,
  imports: [NgFor],
  host: {
    class: 'collab-canvas',
    '(pointermove)': 'onPointerMove($event)',
    'aria-label':    'Collaborative canvas',
  },
  template: `
    <div class="canvas-content">
      <ng-content />
    </div>

    <!-- aria-hidden: cursors are decorative, screen readers do not need position data -->
    <div class="cursor-layer" aria-hidden="true">
      <div
        *ngFor="let cursor of presenceService.cursors(); trackBy: trackById"
        class="remote-cursor"
        [attr.data-user-id]="cursor.userId"
        [style.--cx]="cursor.x + '%'"
        [style.--cy]="cursor.y + '%'"
        [style.--user-color]="cursor.color"
      >
        <svg class="cursor-icon" viewBox="0 0 16 20" aria-hidden="true">
          <path d="M0 0 L0 20 L5 15 L9 20 L11 18 L7 13 L13 13 Z" />
        </svg>
        <span class="cursor-label">{{ cursor.name }}</span>
      </div>
    </div>
  `,
})
export class CollabCanvasComponent implements OnInit {
  @Input({ required: true }) roomUrl!:  string;
  @Input({ required: true }) userId!:   string;
  @Input({ required: true }) userName!: string;

  presenceService = inject(CursorPresenceService);
  private el      = inject(ElementRef<HTMLElement>);

  ngOnInit(): void {
    this.presenceService.connect(this.roomUrl, this.userId, this.userName);
  }

  onPointerMove(e: PointerEvent): void {
    this.presenceService.broadcastMove(this.el.nativeElement, e.clientX, e.clientY);
  }

  trackById(_: number, cursor: RemoteCursor): string {
    return cursor.userId;
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Cursor layer is hidden from assistive technology | `aria-hidden="true"` on `.cursor-layer` — screen reader users do not need a stream of cursor position announcements |
| Canvas has a descriptive accessible name | `aria-label="Collaborative canvas"` on the container element |
| Join/leave events announced once, not continuously | A single `aria-live="polite"` status bar outside the cursor layer; updated on user join/leave only, not on move events |
| Cursor movement never triggers live region updates | Position updates go to CSS custom properties only, not to any ARIA live region |
| Reduced motion respected | `@media (prefers-reduced-motion: reduce)` disables `transition` on `.remote-cursor` — position jumps directly with no animation |
| No keyboard trap introduced | The canvas does not intercept Tab; cursor tracking uses `pointermove`, not `keydown` |
| Colour is not the sole identifier | The cursor `name` label is always visible alongside the colour — colourblind users can read names |
| Sufficient contrast for name labels | White text on the assigned colour; palette choices maintain WCAG AA minimum contrast ratio |
| Cursors never block interaction | `pointer-events: none` on `.cursor-layer` and all `.remote-cursor` descendants ensures clicks pass through |

---

## Production Pitfalls

**1. Viewport coordinates break on different screen sizes**
If you broadcast raw `clientX/clientY`, remote cursors are wrong the moment anyone has a different viewport size or has scrolled. Fix: always normalise to a percentage of the canvas bounding rect — `(clientX - rect.left) / rect.width * 100`. Percentages survive scrolling and different screen resolutions. Never cache `getBoundingClientRect()` in a closure; call it inside the `pointermove` handler every time because the canvas can move after a layout shift.

**2. The cursor slides in from (0, 0) on first appearance**
When a peer's cursor enters the DOM for the first time its initial CSS custom property values are `0%`. The transition then animates the cursor from the top-left corner to the actual position — a visible 80 ms slide. Fix: add a `no-transition` class on the first render, force a reflow (`void el.offsetHeight`), then remove the class so subsequent updates animate normally.

**3. Reconnect storms after a server restart**
If 200 users all receive a WebSocket `close` event simultaneously and all call `setTimeout(reconnect, 1000)`, they produce a synchronised flood one second later. Fix: add random jitter — `1000 + Math.random() * 2000` ms — and cap with exponential back-off. The service above already implements this.

**4. Stale cursors from ungraceful disconnects**
A user whose browser tab crashes will never send a `user_leave` event. Their cursor remains frozen on screen indefinitely. Two fixes work together: (a) the server detects WebSocket closure and broadcasts a synthetic `user_leave` to all room members; (b) the client runs a GC interval that removes any cursor not updated within the TTL window (10 s in the examples above). Never rely on only one mechanism.

**5. `pointer-events: none` missing from the cursor layer**
Without it the remote cursor elements intercept clicks intended for the canvas content beneath them. Drawing tools, clickable nodes, and hyperlinks silently stop working. This is one of the hardest bugs to diagnose because the cursor overlaps vary with user position. Fix: always declare `pointer-events: none` on `.cursor-layer` and verify it in DevTools.

**6. Too many cursors degrading rendering performance**
At 50 concurrent users, 50 elements updating their CSS custom properties 20 times per second is fine. At 200 users it can saturate style recalculation. Fix: switch from `left`/`top` to `transform: translate(x%, y%)` — transforms are handled on the compositor thread and skip layout entirely. Convert percentage positions to pixel offsets once using the canvas dimensions and update `transform` directly via `el.style.transform`.

**7. The 50 ms throttle and the 80 ms CSS transition are mismatched**
If the transition window is shorter than the broadcast interval the cursor teleports instead of gliding. If it is much longer the cursor lags visibly behind the sender. The transition duration should be roughly equal to the broadcast interval — 80 ms transition for a 50 ms send rate means the animation completes just as the next position arrives, giving the appearance of continuous motion.

---

## Interview Angle

**Q: "How would you implement live cursor sharing in a collaborative tool like Figma or Notion?"**

Strong answer covers five points:

1. **Transport** — WebSocket over HTTP/1.1 (or a shared worker for same-origin tabs). A dedicated room channel per document keeps message fanout O(room-size) rather than O(all users). Explain why polling (even at 50 ms) doubles latency and wastes bandwidth compared to server-push.

2. **Throttling** — Two gates working together: `requestAnimationFrame` coalesces multiple `pointermove` events within a single frame into one pending position, then a 50 ms wall-clock check ensures you send at most 20 messages per second. This is not debounce (which would delay the final position) — it is a leaky-bucket rate limit that always sends the most recent position.

3. **Coordinate normalisation** — Broadcast percentages of the canvas dimensions, never raw viewport pixels. Percentages are invariant to scroll position, viewport size, and zoom level. The receiver applies `left: x%; top: y%` and CSS handles the mapping.

4. **Smooth interpolation** — CSS `transition: left 80ms linear, top 80ms linear` is the right tool because it runs on the compositor thread, does not trigger JS, and is cancelled automatically if a newer position arrives mid-transition. The 80 ms window matches the 50 ms send interval plus modest network jitter — short enough to feel live, long enough to hide individual packet delay.

5. **Lifecycle correctness** — `user_join` on WebSocket open, `user_leave` on server-detected socket close (not client-sent — an unclean disconnect cannot send a leave event), plus client-side GC for stale cursors as a safety net. Deterministic colour assignment from a hash of userId avoids a round-trip for colour negotiation.

Follow-up: "What would break first under load, and how would you fix it?"

At low concurrency the bottleneck is network bandwidth per user (50 msgs/s at ~40 bytes each = ~2 KB/s, acceptable). At high concurrency (200+ users) the bottleneck shifts to server fanout — 200 senders broadcasting to 200 receivers each = 40,000 messages per second per room. Fix: spatial partitioning (only broadcast a cursor to users whose viewport overlaps the sender's position) or a server-side aggregation layer that reduces broadcast frequency for users far from the viewport edge. On the client side, switching from `left`/`top` to `transform: translate()` keeps cursor rendering off the main thread entirely and is the first performance improvement to reach for.
