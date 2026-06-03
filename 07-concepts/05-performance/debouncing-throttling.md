# Debouncing and Throttling

## Overview

Debouncing and throttling are rate-limiting techniques that control how often a function executes. They're essential for performance in the browser — without them, scroll handlers, resize listeners, and search inputs fire hundreds of times per second and overwhelm both the browser and your servers. This guide covers the difference between the two, implementations from scratch, and the real-world use cases where each belongs.

## The Core Difference

```
Without rate limiting (every call executes):
  ─────────────────────────────────────────
  user typing: a  b  c  d  e  f  g  h  i
  API calls:   ↓  ↓  ↓  ↓  ↓  ↓  ↓  ↓  ↓  (9 calls)

Debounce (waits for pause, fires once at end):
  ─────────────────────────────────────────
  user typing: a  b  c  d  e  f  g  h  i
               [─────── 300ms wait ──────]
  API calls:                              ↓  (1 call)

Throttle (fires at most once per interval):
  ─────────────────────────────────────────
  user scrolling: ░░░░░░░░░░░░░░░░░░░░░░░░░
  Handler fires:  ↓     ↓     ↓     ↓     ↓  (1 per 200ms)
```

**Debounce:** Waits for the function to stop being called for N milliseconds, then fires once. Good for "when the user stops doing X."

**Throttle:** Ensures the function fires at most once every N milliseconds regardless of how often it's called. Good for "do X but no more than once per interval."

## Debounce — Implementation from Scratch

```typescript
function debounce<T extends (...args: any[]) => any>(
  fn: T,
  delay: number
): (...args: Parameters<T>) => void {
  let timer: ReturnType<typeof setTimeout> | undefined;

  return function debounced(...args: Parameters<T>) {
    // Cancel any pending timer
    clearTimeout(timer);

    // Set a new timer — fn only runs if this timer isn't cancelled again
    timer = setTimeout(() => {
      fn(...args);
      timer = undefined;
    }, delay);
  };
}

// Usage
const handleSearch = debounce((query: string) => {
  fetchSearchResults(query); // Only called 300ms after user stops typing
}, 300);

searchInput.addEventListener('input', (e) => {
  handleSearch((e.target as HTMLInputElement).value);
});
```

### Debounce with Leading Edge (Immediate + Trailing)

```typescript
interface DebounceOptions {
  leading?: boolean;   // Fire immediately on first call (default: false)
  trailing?: boolean;  // Fire after delay on last call (default: true)
}

function debounce<T extends (...args: any[]) => any>(
  fn: T,
  delay: number,
  options: DebounceOptions = {}
): (...args: Parameters<T>) => void {
  const { leading = false, trailing = true } = options;
  let timer: ReturnType<typeof setTimeout> | undefined;
  let lastCallTime: number | undefined;

  return function debounced(...args: Parameters<T>) {
    const now = Date.now();
    const isFirstCall = lastCallTime === undefined;
    lastCallTime = now;

    if (leading && isFirstCall) {
      fn(...args); // Fire immediately on first invocation
    }

    clearTimeout(timer);

    if (trailing) {
      timer = setTimeout(() => {
        if (!leading || lastCallTime !== now) {
          fn(...args); // Fire after final call's delay
        }
        timer = undefined;
        lastCallTime = undefined;
      }, delay);
    }
  };
}

// Leading debounce: fires immediately on first call, ignores subsequent calls until quiet
const handleButtonClick = debounce(submitForm, 500, { leading: true, trailing: false });
// First click → submits immediately
// Rapid double-clicks → ignores second (prevents double submit)
```

### Debounce with Cancel and Flush

```typescript
interface DebouncedFunction<T extends (...args: any[]) => any> {
  (...args: Parameters<T>): void;
  cancel(): void;
  flush(...args: Parameters<T>): void;
}

function debounce<T extends (...args: any[]) => any>(
  fn: T,
  delay: number
): DebouncedFunction<T> {
  let timer: ReturnType<typeof setTimeout> | undefined;
  let pendingArgs: Parameters<T> | undefined;

  function debounced(...args: Parameters<T>): void {
    clearTimeout(timer);
    pendingArgs = args;
    timer = setTimeout(() => {
      fn(...args);
      timer = undefined;
      pendingArgs = undefined;
    }, delay);
  }

  // Cancel pending invocation
  debounced.cancel = function () {
    clearTimeout(timer);
    timer = undefined;
    pendingArgs = undefined;
  };

  // Execute immediately and cancel pending
  debounced.flush = function (...args: Parameters<T>) {
    clearTimeout(timer);
    timer = undefined;
    fn(...(args.length ? args : (pendingArgs ?? args)));
    pendingArgs = undefined;
  };

  return debounced;
}

// Component cleanup
const debouncedSave = debounce(saveToServer, 1000);

// On unmount: flush so unsaved changes aren't lost
useEffect(() => {
  return () => debouncedSave.flush(currentValue);
}, []);
```

## Throttle — Implementation from Scratch

```typescript
function throttle<T extends (...args: any[]) => any>(
  fn: T,
  interval: number
): (...args: Parameters<T>) => void {
  let lastCallTime = 0;

  return function throttled(...args: Parameters<T>) {
    const now = Date.now();
    const remaining = interval - (now - lastCallTime);

    if (remaining <= 0) {
      lastCallTime = now;
      fn(...args);
    }
    // If within interval: call is dropped (leading-edge throttle)
  };
}

// Scroll handler — fires at most once per 100ms
const handleScroll = throttle(() => {
  updateScrollPosition(window.scrollY);
}, 100);

window.addEventListener('scroll', handleScroll, { passive: true });
```

### Throttle with Trailing Call

A common requirement: also fire at the end of a burst (not just the leading edge):

```typescript
function throttle<T extends (...args: any[]) => any>(
  fn: T,
  interval: number
): (...args: Parameters<T>) => void {
  let lastCallTime = 0;
  let timer: ReturnType<typeof setTimeout> | undefined;
  let lastArgs: Parameters<T>;

  return function throttled(...args: Parameters<T>) {
    const now = Date.now();
    const remaining = interval - (now - lastCallTime);
    lastArgs = args;

    if (remaining <= 0) {
      clearTimeout(timer);
      timer = undefined;
      lastCallTime = now;
      fn(...args);
    } else if (!timer) {
      // Schedule trailing call for when the interval expires
      timer = setTimeout(() => {
        lastCallTime = Date.now();
        timer = undefined;
        fn(...lastArgs);
      }, remaining);
    }
    // Else: interval not yet elapsed and trailing already scheduled — update args only
  };
}
```

### Throttle with requestAnimationFrame

For visual updates, `requestAnimationFrame` is the best throttle — it fires at the display refresh rate (60fps = ~16ms):

```typescript
function rafThrottle<T extends (...args: any[]) => any>(
  fn: T
): (...args: Parameters<T>) => void {
  let rafId: number | undefined;

  return function rafThrottled(...args: Parameters<T>) {
    if (rafId !== undefined) return; // Already scheduled

    rafId = requestAnimationFrame(() => {
      fn(...args);
      rafId = undefined;
    });
  };
}

// Scroll position that drives animations — use rAF
const updateParallax = rafThrottle((scrollY: number) => {
  heroElement.style.transform = `translateY(${scrollY * 0.3}px)`;
});

window.addEventListener('scroll', () => updateParallax(window.scrollY), { passive: true });
```

## React Hooks

### useDebounce

```typescript
import { useState, useEffect } from 'react';

function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// Usage
function SearchInput() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);

  // This effect only fires 300ms after user stops typing
  useEffect(() => {
    if (debouncedQuery) {
      fetchResults(debouncedQuery);
    }
  }, [debouncedQuery]);

  return (
    <input
      value={query}
      onChange={(e) => setQuery(e.target.value)}
      placeholder="Search..."
    />
  );
}
```

### useDebouncedCallback

```typescript
import { useCallback, useRef } from 'react';

function useDebouncedCallback<T extends (...args: any[]) => any>(
  fn: T,
  delay: number
): T {
  const fnRef = useRef(fn);
  fnRef.current = fn; // Always call the latest fn

  const timerRef = useRef<ReturnType<typeof setTimeout>>();

  return useCallback(
    (...args: Parameters<T>) => {
      clearTimeout(timerRef.current);
      timerRef.current = setTimeout(() => {
        fnRef.current(...args);
      }, delay);
    },
    [delay]
  ) as T;
}

// Usage — doesn't recreate the function on every render
function CommentEditor({ postId }: { postId: string }) {
  const [content, setContent] = useState('');

  const autoSave = useDebouncedCallback(async (text: string) => {
    await api.saveDraft({ postId, content: text });
  }, 1000);

  return (
    <textarea
      value={content}
      onChange={(e) => {
        setContent(e.target.value);
        autoSave(e.target.value);
      }}
    />
  );
}
```

### useThrottledCallback

```typescript
function useThrottledCallback<T extends (...args: any[]) => any>(
  fn: T,
  interval: number
): T {
  const fnRef = useRef(fn);
  fnRef.current = fn;

  const lastCallRef = useRef(0);

  return useCallback(
    (...args: Parameters<T>) => {
      const now = Date.now();
      if (now - lastCallRef.current >= interval) {
        lastCallRef.current = now;
        fnRef.current(...args);
      }
    },
    [interval]
  ) as T;
}

function InfiniteScrollList() {
  const { loadMore, hasMore } = useInfiniteScroll();

  // Only check scroll position once per 200ms
  const handleScroll = useThrottledCallback(() => {
    const { scrollTop, scrollHeight, clientHeight } = document.documentElement;
    if (scrollHeight - scrollTop <= clientHeight + 200 && hasMore) {
      loadMore();
    }
  }, 200);

  useEffect(() => {
    window.addEventListener('scroll', handleScroll, { passive: true });
    return () => window.removeEventListener('scroll', handleScroll);
  }, [handleScroll]);
}
```

## Use Case Reference

```
When to use debounce vs throttle:

DEBOUNCE — "fire after N ms of silence":
  ✓ Search/autocomplete input (wait for typing pause, then search)
  ✓ Auto-save form fields (save after user stops editing)
  ✓ Window resize handler (calculate layout after resize ends)
  ✓ Validation on input (validate after user finishes field)
  ✓ API call deduplication (multiple rapid triggers → one call)

THROTTLE — "fire at most once per N ms":
  ✓ Scroll position tracking (sticky header, parallax, virtual list)
  ✓ Mouse move / drag handlers
  ✓ Game loop / animation frame rate limiting
  ✓ "Load more" trigger on scroll (prevent multiple loads)
  ✓ WebSocket send rate limiting (live collaboration cursors)
  ✓ Resize observer callbacks (continuous layout updates)

rAF THROTTLE — "fire at display refresh rate":
  ✓ Any visual update driven by user interaction
  ✓ Scroll-driven animations
  ✓ Canvas/WebGL rendering loops
```

## Lodash vs Custom vs Native

```typescript
// Lodash — battle-tested, more features (cancel, flush, maxWait)
import { debounce, throttle } from 'lodash-es'; // tree-shakeable

const debouncedSearch = debounce(search, 300, { leading: false, trailing: true });
const throttledScroll = throttle(updateLayout, 100, { leading: true, trailing: true });

// Custom — zero dependencies, exact API control, smaller (if not already using lodash)
const debouncedSearch = debounce(search, 300); // from this file's implementation

// Specialized libraries
// use-debounce — React hooks, TypeScript, SSR-safe
import { useDebounce, useDebouncedCallback } from 'use-debounce';
```

### Lodash maxWait Option

```typescript
// maxWait ensures the function fires at least every N ms even if user never pauses
const saveProgress = debounce(
  (content: string) => api.autosave(content),
  2000,
  { maxWait: 10_000 } // Force save after 10s even if user types continuously
);
// Prevents "user typed for 30 minutes without a 2s pause — nothing saved"
```

## Common Mistakes

### 1. Creating a New Debounced Function on Every Render

```typescript
// ❌ New debounced function on every render — debounce timer resets every render!
function SearchBox() {
  const handleInput = debounce((query: string) => {
    search(query);
  }, 300);
  // New function instance = new timer = debounce never fires

  return <input onChange={(e) => handleInput(e.target.value)} />;
}

// ✅ Stable reference with useCallback or useDebouncedCallback
function SearchBox() {
  const handleInput = useDebouncedCallback((query: string) => {
    search(query);
  }, 300);

  return <input onChange={(e) => handleInput(e.target.value)} />;
}
```

### 2. Using Debounce for Scroll Handlers

```typescript
// ❌ Debounce for scroll — fires AFTER scrolling stops, not during
window.addEventListener('scroll', debounce(updateStickyHeader, 100));
// Sticky header doesn't update while scrolling — terrible UX

// ✅ Throttle or rAF for scroll
window.addEventListener('scroll', throttle(updateStickyHeader, 16), { passive: true });
// OR
window.addEventListener('scroll', rafThrottle(updateStickyHeader), { passive: true });
```

### 3. Not Cleaning Up on Unmount

```typescript
// ❌ Memory leak — timer fires after component unmounts
useEffect(() => {
  window.addEventListener('resize', debouncedResize);
}, []);

// ✅ Remove listener AND cancel pending debounce on unmount
useEffect(() => {
  window.addEventListener('resize', debouncedResize);
  return () => {
    window.removeEventListener('resize', debouncedResize);
    debouncedResize.cancel(); // Cancel any pending timer
  };
}, [debouncedResize]);
```

### 4. Debouncing Already-Debounced Values

```typescript
// ❌ Double debounce — actual delay is delay1 + delay2
const debouncedQuery = useDebounce(query, 300);

const handleSearch = useDebouncedCallback((q: string) => {
  search(q);
}, 300);

useEffect(() => {
  handleSearch(debouncedQuery); // 600ms total delay!
}, [debouncedQuery]);

// ✅ Debounce once, at the source
const debouncedQuery = useDebounce(query, 300);
useEffect(() => {
  if (debouncedQuery) search(debouncedQuery);
}, [debouncedQuery]);
```

## Interview Questions

### 1. What is the difference between debounce and throttle?

**Answer:** Debounce delays execution until the function hasn't been called for N milliseconds — it fires once after a burst of calls ends. Throttle ensures the function fires at most once per N milliseconds regardless of call frequency — it fires at a consistent rate during a burst. Mental model: debounce is "fire when things settle down," throttle is "fire regularly but not too fast." Search inputs use debounce (wait for typing to pause, then search). Scroll handlers use throttle (update layout at a fixed rate while scrolling).

### 2. Implement debounce from scratch without using any library.

**Answer:**
```typescript
function debounce<T extends (...args: any[]) => any>(fn: T, delay: number) {
  let timer: ReturnType<typeof setTimeout> | undefined;
  return function debounced(...args: Parameters<T>) {
    clearTimeout(timer);
    timer = setTimeout(() => { fn(...args); timer = undefined; }, delay);
  };
}
```
Key points: (1) close over a single `timer` variable so all calls share state, (2) `clearTimeout` before setting a new timer — this is what resets the countdown, (3) call the original function with the latest arguments after the delay.

### 3. When would you choose requestAnimationFrame over throttle for rate limiting?

**Answer:** Use `requestAnimationFrame` when the work you're rate-limiting is a visual update tied to rendering. `rAF` fires exactly once before each browser paint (up to 60fps on 60Hz displays, 120fps on high-refresh screens) — it's synchronized with the browser's rendering cycle. `setTimeout`-based throttle fires on a JavaScript timer which can be misaligned with paint cycles, causing jank. Use rAF for scroll-driven animations, canvas drawing, and any style/transform updates. Use setTimeout-based throttle for non-visual work like logging, analytics, or API calls where perfect frame alignment doesn't matter.

### 4. Why does creating a new debounced function inside a React component on every render break it?

**Answer:** Each call to `debounce(fn, delay)` creates a new closure with a fresh, independent timer. If a component re-renders before the debounce fires, the old debounced function (with its timer) is discarded and replaced with a brand-new one — the timer is reset on every render. For a component that re-renders on every keystroke, the debounce can never fire because its timer is always reset before it expires. The fix is to give the debounced function a stable reference using `useCallback`, `useMemo`, `useRef`, or a dedicated hook like `useDebouncedCallback`.

### 5. How do you handle cleanup for debounced functions in React?

**Answer:** Two things need cleanup: (1) the event listener, removed with `removeEventListener` in the `useEffect` cleanup function; (2) the pending timer in the debounced function, cancelled by calling `.cancel()`. Without cancelling pending timers, you risk a state update on an unmounted component — the timer fires, calls `setState`, and React warns about memory leaks. Use `lodash.debounce`'s built-in `.cancel()` method, or implement it yourself with `clearTimeout`. If the debounce is for auto-save, consider calling `.flush()` on unmount instead so pending changes are saved before the component disappears.
