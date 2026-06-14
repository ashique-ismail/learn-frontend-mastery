# Live Region Announcements

## The Idea

**In plain English:** When something changes on screen — a search result count updates, a form field shows an error, data finishes loading — a sighted user sees it immediately. A screen reader user hears nothing unless you explicitly tell the browser "this region can change, and when it does, read it aloud." That is an ARIA live region: a DOM node the browser monitors, announcing mutations to the screen reader automatically without moving focus.

**Real-world analogy:** Think of a radio station and a sports scoreboard.

- The **scoreboard** = your live region element. It sits in one place and its content changes.
- The **radio announcer** = the screen reader. They are not watching the scoreboard directly; they are reading from a dedicated feed.
- `aria-live="polite"` = the announcer waits for the current sentence to finish, then reads the score update during the next natural pause.
- `aria-live="assertive"` = the announcer cuts in immediately, mid-sentence, to read a breaking alert. Use only for genuine emergencies (payment failed, session expiring).
- `aria-atomic="true"` = the announcer reads the entire scoreboard ("Arsenal 2 - Chelsea 1"), not just the digit that changed ("1").
- **Clearing then setting** = the equivalent of the scoreboard briefly going blank before showing the new score — this is the only reliable way to re-announce the same text.

The key insight: live regions are a one-way broadcast channel. You write text in, the screen reader reads it out. You are never guaranteed timing — you can only control urgency (`polite` vs `assertive`) and granularity (`atomic`).

---

## Learning Objectives

- Know when to use `aria-live="polite"` vs `"assertive"` and when to use `role="status"` / `role="alert"`
- Understand `aria-atomic` and `aria-relevant`
- Implement announcements for: search result counts, inline form errors, loading states, toast notifications
- Avoid the most common live region bugs: pre-rendered content, wrong region, double announcement
- Integrate live regions cleanly in React and Angular

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Show a visible error message below an input | ✅ with generated content / animations | — |
| Announce that error to a screen reader | ❌ | Requires `aria-live` or `aria-describedby` |
| Re-announce the same message a second time | ❌ | Requires JS clear-then-set pattern |
| Announce only after an async request completes | ❌ | Requires JS promise resolution |
| Choose polite vs assertive based on severity | ❌ | CSS has no concept of announcement urgency |
| Track result count updates in real-time | ❌ | CSS cannot observe data state |

---

## HTML & CSS Foundation

```html
<!-- role="status" is shorthand for aria-live="polite" aria-atomic="true" -->
<div role="status" id="search-status" class="sr-only"></div>

<!-- role="alert" is shorthand for aria-live="assertive" aria-atomic="true" -->
<div role="alert" id="error-alert" class="sr-only"></div>

<!-- Explicit live region with fine-grained control -->
<div
  aria-live="polite"
  aria-atomic="false"
  aria-relevant="additions text"
  id="log-region"
  class="sr-only"
></div>

<!-- Visible status bar (combines live region with visible UI) -->
<div
  role="status"
  class="search-status-bar"
  aria-live="polite"
  aria-atomic="true"
>
  <!-- JS will write the text content here -->
</div>
```

```css
/* ── Screen-reader-only region ── */
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

/* ── Visible status bar ── */
.search-status-bar {
  font-size: 0.875rem;
  color: #555;
  padding: 0.5rem 0;
  min-height: 1.5rem; /* reserve space so layout doesn't jump */
}

/* ── Inline error message (below a form field) ── */
.field-error {
  display: block;
  font-size: 0.8125rem;
  color: #c0392b;
  margin-top: 0.25rem;
}

/* ── Toast notifications ── */
.toast-container {
  position: fixed;
  bottom: 1.5rem;
  right: 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  z-index: 9000;
}

.toast {
  padding: 0.75rem 1.25rem;
  background: #1a1a1a;
  color: #fff;
  border-radius: 6px;
  font-size: 0.9375rem;
  animation: toast-in 0.25s ease;
}

.toast--error { background: #c0392b; }
.toast--success { background: #27ae60; }

@keyframes toast-in {
  from { opacity: 0; transform: translateY(8px); }
  to   { opacity: 1; transform: translateY(0);   }
}
```

---

## React Implementation

### The `useAnnouncer` Hook

```tsx
// hooks/useAnnouncer.ts
import { useCallback, useEffect, useRef } from 'react';

type Politeness = 'polite' | 'assertive';

interface AnnouncerOptions {
  politeness?: Politeness;
  clearDelay?: number; // ms to wait before clearing (default 5000)
}

/**
 * Returns an `announce(message)` function.
 * Each politeness level manages its own DOM node.
 */
export function useAnnouncer(options: AnnouncerOptions = {}) {
  const { politeness = 'polite', clearDelay = 5000 } = options;
  const regionRef = useRef<HTMLElement | null>(null);
  const clearTimer = useRef<number | undefined>(undefined);

  useEffect(() => {
    const id = `announcer-${politeness}`;
    let el = document.getElementById(id) as HTMLElement | null;

    if (!el) {
      el = document.createElement('div');
      el.id = id;
      el.setAttribute('aria-live', politeness);
      el.setAttribute('aria-atomic', 'true');
      el.className = 'sr-only';
      document.body.appendChild(el);
    }

    regionRef.current = el;

    return () => {
      clearTimeout(clearTimer.current);
    };
  }, [politeness]);

  const announce = useCallback(
    (message: string) => {
      const el = regionRef.current;
      if (!el) return;

      // Clear first so re-announcing the same message triggers a new read
      el.textContent = '';

      // Yield to the browser before setting the new text
      requestAnimationFrame(() => {
        el.textContent = message;

        // Auto-clear so stale messages don't confuse users who navigate back
        clearTimeout(clearTimer.current);
        clearTimer.current = window.setTimeout(() => {
          el.textContent = '';
        }, clearDelay);
      });
    },
    [clearDelay]
  );

  return announce;
}
```

### Search Results Count

```tsx
// components/SearchResults.tsx
import { useState, useEffect } from 'react';
import { useAnnouncer } from '../hooks/useAnnouncer';

interface SearchResultsProps {
  query: string;
  results: { id: string; title: string }[];
  isLoading: boolean;
}

export function SearchResults({ query, results, isLoading }: SearchResultsProps) {
  const announce = useAnnouncer({ politeness: 'polite' });

  useEffect(() => {
    if (isLoading) {
      // Don't announce while loading — wait for the final count
      return;
    }
    if (!query) return;

    if (results.length === 0) {
      announce(`No results found for "${query}"`);
    } else {
      announce(`${results.length} result${results.length === 1 ? '' : 's'} found for "${query}"`);
    }
  }, [results, isLoading, query, announce]);

  return (
    <section aria-label="Search results">
      {/* Visible status — also serves as a live region for the visible UI */}
      <p
        role="status"
        aria-live="polite"
        aria-atomic="true"
        className="search-status-bar"
      >
        {isLoading
          ? 'Searching…'
          : query && results.length === 0
          ? `No results for "${query}"`
          : query
          ? `${results.length} result${results.length === 1 ? '' : 's'}`
          : ''}
      </p>

      {isLoading ? (
        <p aria-busy="true" aria-label="Loading results">
          {/* Spinner graphic */}
        </p>
      ) : (
        <ul>
          {results.map(r => (
            <li key={r.id}>{r.title}</li>
          ))}
        </ul>
      )}
    </section>
  );
}
```

### Form Inline Errors

```tsx
// components/AccessibleForm.tsx
import { useState, useId } from 'react';
import { useAnnouncer } from '../hooks/useAnnouncer';

export function AccessibleForm() {
  const [email, setEmail] = useState('');
  const [emailError, setEmailError] = useState('');
  const [serverError, setServerError] = useState('');
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [success, setSuccess] = useState(false);

  const announceAssertive = useAnnouncer({ politeness: 'assertive' });
  const announcePolite = useAnnouncer({ politeness: 'polite' });

  const emailId = useId();
  const emailErrorId = `${emailId}-error`;

  function validateEmail(value: string): string {
    if (!value) return 'Email is required';
    if (!value.includes('@')) return 'Enter a valid email address';
    return '';
  }

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    const error = validateEmail(email);

    if (error) {
      setEmailError(error);
      // Assertive: interrupt whatever the screen reader is doing
      announceAssertive(`Form error: ${error}`);
      // Also move focus to the field (see FORM-03 for full focus management)
      document.getElementById(emailId)?.focus();
      return;
    }

    setEmailError('');
    setIsSubmitting(true);
    announcePolite('Submitting…');

    try {
      await fakeSubmit(email);
      setSuccess(true);
      announcePolite('Form submitted successfully');
    } catch {
      const msg = 'Submission failed. Please try again.';
      setServerError(msg);
      announceAssertive(msg);
    } finally {
      setIsSubmitting(false);
    }
  }

  return (
    <form onSubmit={handleSubmit} noValidate>
      {/* Server-level error: role="alert" so it fires immediately */}
      {serverError && (
        <div role="alert" className="field-error" aria-atomic="true">
          {serverError}
        </div>
      )}

      <div>
        <label htmlFor={emailId}>Email address</label>
        <input
          id={emailId}
          type="email"
          value={email}
          onChange={e => setEmail(e.target.value)}
          aria-describedby={emailError ? emailErrorId : undefined}
          aria-invalid={emailError ? true : undefined}
          aria-required="true"
          autoComplete="email"
        />
        {/* Inline error: NOT a live region — linked via aria-describedby */}
        {emailError && (
          <span id={emailErrorId} className="field-error" role="alert">
            {emailError}
          </span>
        )}
      </div>

      <button type="submit" disabled={isSubmitting} aria-busy={isSubmitting}>
        {isSubmitting ? 'Submitting…' : 'Submit'}
      </button>

      {success && (
        <p role="status">Thank you! Check your inbox.</p>
      )}
    </form>
  );
}

function fakeSubmit(email: string): Promise<void> {
  return new Promise((res, rej) =>
    setTimeout(() => (email.includes('error') ? rej() : res()), 800)
  );
}
```

### Toast Notification System

```tsx
// hooks/useToast.ts
import { useState, useCallback, useId } from 'react';

interface Toast {
  id: string;
  message: string;
  type: 'info' | 'success' | 'error';
}

export function useToast() {
  const [toasts, setToasts] = useState<Toast[]>([]);
  const baseId = useId();

  const addToast = useCallback(
    (message: string, type: Toast['type'] = 'info', duration = 5000) => {
      const id = `${baseId}-${Date.now()}`;
      setToasts(prev => [...prev, { id, message, type }]);
      setTimeout(() => {
        setToasts(prev => prev.filter(t => t.id !== id));
      }, duration);
    },
    [baseId]
  );

  return { toasts, addToast };
}

// components/ToastRegion.tsx
import type { ReturnType } from '../types';

interface ToastRegionProps {
  toasts: { id: string; message: string; type: string }[];
}

/**
 * The container uses role="log" (implied aria-live="polite") for info/success.
 * Errors use a sibling role="alert" region so they interrupt immediately.
 */
export function ToastRegion({ toasts }: ToastRegionProps) {
  const errors = toasts.filter(t => t.type === 'error');
  const others = toasts.filter(t => t.type !== 'error');

  return (
    <>
      {/* Polite region for info / success toasts */}
      <div role="log" aria-live="polite" aria-atomic="false" className="toast-container">
        {others.map(t => (
          <div key={t.id} className={`toast toast--${t.type}`}>
            {t.message}
          </div>
        ))}
      </div>

      {/* Assertive region for error toasts only */}
      <div role="alert" aria-live="assertive" aria-atomic="true" className="toast-container" style={{ bottom: '5rem' }}>
        {errors.map(t => (
          <div key={t.id} className="toast toast--error">
            {t.message}
          </div>
        ))}
      </div>
    </>
  );
}
```

---

## Angular Implementation

### Announcer Service

```typescript
// services/announcer.service.ts
import { Injectable, OnDestroy } from '@angular/core';

type Politeness = 'polite' | 'assertive';

@Injectable({ providedIn: 'root' })
export class AnnouncerService implements OnDestroy {
  private regions = new Map<Politeness, HTMLElement>();
  private timers = new Map<Politeness, ReturnType<typeof setTimeout>>();

  announce(message: string, politeness: Politeness = 'polite', clearDelay = 5000): void {
    const el = this.getOrCreate(politeness);

    // Clear, then yield, then set — ensures re-announcement of same text
    el.textContent = '';
    requestAnimationFrame(() => {
      el.textContent = message;

      clearTimeout(this.timers.get(politeness));
      this.timers.set(
        politeness,
        setTimeout(() => { el.textContent = ''; }, clearDelay)
      );
    });
  }

  private getOrCreate(politeness: Politeness): HTMLElement {
    if (this.regions.has(politeness)) return this.regions.get(politeness)!;

    const el = document.createElement('div');
    el.setAttribute('aria-live', politeness);
    el.setAttribute('aria-atomic', 'true');
    el.style.cssText =
      'position:absolute;width:1px;height:1px;margin:-1px;overflow:hidden;clip:rect(0,0,0,0);white-space:nowrap;border:0';
    document.body.appendChild(el);
    this.regions.set(politeness, el);
    return el;
  }

  ngOnDestroy(): void {
    this.timers.forEach(t => clearTimeout(t));
    this.regions.forEach(el => el.remove());
  }
}
```

### Search Component

```typescript
// components/product-search/product-search.component.ts
import { Component, signal, effect, inject } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { AnnouncerService } from '../../services/announcer.service';

@Component({
  selector: 'app-product-search',
  standalone: true,
  imports: [FormsModule],
  template: `
    <form (submit)="onSearch($event)" role="search" aria-label="Product search">
      <label for="search-input">Search products</label>
      <input
        id="search-input"
        type="search"
        [(ngModel)]="query"
        name="query"
        autocomplete="off"
      />
      <button type="submit">Search</button>
    </form>

    <!-- Visible live region — also announced by screen reader -->
    <p role="status" aria-live="polite" aria-atomic="true" class="search-status-bar">
      {{ statusText() }}
    </p>

    <ul *ngIf="!loading() && results().length > 0" aria-label="Search results">
      <li *ngFor="let r of results()">{{ r.title }}</li>
    </ul>
  `,
})
export class ProductSearchComponent {
  private announcer = inject(AnnouncerService);

  query = '';
  results = signal<{ id: string; title: string }[]>([]);
  loading = signal(false);

  statusText = () => {
    if (this.loading()) return 'Searching…';
    if (!this.query) return '';
    const n = this.results().length;
    return n === 0
      ? `No results for "${this.query}"`
      : `${n} result${n === 1 ? '' : 's'} for "${this.query}"`;
  };

  async onSearch(e: Event) {
    e.preventDefault();
    if (!this.query.trim()) return;

    this.loading.set(true);
    this.announcer.announce('Searching…');

    try {
      // Replace with real HTTP call
      const data = await fakeSearch(this.query);
      this.results.set(data);

      const n = data.length;
      const msg = n === 0
        ? `No results found for "${this.query}"`
        : `${n} result${n === 1 ? '' : 's'} found for "${this.query}"`;

      this.announcer.announce(msg);
    } finally {
      this.loading.set(false);
    }
  }
}

function fakeSearch(q: string): Promise<{ id: string; title: string }[]> {
  return new Promise(res =>
    setTimeout(() => res(q.length > 3 ? [{ id: '1', title: 'Product A' }, { id: '2', title: 'Product B' }] : []), 600)
  );
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| `role="status"` for non-urgent updates | Search counts, loading complete, success messages |
| `role="alert"` for urgent updates | Auth errors, payment failures, session expiry |
| `aria-live="polite"` for queue-based announcements | Same as `role="status"` — use roles when possible |
| `aria-live="assertive"` reserved for interruptions | Only genuine emergencies; causes VoiceOver to cut in |
| `aria-atomic="true"` to read the full region | Prevents partial reads when multiple text nodes update |
| Clear-then-set to re-trigger same text | `el.textContent = ''; rAF(() => el.textContent = msg)` |
| Live region exists in DOM before content is written | Must be mounted before JS writes to it |
| Inline errors use `aria-describedby`, not live regions | Screen reader reads error on focus; live region reads it on change |
| `aria-invalid="true"` on failing fields | Combined with `aria-describedby` pointing to the error span |
| Auto-clear after N seconds | Prevent stale messages from re-announcing on page revisit |

---

## Production Pitfalls

**1. Writing to a live region before it is mounted**
If you server-render the live region with text in it, most screen readers do not announce it (they only watch for changes, not initial content). Always render live regions empty, then write to them with JS.

**2. Using `aria-live="assertive"` for routine messages**
`assertive` interrupts the screen reader mid-sentence. Using it for search result counts or loading messages is aggressive and confusing. Reserve it for session timeouts, payment errors, and genuinely destructive or time-sensitive events.

**3. Forgetting `aria-atomic`**
Without `aria-atomic="true"`, when text changes inside a region containing multiple text nodes, NVDA may announce only the changed node. For a region like "12 results found", setting atomic ensures the whole sentence is read.

**4. Mounting multiple live regions with the same `aria-live` value**
This is fine — each region is independent. But mounting a region inside a component that re-mounts creates a new DOM element each time, leaving orphans. Use a singleton service pattern (as shown above) that lazily creates a single element in `document.body`.

**5. Inline form errors should NOT use live regions**
Putting `aria-live="assertive"` on every field error paragraph causes the screen reader to interrupt when any error appears. Instead, link errors via `aria-describedby` — the reader announces them when focus lands on the field — and use a single live region to summarize all errors after submit.

**6. React StrictMode and double-invocation**
In React 18 StrictMode, effects run twice in development. Your announcer hook may create two DOM elements. The `getOrCreate` pattern (checking by `id` before creating) prevents duplicates.
