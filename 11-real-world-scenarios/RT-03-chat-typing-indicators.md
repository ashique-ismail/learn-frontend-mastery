# Chat Interface with Typing Indicators

## The Idea

**In plain English:** You have a real-time chat UI. When you hit Send, the message should appear immediately in your UI (optimistic update) with a temporary local ID and a "sending" status badge — before the server confirms it. As the message travels through the system, its status badge transitions: sending → sent → delivered → read. Simultaneously, when the other person starts typing, a "..." bubble appears. When they stop (and hold for 2–3 seconds), the bubble disappears. The scroll position auto-anchors to the bottom — but only if you were already at the bottom before the new message arrived. If you scrolled up to read history, the anchor does not steal your position.

**Real-world analogy:** Think of a physical postal sorting office combined with read receipts on a letter.

- **Sending** = you dropped the letter in the outbox tray — it is in your building, not yet picked up.
- **Sent** = the postal van picked it up — it left your hands, confirmation from the van driver.
- **Delivered** = the letter arrived in the recipient's mailbox — the post office confirmed delivery.
- **Read** = the recipient opened and read the letter — they signed a return receipt.
- The **optimistic temp ID** = the tracking number you print at home before handing the package to the courier. You already labeled the parcel; the server will later overwrite the tracking number with its own canonical ID.
- **Scroll anchoring** = sitting at the bottom of a long conveyor belt. If you walk upstream to inspect an old parcel, the belt does not drag you back. If you are already standing at the output end, each new parcel slides to you naturally.
- **Typing indicator** = the "..." speech bubble. It appears the moment the other person presses a key. A debounced timer clears it — not immediately on each keystroke, but after a pause of inactivity (otherwise it would flicker with every word).

The key insight: optimistic UI, delivery status, scroll anchoring, and typing indicators are four separate, interlocking state machines. Each has its own lifecycle and failure mode.

---

## Learning Objectives

- Understand optimistic updates: assign a temp ID, render immediately, reconcile on server ack
- Model delivery status as a typed enum and update it through a predictable state machine
- Implement scroll anchoring that only auto-scrolls when the user is already at the bottom
- Debounce the typing-stop event correctly so the indicator does not flicker
- Handle the failure case: revert an optimistic message when the server returns an error
- Build the pattern in both React (hooks) and Angular (signals + service)
- Apply the correct ARIA roles for live regions in a chat interface

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Render a message bubble immediately on send | ❌ | Requires inserting DOM from JS before server responds |
| Show "sending / sent / delivered / read" badge | ❌ | CSS has no async state; badge text and icon depend on server events |
| Replace temp ID with server-assigned ID | ❌ | DOM reconciliation requires JS array identity matching |
| Revert message on send failure | ❌ | Requires removing or marking a specific array item |
| Auto-scroll only when already at bottom | ❌ | Requires measuring `scrollTop + clientHeight ≈ scrollHeight` in JS |
| Show typing indicator after first keystroke | ⚠️ | CSS can show a pre-rendered bubble, but cannot receive WebSocket events |
| Hide typing indicator after pause in typing | ❌ | Requires a debounced timer (setTimeout cleared on each keystroke) |
| Announce new messages to screen readers | ❌ | Requires `aria-live` region managed by JS |

**Conclusion:** CSS owns the visual layer — bubble shapes, status icon glyphs, the animated dots in the typing indicator, reduced-motion fallbacks. JS/framework owns every state transition and every timing decision.

---

## HTML & CSS Foundation

### The Structure

```html
<!-- Chat container -->
<section class="chat-window" aria-label="Conversation with Alice">

  <!-- Message list — the scroll container -->
  <div class="chat-messages" role="log" aria-live="polite" aria-label="Messages">

    <!-- Sent message (my side) -->
    <div class="chat-message chat-message--outbound">
      <p class="chat-message__text">Hey, are you free tomorrow?</p>
      <span class="chat-message__status" aria-label="Delivered">
        <!-- SVG double-tick icon -->
        <svg class="status-icon status-icon--delivered" aria-hidden="true">...</svg>
      </span>
    </div>

    <!-- Received message (their side) -->
    <div class="chat-message chat-message--inbound">
      <p class="chat-message__text">Yes! What time?</p>
    </div>

    <!-- Optimistic message — not yet acknowledged by server -->
    <div class="chat-message chat-message--outbound chat-message--sending">
      <p class="chat-message__text">Around 3pm?</p>
      <span class="chat-message__status" aria-label="Sending">
        <svg class="status-icon status-icon--sending" aria-hidden="true">...</svg>
      </span>
    </div>

    <!-- Typing indicator bubble -->
    <div class="chat-typing-indicator" aria-label="Alice is typing" role="status">
      <span class="typing-dot"></span>
      <span class="typing-dot"></span>
      <span class="typing-dot"></span>
    </div>

  </div>

  <!-- Scroll anchor sentinel — JS scrolls this into view -->
  <div class="chat-scroll-anchor" aria-hidden="true"></div>

  <!-- Composer -->
  <form class="chat-composer" aria-label="New message">
    <textarea
      class="chat-composer__input"
      placeholder="Type a message…"
      aria-label="Message text"
      rows="1"
    ></textarea>
    <button class="chat-composer__send" type="submit" aria-label="Send message">
      Send
    </button>
  </form>

</section>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --bubble-out-bg:   #0b57d0;
  --bubble-out-text: #ffffff;
  --bubble-in-bg:    #f1f3f4;
  --bubble-in-text:  #202124;
  --status-sending:  #9aa0a6;
  --status-sent:     #9aa0a6;
  --status-delivered:#0b57d0;
  --status-read:     #0b57d0;
  --typing-dot-size: 8px;
  --chat-transition: 0.18s ease;
}

/* ─── Chat window layout ─── */
.chat-window {
  display: flex;
  flex-direction: column;
  height: 100%;
  overflow: hidden;
}

/* ─── Scrollable message list ─── */
.chat-messages {
  flex: 1;
  overflow-y: auto;
  padding: 1rem;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  scroll-behavior: smooth;  /* JS can override with 'instant' for initial load */
  overscroll-behavior: contain;
}

/* ─── Message bubbles ─── */
.chat-message {
  display: flex;
  flex-direction: column;
  max-width: 72%;
  gap: 0.25rem;
}

.chat-message--outbound {
  align-self: flex-end;
  align-items: flex-end;
}

.chat-message--inbound {
  align-self: flex-start;
  align-items: flex-start;
}

.chat-message__text {
  margin: 0;
  padding: 0.6rem 0.9rem;
  border-radius: 1.2rem;
  font-size: 0.9375rem;
  line-height: 1.45;
  word-break: break-word;
}

.chat-message--outbound .chat-message__text {
  background: var(--bubble-out-bg);
  color: var(--bubble-out-text);
  border-bottom-right-radius: 0.25rem;
}

.chat-message--inbound .chat-message__text {
  background: var(--bubble-in-bg);
  color: var(--bubble-in-text);
  border-bottom-left-radius: 0.25rem;
}

/* Optimistic message: slightly dimmed while sending */
.chat-message--sending .chat-message__text {
  opacity: 0.65;
}

/* Failed message: red tint */
.chat-message--failed .chat-message__text {
  background: #fce8e6;
  color: #c5221f;
}

/* ─── Status icons ─── */
.chat-message__status {
  display: flex;
  align-items: center;
  gap: 0.25rem;
  font-size: 0.75rem;
  color: var(--status-sending);
}

.status-icon--delivered,
.status-icon--read {
  color: var(--status-delivered);
}

/* ─── Typing indicator ─── */
.chat-typing-indicator {
  display: flex;
  align-items: center;
  gap: 4px;
  padding: 0.6rem 0.9rem;
  background: var(--bubble-in-bg);
  border-radius: 1.2rem;
  border-bottom-left-radius: 0.25rem;
  align-self: flex-start;
  width: fit-content;
}

.typing-dot {
  width: var(--typing-dot-size);
  height: var(--typing-dot-size);
  border-radius: 50%;
  background: #80868b;
  animation: typing-bounce 1.2s infinite ease-in-out;
}

.typing-dot:nth-child(1) { animation-delay: 0s;    }
.typing-dot:nth-child(2) { animation-delay: 0.2s;  }
.typing-dot:nth-child(3) { animation-delay: 0.4s;  }

@keyframes typing-bounce {
  0%, 60%, 100% { transform: translateY(0);    }
  30%           { transform: translateY(-6px); }
}

/* ─── Composer ─── */
.chat-composer {
  display: flex;
  align-items: flex-end;
  gap: 0.5rem;
  padding: 0.75rem 1rem;
  border-top: 1px solid #e0e0e0;
}

.chat-composer__input {
  flex: 1;
  resize: none;
  border: 1px solid #dadce0;
  border-radius: 1.5rem;
  padding: 0.5rem 1rem;
  font-size: 0.9375rem;
  line-height: 1.5;
  outline: none;
  max-height: 160px;
  overflow-y: auto;
}

.chat-composer__input:focus {
  border-color: #0b57d0;
}

.chat-composer__send {
  padding: 0.5rem 1.25rem;
  background: var(--bubble-out-bg);
  color: #fff;
  border: none;
  border-radius: 1.5rem;
  font-size: 0.9375rem;
  cursor: pointer;
  white-space: nowrap;
}

.chat-composer__send:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .typing-dot {
    animation: none;
    opacity: 0.5;
  }
  .chat-messages {
    scroll-behavior: auto;
  }
}
```

**What CSS owns:** bubble layout and color theming, the animated dots, status icon color by class, reduced-motion fallback, optimistic dim and failed-state tint.

**What CSS cannot own:** which status class to apply (requires async server events), whether the typing indicator is rendered at all (requires WebSocket data), whether scrolling snaps to bottom (requires measuring scroll position in JS).

---

## React Implementation

### Types

```tsx
// chat.types.ts
export type DeliveryStatus = 'sending' | 'sent' | 'delivered' | 'read' | 'failed';

export interface ChatMessage {
  /** Stable identity. Optimistic messages use a temp- prefix until server acks. */
  id: string;
  text: string;
  authorId: string;
  timestamp: number;
  status: DeliveryStatus;  // only meaningful for outbound messages
}
```

### The Hook — useChatMessages

```tsx
// useChatMessages.ts
import { useState, useCallback, useRef } from 'react';
import type { ChatMessage, DeliveryStatus } from './chat.types';

let tempCounter = 0;
function tempId(): string {
  return `temp-${Date.now()}-${++tempCounter}`;
}

interface UseChatMessagesOptions {
  myUserId: string;
  sendToServer: (text: string) => Promise<{ id: string }>;
}

export function useChatMessages({ myUserId, sendToServer }: UseChatMessagesOptions) {
  const [messages, setMessages] = useState<ChatMessage[]>([]);

  /**
   * Optimistic send:
   * 1. Create a temp message and push it immediately.
   * 2. Fire the API call.
   * 3. On success: replace the temp ID with the server's canonical ID, set status = 'sent'.
   * 4. On failure: mark the message as 'failed' so the UI can show a retry option.
   */
  const sendMessage = useCallback(async (text: string) => {
    const tid = tempId();

    const optimistic: ChatMessage = {
      id: tid,
      text,
      authorId: myUserId,
      timestamp: Date.now(),
      status: 'sending',
    };

    // 1. Render immediately
    setMessages(prev => [...prev, optimistic]);

    try {
      // 2. Hit the server
      const { id: serverId } = await sendToServer(text);

      // 3. Reconcile: swap temp ID for server ID, advance status
      setMessages(prev =>
        prev.map(m => m.id === tid ? { ...m, id: serverId, status: 'sent' } : m)
      );
    } catch {
      // 4. Mark failed — do not remove; let user retry
      setMessages(prev =>
        prev.map(m => m.id === tid ? { ...m, status: 'failed' } : m)
      );
    }
  }, [myUserId, sendToServer]);

  /** Called by WebSocket handler when server delivers a delivery-status update */
  const updateStatus = useCallback((messageId: string, status: DeliveryStatus) => {
    setMessages(prev =>
      prev.map(m => m.id === messageId ? { ...m, status } : m)
    );
  }, []);

  /** Called by WebSocket handler when inbound messages arrive */
  const receiveMessage = useCallback((msg: ChatMessage) => {
    setMessages(prev => [...prev, msg]);
  }, []);

  return { messages, sendMessage, updateStatus, receiveMessage };
}
```

### The Hook — useScrollAnchor

```tsx
// useScrollAnchor.ts
import { useRef, useCallback, useLayoutEffect } from 'react';

/**
 * Scroll anchoring rule:
 * - If the user is already at (or within THRESHOLD px of) the bottom, auto-scroll on new messages.
 * - If the user scrolled up, leave scroll position alone.
 */
const THRESHOLD = 80; // px from bottom to consider "at bottom"

export function useScrollAnchor(messageCount: number) {
  const containerRef = useRef<HTMLDivElement>(null);
  const anchorRef    = useRef<HTMLDivElement>(null);
  const isAtBottom   = useRef(true); // start anchored

  const onScroll = useCallback(() => {
    const el = containerRef.current;
    if (!el) return;
    const distanceFromBottom = el.scrollHeight - el.scrollTop - el.clientHeight;
    isAtBottom.current = distanceFromBottom <= THRESHOLD;
  }, []);

  // After every render triggered by a new message, scroll if anchored
  useLayoutEffect(() => {
    if (isAtBottom.current) {
      anchorRef.current?.scrollIntoView({ behavior: 'smooth' });
    }
  }, [messageCount]);

  return { containerRef, anchorRef, onScroll };
}
```

### The Hook — useTypingIndicator

```tsx
// useTypingIndicator.ts
import { useState, useCallback, useRef } from 'react';

const TYPING_STOP_DELAY = 2500; // ms of silence before we emit "stopped typing"

interface UseTypingIndicatorOptions {
  /** Call this to broadcast "I am typing" to the server/socket */
  onStartTyping: () => void;
  /** Call this to broadcast "I stopped typing" to the server/socket */
  onStopTyping: () => void;
}

export function useTypingIndicator({ onStartTyping, onStopTyping }: UseTypingIndicatorOptions) {
  const timerRef    = useRef<ReturnType<typeof setTimeout> | null>(null);
  const isTypingRef = useRef(false);

  /** Called on every keystroke in the composer */
  const handleKeyPress = useCallback(() => {
    // Emit "start" only on the FIRST keystroke of a session
    if (!isTypingRef.current) {
      isTypingRef.current = true;
      onStartTyping();
    }

    // Reset the debounce timer on every keystroke
    if (timerRef.current) clearTimeout(timerRef.current);
    timerRef.current = setTimeout(() => {
      isTypingRef.current = false;
      onStopTyping();
    }, TYPING_STOP_DELAY);
  }, [onStartTyping, onStopTyping]);

  /** Call this when the message is actually sent (clears the timer immediately) */
  const handleSend = useCallback(() => {
    if (timerRef.current) clearTimeout(timerRef.current);
    if (isTypingRef.current) {
      isTypingRef.current = false;
      onStopTyping();
    }
  }, [onStopTyping]);

  return { handleKeyPress, handleSend };
}
```

### The Chat Component

```tsx
// ChatWindow.tsx
import { useRef, useState } from 'react';
import { useChatMessages } from './useChatMessages';
import { useScrollAnchor } from './useScrollAnchor';
import { useTypingIndicator } from './useTypingIndicator';
import type { ChatMessage, DeliveryStatus } from './chat.types';

const STATUS_LABEL: Record<DeliveryStatus, string> = {
  sending:   'Sending',
  sent:      'Sent',
  delivered: 'Delivered',
  read:      'Read',
  failed:    'Failed to send',
};

const STATUS_ICON: Record<DeliveryStatus, string> = {
  sending:   '○',
  sent:      '✓',
  delivered: '✓✓',
  read:      '✓✓',   // blue tint applied via CSS class
  failed:    '⚠',
};

interface Props {
  myUserId: string;
  /** True while the remote user is typing — driven by your WebSocket layer */
  remoteIsTyping: boolean;
  /** Injected so the component stays testable */
  sendToServer: (text: string) => Promise<{ id: string }>;
  onStartTyping: () => void;
  onStopTyping: () => void;
}

export function ChatWindow({
  myUserId, remoteIsTyping, sendToServer, onStartTyping, onStopTyping
}: Props) {
  const [draft, setDraft] = useState('');
  const { messages, sendMessage } = useChatMessages({ myUserId, sendToServer });
  const { containerRef, anchorRef, onScroll } = useScrollAnchor(messages.length);
  const { handleKeyPress, handleSend } = useTypingIndicator({ onStartTyping, onStopTyping });

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    const text = draft.trim();
    if (!text) return;
    setDraft('');
    handleSend();
    await sendMessage(text);
  };

  const handleChange = (e: React.ChangeEvent<HTMLTextAreaElement>) => {
    setDraft(e.target.value);
    handleKeyPress();
  };

  return (
    <section className="chat-window" aria-label="Chat">
      {/* role="log" is the correct ARIA role for a message stream */}
      <div
        ref={containerRef}
        className="chat-messages"
        role="log"
        aria-live="polite"
        aria-label="Messages"
        onScroll={onScroll}
      >
        {messages.map(msg => (
          <MessageBubble key={msg.id} message={msg} myUserId={myUserId} />
        ))}

        {remoteIsTyping && (
          <div
            className="chat-typing-indicator"
            role="status"
            aria-label="The other person is typing"
          >
            <span className="typing-dot" />
            <span className="typing-dot" />
            <span className="typing-dot" />
          </div>
        )}

        {/* Scroll anchor sentinel */}
        <div ref={anchorRef} aria-hidden="true" />
      </div>

      <form className="chat-composer" aria-label="New message" onSubmit={handleSubmit}>
        <textarea
          className="chat-composer__input"
          value={draft}
          onChange={handleChange}
          placeholder="Type a message…"
          aria-label="Message text"
          rows={1}
          onKeyDown={e => {
            // Send on Enter (not Shift+Enter)
            if (e.key === 'Enter' && !e.shiftKey) {
              e.preventDefault();
              handleSubmit(e as unknown as React.FormEvent);
            }
          }}
        />
        <button
          className="chat-composer__send"
          type="submit"
          aria-label="Send message"
          disabled={!draft.trim()}
        >
          Send
        </button>
      </form>
    </section>
  );
}

/* ── Single bubble ── */
function MessageBubble({ message, myUserId }: { message: ChatMessage; myUserId: string }) {
  const isOutbound = message.authorId === myUserId;
  const baseClass  = `chat-message chat-message--${isOutbound ? 'outbound' : 'inbound'}`;
  const statusClass = isOutbound ? ` chat-message--${message.status}` : '';

  return (
    <div className={baseClass + statusClass}>
      <p className="chat-message__text">{message.text}</p>
      {isOutbound && (
        <span
          className={`chat-message__status status-icon--${message.status}`}
          aria-label={STATUS_LABEL[message.status]}
        >
          {STATUS_ICON[message.status]}
        </span>
      )}
    </div>
  );
}
```

---

## Angular Implementation

### Types & Service

```typescript
// chat.model.ts
export type DeliveryStatus = 'sending' | 'sent' | 'delivered' | 'read' | 'failed';

export interface ChatMessage {
  id: string;
  text: string;
  authorId: string;
  timestamp: number;
  status: DeliveryStatus;
}
```

```typescript
// chat.service.ts
import { Injectable, signal, computed } from '@angular/core';
import type { ChatMessage, DeliveryStatus } from './chat.model';

let tempCounter = 0;
function tempId(): string {
  return `temp-${Date.now()}-${++tempCounter}`;
}

@Injectable({ providedIn: 'root' })
export class ChatService {
  private _messages    = signal<ChatMessage[]>([]);
  private _remoteTyping = signal(false);

  readonly messages      = this._messages.asReadonly();
  readonly remoteTyping  = this._remoteTyping.asReadonly();
  readonly messageCount  = computed(() => this._messages().length);

  async sendMessage(
    text: string,
    myUserId: string,
    sendToServer: (text: string) => Promise<{ id: string }>
  ): Promise<void> {
    const tid = tempId();
    const optimistic: ChatMessage = {
      id: tid, text, authorId: myUserId,
      timestamp: Date.now(), status: 'sending',
    };

    // Optimistic: push immediately
    this._messages.update(msgs => [...msgs, optimistic]);

    try {
      const { id: serverId } = await sendToServer(text);
      this._messages.update(msgs =>
        msgs.map(m => m.id === tid ? { ...m, id: serverId, status: 'sent' } : m)
      );
    } catch {
      this._messages.update(msgs =>
        msgs.map(m => m.id === tid ? { ...m, status: 'failed' } : m)
      );
    }
  }

  updateStatus(messageId: string, status: DeliveryStatus) {
    this._messages.update(msgs =>
      msgs.map(m => m.id === messageId ? { ...m, status } : m)
    );
  }

  receiveMessage(msg: ChatMessage) {
    this._messages.update(msgs => [...msgs, msg]);
  }

  setRemoteTyping(typing: boolean) {
    this._remoteTyping.set(typing);
  }
}
```

### Chat Component

```typescript
// chat-window.component.ts
import {
  Component, inject, signal, computed,
  ViewChild, ElementRef, AfterViewChecked, effect
} from '@angular/core';
import { FormsModule } from '@angular/forms';
import { NgFor, NgIf, NgClass } from '@angular/common';
import { ChatService } from './chat.service';
import type { ChatMessage, DeliveryStatus } from './chat.model';

const STATUS_LABEL: Record<DeliveryStatus, string> = {
  sending: 'Sending', sent: 'Sent', delivered: 'Delivered', read: 'Read', failed: 'Failed to send',
};

const STATUS_ICON: Record<DeliveryStatus, string> = {
  sending: '○', sent: '✓', delivered: '✓✓', read: '✓✓', failed: '⚠',
};

const TYPING_STOP_DELAY = 2500;
const SCROLL_THRESHOLD  = 80;

@Component({
  selector: 'app-chat-window',
  standalone: true,
  imports: [FormsModule, NgFor, NgIf, NgClass],
  template: `
    <section class="chat-window" aria-label="Chat">

      <!-- Message log -->
      <div
        #messageList
        class="chat-messages"
        role="log"
        aria-live="polite"
        aria-label="Messages"
        (scroll)="onScroll()"
      >
        <div
          *ngFor="let msg of chatService.messages(); trackBy: trackById"
          [class]="bubbleClass(msg)"
        >
          <p class="chat-message__text">{{ msg.text }}</p>
          <span
            *ngIf="isOutbound(msg)"
            class="chat-message__status"
            [class]="'status-icon--' + msg.status"
            [attr.aria-label]="statusLabel(msg.status)"
          >
            {{ statusIcon(msg.status) }}
          </span>
        </div>

        <!-- Typing indicator -->
        <div
          *ngIf="chatService.remoteTyping()"
          class="chat-typing-indicator"
          role="status"
          aria-label="The other person is typing"
        >
          <span class="typing-dot"></span>
          <span class="typing-dot"></span>
          <span class="typing-dot"></span>
        </div>

        <!-- Scroll anchor -->
        <div #scrollAnchor aria-hidden="true"></div>
      </div>

      <!-- Composer -->
      <form class="chat-composer" aria-label="New message" (submit)="handleSubmit($event)">
        <textarea
          class="chat-composer__input"
          [(ngModel)]="draft"
          name="draft"
          placeholder="Type a message…"
          aria-label="Message text"
          rows="1"
          (input)="handleInput()"
          (keydown.enter.prevent)="handleEnter($event)"
        ></textarea>
        <button
          class="chat-composer__send"
          type="submit"
          aria-label="Send message"
          [disabled]="!draft().trim()"
        >
          Send
        </button>
      </form>

    </section>
  `,
})
export class ChatWindowComponent implements AfterViewChecked {
  chatService = inject(ChatService);

  readonly myUserId = 'me';
  draft = signal('');

  @ViewChild('messageList') messageListEl!: ElementRef<HTMLDivElement>;
  @ViewChild('scrollAnchor') scrollAnchorEl!: ElementRef<HTMLDivElement>;

  private isAtBottom = true;
  private typingTimer: ReturnType<typeof setTimeout> | null = null;
  private isCurrentlyTyping = false;

  // ── Scroll anchoring ──────────────────────────────────────────────
  onScroll() {
    const el = this.messageListEl.nativeElement;
    const distFromBottom = el.scrollHeight - el.scrollTop - el.clientHeight;
    this.isAtBottom = distFromBottom <= SCROLL_THRESHOLD;
  }

  ngAfterViewChecked() {
    if (this.isAtBottom) {
      this.scrollAnchorEl?.nativeElement.scrollIntoView({ behavior: 'smooth' });
    }
  }

  // ── Typing indicator ──────────────────────────────────────────────
  handleInput() {
    if (!this.isCurrentlyTyping) {
      this.isCurrentlyTyping = true;
      this.emitStartTyping();
    }
    if (this.typingTimer) clearTimeout(this.typingTimer);
    this.typingTimer = setTimeout(() => {
      this.isCurrentlyTyping = false;
      this.emitStopTyping();
    }, TYPING_STOP_DELAY);
  }

  private emitStartTyping() {
    // In production: this.socketService.emit('typing:start', { roomId })
    console.log('[typing] start');
  }

  private emitStopTyping() {
    // In production: this.socketService.emit('typing:stop', { roomId })
    console.log('[typing] stop');
  }

  // ── Send ──────────────────────────────────────────────────────────
  handleSubmit(e: Event) {
    e.preventDefault();
    const text = this.draft().trim();
    if (!text) return;
    this.draft.set('');
    // Clear typing timer on send
    if (this.typingTimer) clearTimeout(this.typingTimer);
    if (this.isCurrentlyTyping) { this.isCurrentlyTyping = false; this.emitStopTyping(); }
    this.chatService.sendMessage(text, this.myUserId, this.mockServer);
  }

  handleEnter(e: KeyboardEvent) {
    if (!e.shiftKey) this.handleSubmit(e);
  }

  // ── Helpers ───────────────────────────────────────────────────────
  isOutbound(msg: ChatMessage) { return msg.authorId === this.myUserId; }

  bubbleClass(msg: ChatMessage) {
    const dir = this.isOutbound(msg) ? 'outbound' : 'inbound';
    return `chat-message chat-message--${dir} chat-message--${msg.status}`;
  }

  statusLabel(s: DeliveryStatus) { return STATUS_LABEL[s]; }
  statusIcon(s: DeliveryStatus)  { return STATUS_ICON[s]; }
  trackById(_: number, msg: ChatMessage) { return msg.id; }

  // Mock server — replace with real HTTP/WebSocket call
  private mockServer = (text: string): Promise<{ id: string }> =>
    new Promise(resolve => setTimeout(() => resolve({ id: `srv-${Date.now()}` }), 600));
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Message list is announced to screen readers | `role="log"` + `aria-live="polite"` — new messages are announced without interrupting |
| Typing indicator is announced without stealing focus | `role="status"` (implicit `aria-live="polite"`) on the typing bubble |
| Each message's delivery status is accessible | `aria-label` on the status span: "Sent", "Delivered", "Read", "Failed to send" |
| Composer textarea has a label | `aria-label="Message text"` — no visible label needed given placeholder, but aria-label is required |
| Send button accessible when disabled | `disabled` attribute is present; screen reader announces "Send, dimmed" |
| Failed message is distinguishable | CSS class changes visual treatment; `aria-label` on status span says "Failed to send" |
| Scroll does not force screen reader to jump | `aria-live="polite"` queues announcements — never interrupts current speech |
| Typing indicator disappears cleanly | When the `remoteTyping` flag becomes false, the element is removed from DOM, ending any pending announcement |
| Enter key sends without form submission side effects | `e.preventDefault()` on Enter prevents page reload; Shift+Enter inserts newline |
| Minimum tap target on Send button | `padding: 0.5rem 1.25rem` on a `<button>` — reaches 44px height at default font size |

---

## Production Pitfalls

**1. Scroll anchor fires on every render, not just on new messages**
If you call `scrollIntoView` inside `useEffect` or `ngAfterViewChecked` without a guard, the view will snap to the bottom whenever *any* state changes — including the delivery status updating from "sending" to "sent". Fix: tie the scroll trigger to a count of messages, not to the full message array reference. In React, pass `messageCount` to `useScrollAnchor` and only trigger scroll in `useLayoutEffect([messageCount])`. In Angular, compare the count before and after in `ngAfterViewChecked`.

**2. Temp ID collisions when tabs/windows share storage**
Using `Date.now()` as a temp ID is nearly unique in a single tab, but two rapid sends within the same millisecond will collide. Fix: combine `Date.now()` with a monotonic counter that increments in memory. Never use `Math.random()` alone — it is not guaranteed unique.

**3. Optimistic message never reconciled when the user closes the tab mid-send**
The `sendMessage` promise is in-flight; if the tab closes, the temp message is lost from state, but the server may still process the request. On the next page load the message appears from the server with no temp counterpart. Fix: persist the pending temp-ID-to-server-ID map in `sessionStorage`. On mount, cross-reference any server-loaded messages and remove orphaned optimistic entries.

**4. Typing indicator flickers when the remote user has a slow network**
WebSocket "typing:stop" events can arrive out of order with "typing:start" events on a slow connection. Fix: version typing events with a sequence number or timestamp. Only apply a "stop" event if its timestamp is later than the last "start" event you received.

**5. `scrollIntoView({ behavior: 'smooth' })` causes jank during rapid message bursts**
If 10 messages arrive in 200ms (e.g., during initial history load), each triggers a smooth scroll, creating a stuttering effect. Fix: use `behavior: 'instant'` for the initial history load and only switch to `'smooth'` for new real-time messages. Set a flag (`isInitialLoad`) and clear it after the first render.

**6. The typing-stop debounce timer leaks on unmount**
If the component unmounts while the timer is running (e.g., the user navigates away mid-sentence), the `setTimeout` fires and calls `onStopTyping`, potentially on a stale socket connection. Fix: return a cleanup function from `useEffect` (React) or clear the timer in `ngOnDestroy` (Angular) that calls `clearTimeout(timerRef.current)`.

**7. Delivery status regression: "read" → "delivered"**
If WebSocket events arrive out of order (e.g., a "delivered" event has a higher server-side ID than a "read" event but arrives later), the status can regress backward. Fix: enforce a status state machine. Define a numeric rank: `sending=0, sent=1, delivered=2, read=3, failed=0`. Only apply an update if the incoming status rank is greater than the current status rank for that message.

---

## Interview Angle

**Q: "Walk me through how you'd implement optimistic message sending with proper error recovery."**

The core pattern is: (1) assign a client-generated temp ID, (2) push the message to local state immediately with `status: 'sending'`, (3) fire the async call, (4) on success replace the temp ID with the server's canonical ID and advance status to `'sent'`, (5) on failure set `status: 'failed'` — never silently remove the message, because the user needs to see that their message did not go through. The temp ID must be deterministic enough to be unique within a session (combine `Date.now()` with a monotonic counter) but you should never rely on it beyond the current browser session. The reconciliation step — replacing temp ID with server ID — is what keeps React's `key` prop stable after the server responds; if you swap the key, the bubble will flash (unmount and remount).

**Follow-up: "How does scroll anchoring interact with optimistic updates and status changes?"**

This is the subtlety most candidates miss. Status changes (`sending` → `sent` → `delivered` → `read`) modify existing messages — they do not add new ones. If your scroll anchor is triggered by any state change in the messages array, a "read" receipt will snap the scroll position to the bottom even while the user is reading history from 2 days ago. The fix is to anchor only on `messages.length` changing, not on the content of existing messages. In React: `useLayoutEffect([messageCount])`. In Angular: track `previousCount` in `ngAfterViewChecked` and only scroll if `currentCount > previousCount`. Additionally, always respect the "are we at the bottom?" guard — never auto-scroll if the user has scrolled up more than the threshold.

**Follow-up: "How would you prevent the typing indicator from flickering?"**

Two layers of debouncing are needed. First, on the *sender* side: emit "typing:start" on the first keystroke, then reset a `setTimeout` on every subsequent keystroke. Only emit "typing:stop" once the timer fires — not on every key release. This means the server sees at most one "start" event per burst, not one per character. Second, on the *receiver* side: do not immediately hide the typing indicator when a "stop" event arrives; instead, apply a short local delay (300–500ms) as a smoothing buffer for network jitter. Combined, these two debounces prevent the "..." bubble from appearing and disappearing with every word the remote user types.
