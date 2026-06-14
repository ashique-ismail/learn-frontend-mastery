# Infinite Scroll with Intersection Observer

## The Idea

**In plain English:** Instead of a "Load More" button, a page automatically fetches the next chunk of content when the user scrolls near the bottom. The cleanest implementation attaches a tiny invisible "sentinel" element at the bottom of the list. An Intersection Observer watches the sentinel: when it enters the viewport, the next page fetch fires. The hard parts are handling prepend-without-scroll-jump for reverse-chronological feeds (CSS `overflow-anchor` / manual `scrollHeight` bookkeeping), cleaning up the observer so it does not leak after the component unmounts, restoring the exact scroll position when the user presses Back, and exposing distinct loading / error / end-of-list states per page request.

**Real-world analogy:** Think of a reading room with a moving bookshelf.

- The **bookshelf** is the content list — it appears infinitely long because the assistant keeps extending it.
- You sit in a chair and read whatever is in front of you. As you near the blank label at the very bottom of the shelf, an assistant (Intersection Observer) notices that label has entered your eyeline.
- The assistant immediately slides more books onto the end of the shelf — **loading the next page** — without disturbing the books you are currently reading.
- **Back button as a door:** you walked through that door earlier and left a bookmark at eye-level. When you come back through the door, the assistant turns the shelf to exactly that bookmark. This is **scroll restoration**.
- **Prepending without a jump:** if the assistant also adds books above you (a live feed inserting new posts at the top), your chair must not move. The books shift up but you stay at exactly the same reading spot. This is `overflow-anchor`.

---

## Learning Objectives

- Implement the sentinel-element pattern using `IntersectionObserver` for triggered pagination
- Understand why `rootMargin` in pixels beats percentages for early pre-fetching
- Prepend items without a scroll jump using `overflow-anchor: auto` (CSS) and the manual `scrollHeight` delta technique (JS fallback)
- Set `history.scrollRestoration = 'manual'` and save/restore `scrollTop` to `history.state` for Back-button restoration
- Clean up the observer on component unmount to prevent memory and callback leaks
- Model loading, error, and end-of-list state independently per page — not as a single boolean flag
- Abort in-flight fetches when the component unmounts or the user navigates away

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
| --- | --- | --- |
| Detect when sentinel enters viewport | ❌ | Requires `IntersectionObserver` API |
| Fetch next page on scroll event | ❌ | Requires JS `fetch` / HTTP client |
| Prevent scroll jump on prepend | ⚠️ | `overflow-anchor: auto` helps; JS bookkeeping required for full control |
| Restore scroll position on Back | ❌ | Requires reading and writing `history.state` |
| Abort stale requests | ❌ | Requires `AbortController` |
| Show per-page loading or error UI | ✅ with class toggle | CSS can style it; JS must toggle the class |

---

## HTML & CSS Foundation

### Structure

```html
<!--
  If the scroll container is the full page, overflow-y on this div should be visible.
  If it is a panel with a fixed height, add a height and overflow-y: auto.
  The sentinel MUST live inside the same scroll root as the cards.
-->
<div class="feed-scroll-container" id="scroll-container">

  <!-- Feed items rendered by JS -->
  <article class="feed-card" data-id="post-1">
    <h3 class="feed-card__title">Post Title</h3>
    <p class="feed-card__body">Content goes here…</p>
  </article>

  <!--
    Sentinel: the only element the observer watches.
    aria-hidden keeps it out of the accessibility tree.
    overflow-anchor: none prevents the browser from pinning to it
    when items are appended right after it.
  -->
  <div class="scroll-sentinel" id="scroll-sentinel" aria-hidden="true"></div>

  <!-- Per-page loading indicator -->
  <div
    class="scroll-loading"
    role="status"
    aria-live="polite"
    aria-label="Loading more content"
    hidden
  >
    <span class="scroll-spinner" aria-hidden="true"></span>
    Loading…
  </div>

  <!-- Per-page error -->
  <div
    class="scroll-error"
    role="alert"
    hidden
  >
    <p class="scroll-error__message">Failed to load posts.</p>
    <button class="scroll-error__retry">Try again</button>
  </div>

  <!-- End of list -->
  <div class="scroll-end" role="status" aria-live="polite" hidden>
    All posts loaded
  </div>
</div>
```

### CSS

```css
/* ─── Tokens ─── */
:root {
  --feed-max-width: 680px;
  --card-radius:    8px;
  --card-border:    1px solid #e5e7eb;
  --card-bg:        #ffffff;
  --muted:          #9ca3af;
  --error-fg:       #dc2626;
}

/* ─── Scroll container ─── */
.feed-scroll-container {
  max-width: var(--feed-max-width);
  margin: 0 auto;
  /*
    overflow-anchor: auto  — browser pins scroll to content above the viewport
    when new items are appended. This alone prevents jump for append-at-bottom.
    For prepend-at-top, see the JS prepend technique in the React section.
  */
  overflow-anchor: auto;
}

/* ─── Feed card ─── */
.feed-card {
  padding: 1.25rem;
  border: var(--card-border);
  border-radius: var(--card-radius);
  margin-bottom: 1rem;
  background: var(--card-bg);
  animation: card-in 0.2s ease;
}

@keyframes card-in {
  from { opacity: 0; transform: translateY(8px); }
  to   { opacity: 1; transform: translateY(0);   }
}

.feed-card__title { margin: 0 0 0.5rem; font-size: 1rem; font-weight: 600; }
.feed-card__body  { margin: 0; color: #4b5563; font-size: 0.9rem; line-height: 1.6; }

/* ─── Sentinel ─── */
/*
  Must have non-zero height so IntersectionObserver can observe it.
  overflow-anchor: none stops the browser choosing the sentinel as its
  anchor element when items are inserted after it.
*/
.scroll-sentinel {
  height: 1px;
  width: 100%;
  overflow-anchor: none;
}

/* ─── Loading indicator ─── */
.scroll-loading {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.75rem;
  padding: 1.5rem;
  color: var(--muted);
  font-size: 0.875rem;
}

.scroll-spinner {
  flex-shrink: 0;
  width: 20px;
  height: 20px;
  border: 2px solid #e5e7eb;
  border-top-color: #6b7280;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
}

@keyframes spin { to { transform: rotate(360deg); } }

/* ─── Error state ─── */
.scroll-error {
  padding: 1rem;
  text-align: center;
}

.scroll-error__message {
  margin: 0 0 0.5rem;
  color: var(--error-fg);
  font-size: 0.875rem;
}

.scroll-error__retry {
  padding: 0.4rem 1rem;
  border: 1px solid var(--error-fg);
  border-radius: 4px;
  background: none;
  color: var(--error-fg);
  cursor: pointer;
  font-size: 0.875rem;
}

/* ─── End-of-list message ─── */
.scroll-end {
  padding: 1.5rem;
  text-align: center;
  color: var(--muted);
  font-size: 0.875rem;
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .feed-card    { animation: none; }
  .scroll-spinner { animation: none; border-top-color: #6b7280; }
}
```

**What CSS owns:** card entry animation, sentinel height and anchor exclusion, spinner keyframes, error and end-of-list visual states, reduced-motion guard.

**What CSS cannot own:** detecting viewport intersection, triggering fetches, aborting requests, scroll position saving or restoration.

---

## React Implementation

### Types

```tsx
// types.ts
export interface Post {
  id:    string;
  title: string;
  body:  string;
}

export interface PageResult {
  posts:   Post[];
  hasMore: boolean;
}

// Discriminated union gives per-page loading AND error states
export type PageStatus = 'idle' | 'loading' | 'error' | 'done';
```

### Reusable Intersection Observer Hook

```tsx
// useIntersectionObserver.ts
import { useEffect, useRef, useState } from 'react';

interface IOOptions {
  root?:       Element | null;
  rootMargin?: string;       // e.g. '300px' — start loading before sentinel hits viewport
  threshold?:  number | number[];
}

export function useIntersectionObserver(options: IOOptions = {}) {
  const targetRef = useRef<HTMLDivElement | null>(null);
  const [isIntersecting, setIsIntersecting] = useState(false);

  useEffect(() => {
    const el = targetRef.current;
    if (!el) return;

    const observer = new IntersectionObserver(
      ([entry]) => setIsIntersecting(entry.isIntersecting),
      {
        root:       options.root       ?? null,
        rootMargin: options.rootMargin ?? '0px',
        threshold:  options.threshold  ?? 0,
      }
    );

    observer.observe(el);

    // Cleanup: disconnect on unmount or options change
    return () => {
      observer.disconnect();
    };
  }, [options.root, options.rootMargin, options.threshold]);

  return { ref: targetRef, isIntersecting };
}
```

### Infinite Scroll Data Hook

```tsx
// useInfiniteScroll.ts
import { useState, useEffect, useCallback, useRef } from 'react';
import type { Post, PageStatus } from './types';

interface ScrollState {
  posts:     Post[];
  page:      number;
  status:    PageStatus;
  error:     string | null;
  hasMore:   boolean;
}

async function fetchPage(
  page: number,
  signal: AbortSignal
): Promise<{ posts: Post[]; hasMore: boolean }> {
  const res = await fetch(`/api/posts?page=${page}&pageSize=20`, { signal });
  if (!res.ok) throw new Error(`Server error ${res.status} on page ${page}`);
  return res.json() as Promise<{ posts: Post[]; hasMore: boolean }>;
}

export function useInfiniteScroll() {
  const [state, setState] = useState<ScrollState>({
    posts:   [],
    page:    0,
    status:  'idle',
    error:   null,
    hasMore: true,
  });

  // Keep a ref to the latest AbortController so we can cancel on unmount
  const abortRef = useRef<AbortController | null>(null);

  const loadNext = useCallback(() => {
    setState(prev => {
      // Guard: do not fire while loading or when there is nothing left
      if (prev.status === 'loading' || !prev.hasMore) return prev;
      return { ...prev, status: 'loading', error: null };
    });
  }, []);

  const retry = useCallback(() => {
    setState(prev => ({ ...prev, status: 'loading', error: null }));
  }, []);

  // Execute the fetch whenever status transitions to 'loading'
  useEffect(() => {
    if (state.status !== 'loading') return;

    // Cancel any in-flight request from the previous call
    abortRef.current?.abort();
    abortRef.current = new AbortController();
    const { signal } = abortRef.current;

    fetchPage(state.page, signal)
      .then(({ posts, hasMore }) => {
        setState(prev => ({
          ...prev,
          posts:   [...prev.posts, ...posts],
          page:    prev.page + 1,
          hasMore,
          status:  hasMore ? 'idle' : 'done',
          error:   null,
        }));
      })
      .catch(err => {
        if (err.name === 'AbortError') return; // unmount cleanup, ignore
        setState(prev => ({
          ...prev,
          status: 'error',
          error:  err instanceof Error ? err.message : 'Unknown error',
        }));
      });
  }, [state.status, state.page]);

  // Abort on unmount
  useEffect(() => {
    return () => abortRef.current?.abort();
  }, []);

  const isLoading = state.status === 'loading';
  const isError   = state.status === 'error';

  return { ...state, isLoading, isError, loadNext, retry };
}
```

### Scroll Restoration Hook

```tsx
// useScrollRestoration.ts
import { useEffect } from 'react';

/**
 * Opts out of browser automatic scroll restoration and takes manual control.
 * Saves scrollTop to history.state on each scroll (debounced).
 * Restores scrollTop from history.state on mount (Back-button navigation).
 */
export function useScrollRestoration(containerId: string) {
  // Take over from the browser — without this, Chrome may reset scrollTop to 0
  // before our effect can restore it
  useEffect(() => {
    if ('scrollRestoration' in history) {
      history.scrollRestoration = 'manual';
    }
  }, []);

  // Restore on mount
  useEffect(() => {
    const saved = (window.history.state as Record<string, number> | null)?.[containerId];
    if (typeof saved === 'number' && saved > 0) {
      // rAF: wait for the first paint so items exist in the DOM before scrolling
      requestAnimationFrame(() => {
        const el = document.getElementById(containerId);
        if (el) el.scrollTop = saved;
      });
    }
  }, [containerId]);

  // Save on scroll (debounced, passive listener)
  useEffect(() => {
    const el = document.getElementById(containerId);
    if (!el) return;

    let timer: ReturnType<typeof setTimeout> | undefined;

    const onScroll = () => {
      clearTimeout(timer);
      timer = setTimeout(() => {
        // history.replaceState merges with existing state so other keys are preserved
        history.replaceState(
          { ...(history.state ?? {}), [containerId]: el.scrollTop },
          ''
        );
      }, 150);
    };

    el.addEventListener('scroll', onScroll, { passive: true });

    return () => {
      el.removeEventListener('scroll', onScroll);
      clearTimeout(timer);
    };
  }, [containerId]);
}
```

### Prepend Without Scroll Jump Utility

```tsx
// prependWithoutJump.ts
/**
 * Prepend items to a scroll container without changing the user's reading position.
 * CSS overflow-anchor handles append; this handles prepend (e.g. live feeds).
 *
 * Usage:
 *   const restore = captureScrollAnchor(containerEl);
 *   // mutate the DOM — prepend items
 *   restore();
 */
export function captureScrollAnchor(container: HTMLElement) {
  const before = container.scrollHeight - container.scrollTop;

  return function restoreScrollAnchor() {
    // After the DOM mutation, scrollHeight has grown by the height of prepended items.
    // Adjust scrollTop so the delta (scrollHeight - scrollTop) stays the same.
    container.scrollTop = container.scrollHeight - before;
  };
}
```

### The Component

```tsx
// InfinitePostFeed.tsx
import { useEffect } from 'react';
import { useIntersectionObserver } from './useIntersectionObserver';
import { useInfiniteScroll } from './useInfiniteScroll';
import { useScrollRestoration } from './useScrollRestoration';

const CONTAINER_ID = 'scroll-container';

export function InfinitePostFeed() {
  const { posts, hasMore, isLoading, isError, error, status, loadNext, retry } =
    useInfiniteScroll();

  useScrollRestoration(CONTAINER_ID);

  // Sentinel sits 300px below the viewport — pre-fetches before the user reaches the end
  const { ref: sentinelRef, isIntersecting } = useIntersectionObserver({
    rootMargin: '300px',
  });

  // Trigger loadNext when sentinel enters the extended viewport
  useEffect(() => {
    if (isIntersecting && hasMore && !isLoading) {
      loadNext();
    }
  }, [isIntersecting, hasMore, isLoading, loadNext]);

  // Load page 0 on mount
  useEffect(() => {
    loadNext();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  return (
    <div className="feed-scroll-container" id={CONTAINER_ID}>
      {posts.map(post => (
        <article key={post.id} className="feed-card" data-id={post.id}>
          <h3 className="feed-card__title">{post.title}</h3>
          <p className="feed-card__body">{post.body}</p>
        </article>
      ))}

      {/*
        Sentinel is always in the DOM so the observer target does not unmount.
        Placing it before the loading/error/end states means it is outside those
        elements and the observer fires even if loading takes a long time.
      */}
      <div ref={sentinelRef} className="scroll-sentinel" aria-hidden="true" />

      {isLoading && (
        <div className="scroll-loading" role="status" aria-live="polite">
          <span className="scroll-spinner" aria-hidden="true" />
          Loading…
        </div>
      )}

      {isError && (
        <div className="scroll-error" role="alert">
          <p className="scroll-error__message">{error}</p>
          <button className="scroll-error__retry" onClick={retry}>
            Try again
          </button>
        </div>
      )}

      {status === 'done' && posts.length > 0 && (
        <div className="scroll-end" role="status" aria-live="polite">
          All {posts.length} posts loaded
        </div>
      )}

      {status === 'done' && posts.length === 0 && (
        <p style={{ textAlign: 'center', color: '#9ca3af' }}>No posts found.</p>
      )}
    </div>
  );
}
```

---

## Angular Implementation

### Service

```typescript
// infinite-feed.service.ts
import { Injectable, signal, computed, inject } from '@angular/core';
import { HttpClient, HttpErrorResponse } from '@angular/common/http';
import { switchMap, tap, catchError, of, Subject } from 'rxjs';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

export interface Post { id: string; title: string; body: string; }
export type PageStatus = 'idle' | 'loading' | 'error' | 'done';

@Injectable()
export class InfiniteFeedService {
  private http = inject(HttpClient);

  private _posts    = signal<Post[]>([]);
  private _status   = signal<PageStatus>('idle');
  private _error    = signal<string | null>(null);
  private _page     = 0;

  readonly posts    = this._posts.asReadonly();
  readonly status   = this._status.asReadonly();
  readonly error    = this._error.asReadonly();
  readonly isLoading = computed(() => this._status() === 'loading');
  readonly hasMore   = computed(() => this._status() !== 'done');

  private load$ = new Subject<void>();

  constructor() {
    this.load$.pipe(
      tap(() => this._status.set('loading')),
      switchMap(() =>
        this.http
          .get<{ posts: Post[]; hasMore: boolean }>(
            `/api/posts?page=${this._page}&pageSize=20`
          )
          .pipe(
            catchError((err: HttpErrorResponse) => {
              this._error.set(err.message ?? 'Unknown error');
              this._status.set('error');
              return of(null);
            })
          )
      ),
      takeUntilDestroyed(),
    ).subscribe(data => {
      if (!data) return;
      this._posts.update(prev => [...prev, ...data.posts]);
      this._page++;
      this._status.set(data.hasMore ? 'idle' : 'done');
      this._error.set(null);
    });
  }

  loadNext(): void {
    if (this._status() === 'loading' || this._status() === 'done') return;
    this.load$.next();
  }

  retry(): void {
    if (this._status() !== 'error') return;
    this._status.set('idle');
    this.load$.next();
  }
}
```

### Component

```typescript
// infinite-feed.component.ts
import {
  Component, OnInit, AfterViewInit, OnDestroy,
  ViewChild, ElementRef, inject, DestroyRef
} from '@angular/core';
import { InfiniteFeedService } from './infinite-feed.service';

@Component({
  selector: 'app-infinite-feed',
  standalone: true,
  providers: [InfiniteFeedService],   // component-scoped — destroyed with component
  template: `
    <div class="feed-scroll-container" id="scroll-container">

      @for (post of feed.posts(); track post.id) {
        <article class="feed-card" [attr.data-id]="post.id">
          <h3 class="feed-card__title">{{ post.title }}</h3>
          <p class="feed-card__body">{{ post.body }}</p>
        </article>
      }

      <!-- Sentinel: always present so observer does not need to re-attach -->
      <div #sentinel class="scroll-sentinel" aria-hidden="true"></div>

      @if (feed.isLoading()) {
        <div class="scroll-loading" role="status" aria-live="polite">
          <span class="scroll-spinner" aria-hidden="true"></span>
          Loading…
        </div>
      }

      @if (feed.status() === 'error') {
        <div class="scroll-error" role="alert">
          <p class="scroll-error__message">{{ feed.error() }}</p>
          <button class="scroll-error__retry" (click)="feed.retry()">Try again</button>
        </div>
      }

      @if (feed.status() === 'done' && feed.posts().length > 0) {
        <div class="scroll-end" role="status" aria-live="polite">
          All {{ feed.posts().length }} posts loaded
        </div>
      }

    </div>
  `,
})
export class InfiniteFeedComponent implements OnInit, AfterViewInit, OnDestroy {
  feed = inject(InfiniteFeedService);

  @ViewChild('sentinel') sentinelEl!: ElementRef<HTMLDivElement>;

  private observer: IntersectionObserver | null = null;

  ngOnInit(): void {
    // Disable browser automatic scroll restoration; we manage it manually
    if ('scrollRestoration' in history) {
      history.scrollRestoration = 'manual';
    }

    // Restore scroll position saved before Back navigation
    const saved = (history.state as Record<string, number> | null)?.['scroll-container'];
    if (typeof saved === 'number' && saved > 0) {
      requestAnimationFrame(() => {
        const el = document.getElementById('scroll-container');
        if (el) el.scrollTop = saved;
      });
    }

    this.setupScrollSave();
    this.feed.loadNext(); // initial page
  }

  ngAfterViewInit(): void {
    this.observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting && this.feed.hasMore() && !this.feed.isLoading()) {
          this.feed.loadNext();
        }
      },
      { rootMargin: '300px', threshold: 0 }
    );

    this.observer.observe(this.sentinelEl.nativeElement);
  }

  ngOnDestroy(): void {
    // Disconnect before component teardown to prevent callbacks firing on a
    // destroyed component reference
    this.observer?.disconnect();
    this.observer = null;
    this.scrollSaveCleanup?.();
  }

  // ─── Scroll save (debounced) ───────────────────────────────────────────────
  private scrollSaveCleanup?: () => void;

  private setupScrollSave(): void {
    const el = document.getElementById('scroll-container');
    if (!el) return;

    let timer: ReturnType<typeof setTimeout> | undefined;

    const onScroll = () => {
      clearTimeout(timer);
      timer = setTimeout(() => {
        history.replaceState(
          { ...(history.state ?? {}), 'scroll-container': el.scrollTop },
          ''
        );
      }, 150);
    };

    el.addEventListener('scroll', onScroll, { passive: true });
    this.scrollSaveCleanup = () => {
      el.removeEventListener('scroll', onScroll);
      clearTimeout(timer);
    };
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
| --- | --- |
| Loading state announced to screen readers | `role="status"` + `aria-live="polite"` on loading element |
| Error announced immediately on failure | `role="alert"` — polite assertive announcement with no delay |
| End-of-list announced | `role="status"` + `aria-live="polite"` on done element |
| Sentinel hidden from screen readers | `aria-hidden="true"` — it is purely positional, not content |
| New appended content does not steal focus | Items inserted into DOM; focus stays with the currently focused element |
| Retry button is keyboard-accessible | Standard `<button>` element; no `tabindex` manipulation needed |
| Scroll restoration does not disorient keyboard users | Restoration fires on mount before first interactive paint |
| `prefers-reduced-motion` respected | Card entry animation and spinner animation both disabled in `@media` query |
| Back button history is preserved | `history.replaceState` merges — does not discard previous `history.state` keys |

---

## Production Pitfalls

**1. Sentinel placed outside the scroll container**
If the scroll container is a fixed-height `div` with `overflow-y: auto`, the `IntersectionObserver` root defaults to the browser viewport, not the div. The sentinel will appear visible (relative to the viewport) even when the div has not scrolled at all, firing the callback immediately on mount. Fix: pass `{ root: containerElement }` to the observer options, or ensure the sentinel is inside the same scroll root.

**2. Observer callback fires on mount before any content loads**
On first render the list is empty, the sentinel sits at the top, and the observer fires immediately. This causes a double-fetch: one from `loadNext()` on mount and one from the observer. Fix: use the dedicated `loadNext()` on mount for the first page and gate the observer callback with `posts.length > 0`, or simply rely on `isLoading` guard — the second fire will be a no-op.

**3. `rootMargin` with percentage values behaves unexpectedly**
`rootMargin` percentages are relative to the root element's bounding box, not the document. On a 375 px wide mobile viewport, `50%` means only ~188 px of pre-fetch distance. Use pixel values (`300px`) for consistent, device-independent early loading.

**4. Scroll jump when items are appended just before the sentinel**
If CSS `overflow-anchor` is not active (e.g., `overflow: hidden` on the container, or the sentinel has `overflow-anchor: auto` instead of `none`), the browser may anchor to the sentinel and jump. Fix: set `overflow-anchor: none` explicitly on the sentinel and `overflow-anchor: auto` on the container.

**5. Scroll position not restored after Back navigation**
Browsers reset `scrollTop` to 0 after navigation unless `history.scrollRestoration = 'manual'` is set. Even with that flag set, position must be saved to `history.state` before the user leaves the page. The save-on-scroll listener handles this, but there is a race condition: if the user navigates away before the debounce timer fires, the last position is not saved. Fix: also call `history.replaceState` synchronously in a `beforeunload` handler.

**6. AbortController not cleaned up on unmount**
If the component unmounts while a fetch is in-flight, the response handler runs against a destroyed component state and throws React state update warnings (or Angular zone errors). Fix: call `abortController.abort()` in the cleanup function of `useEffect` (React) or `ngOnDestroy` (Angular).

**7. `switchMap` in Angular cancels the in-flight request correctly but loses the page counter**
`switchMap` cancels the previous HTTP request if `load$` emits again before the first completes. The page counter `this._page` increments in the subscribe callback, which only runs on success. If the request is cancelled, `this._page` stays correct — but if you use `mergeMap` or `concatMap` instead of `switchMap`, duplicate page fetches can arrive out of order. `switchMap` is the right operator here.

**8. Prepend causing scroll jump in a live feed**
`overflow-anchor: auto` handles append but not prepend. When a WebSocket pushes new posts to the top, the `scrollHeight` grows upward and `scrollTop` stays the same — visually the user snaps up. Fix: use the `captureScrollAnchor` / `restoreScrollAnchor` utility. Capture `scrollHeight - scrollTop` before the DOM mutation, then after the mutation set `scrollTop = newScrollHeight - capturedDistance`.

---

## Interview Angle

**Q: "How does Intersection Observer differ from a scroll event listener for infinite scroll?"** A scroll event fires on every pixel of movement — up to 60+ times per second on smooth scroll. Even with debouncing or throttling, you are still polling the DOM on the main thread, calling `getBoundingClientRect()` which forces style recalculation. `IntersectionObserver` is event-driven: the browser's compositor thread checks element visibility against the root bounds asynchronously. Your callback is invoked only when the element crosses a threshold boundary — not on every scroll tick. It handles resize, CSS visibility changes, and layout shifts without any extra code. The API also supports `rootMargin`, letting you pre-fetch before the sentinel actually hits the viewport without any arithmetic on your side.

**Follow-up: "How do you prevent a scroll jump when prepending items to the top of a live feed?"** Two techniques, depending on browser support requirements. CSS `overflow-anchor: auto` on the container is the declarative approach: the browser automatically picks a DOM node visible in the viewport as the anchor and adjusts `scrollTop` to keep it in place after DOM mutations. This works for append at the bottom and for most prepend scenarios in modern browsers. When you need explicit control — for example, in a chat application where new messages push the history up — the manual approach is more reliable: before the DOM mutation, store `distance = container.scrollHeight - container.scrollTop`. After the mutation, immediately set `container.scrollTop = container.scrollHeight - distance`. This calculation compensates for the exact height added at the top, keeping the user's read position visually stable. Wrap this in `requestAnimationFrame` if the mutation happens inside a React state update to ensure it runs after the DOM is painted.

**Follow-up: "Walk me through scroll position restoration on Back navigation."** Browser default scroll restoration (`history.scrollRestoration = 'auto'`) resets scroll to the top after navigation in single-page apps because the router replaces the DOM without a real page reload. The fix has three parts: (1) set `history.scrollRestoration = 'manual'` so the browser stops resetting scroll; (2) listen to scroll events on the container with a debounced handler that calls `history.replaceState` with the current `scrollTop` merged into `history.state`; (3) on component mount, read `history.state` for the saved `scrollTop` and restore it inside a `requestAnimationFrame` callback so the items are in the DOM before the scroll is applied. The `requestAnimationFrame` timing is important — restoring inside `useEffect` without `rAF` can fire before the list renders, finding `scrollTop` of 0.
