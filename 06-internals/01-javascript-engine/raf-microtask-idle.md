# requestAnimationFrame, requestIdleCallback, and queueMicrotask

## Table of Contents
- [Overview](#overview)
- [The Event Loop Refresher](#the-event-loop-refresher)
- [queueMicrotask](#queuemicrotask)
- [requestAnimationFrame](#requestanimationframe)
- [requestIdleCallback](#requestidlecallback)
- [Ordering and Interaction](#ordering-and-interaction)
- [Practical Use Cases](#practical-use-cases)
- [Common Pitfalls](#common-pitfalls)
- [Performance Implications](#performance-implications)
- [Interview Questions](#interview-questions)
- [Key Takeaways](#key-takeaways)
- [Resources](#resources)

## Overview

The browser event loop exposes three distinct scheduling primitives beyond `setTimeout`/`setInterval`:

| API | Queue | Timing | Primary Use |
|-----|-------|--------|-------------|
| `queueMicrotask` | Microtask queue | After current task, before next task | Deferred-but-immediate async work |
| `requestAnimationFrame` | Animation frame callbacks | Before next paint | Visual updates |
| `requestIdleCallback` | Idle callbacks | After paint, during idle periods | Low-priority background work |

Understanding precisely *where* each fits inside a single event-loop tick determines whether visual updates stutter, whether state reads are consistent, and whether background tasks starve the UI.

## The Event Loop Refresher

### Full Tick Anatomy (HTML Living Standard)

```
┌─────────────────────────────────────────────────────────────────┐
│  One Event-Loop Iteration                                        │
│                                                                  │
│  1. Pick ONE task from the task queue (macrotask)               │
│     e.g., setTimeout callback, I/O event, script evaluation     │
│                                                                  │
│  2. Drain microtask queue (ALL pending microtasks)              │
│     Promises (.then/.catch), queueMicrotask, MutationObserver   │
│     NOTE: New microtasks added here are also drained            │
│                                                                  │
│  3. [Rendering opportunity — browser decides if needed]         │
│     a. Run ResizeObserver callbacks                             │
│     b. Run IntersectionObserver callbacks                       │
│     c. Run requestAnimationFrame callbacks                      │
│     d. Run CSS animation callbacks                              │
│     e. Perform style recalc + layout + paint + composite        │
│                                                                  │
│  4. [Idle period — if frame budget not exhausted]               │
│     Run requestIdleCallback callbacks (time-sliced)             │
│                                                                  │
│  5. Go to step 1                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```typescript
// Demonstrating the ordering
console.log('1: sync');

setTimeout(() => console.log('5: setTimeout'), 0);

Promise.resolve().then(() => console.log('3: promise microtask'));

queueMicrotask(() => console.log('4: queueMicrotask'));

requestAnimationFrame(() => console.log('? rAF — before next paint'));

console.log('2: sync end');

// Output order (guaranteed):
// 1: sync
// 2: sync end
// 3: promise microtask
// 4: queueMicrotask
// 5: setTimeout          (next task)
// ? rAF                  (before next paint — may be after setTimeout)
```

## queueMicrotask

### Internal Mechanics

`queueMicrotask(fn)` pushes `fn` directly onto the microtask checkpoint queue — the same queue that `.then()` callbacks target. The key difference from `Promise.resolve().then(fn)` is purely surface-level; there is no promise allocation or GC overhead.

```
Current Task executing
        │
        ▼
  queueMicrotask(fn)  ──► Microtask Queue: [...existing, fn]
        │
        ▼
  Task finishes
        │
        ▼
  Microtask Checkpoint: drain ALL microtasks (including fn)
        │
        ▼
  Next task / rendering
```

### Reentrancy: Microtasks Can Schedule Microtasks

```typescript
// WARNING: This loop never yields to the browser
function infinite() {
  queueMicrotask(() => {
    // This schedules another microtask before the checkpoint clears
    queueMicrotask(infinite);
  });
}
// infinite() will starve the event loop — no rendering ever happens

// SAFE pattern: Use a counter guard
let depth = 0;
const MAX_DEPTH = 100;

function safeRecursiveMicrotask(depth: number) {
  if (depth >= MAX_DEPTH) {
    setTimeout(() => safeRecursiveMicrotask(0), 0); // Yield to macrotask
    return;
  }
  queueMicrotask(() => safeRecursiveMicrotask(depth + 1));
}
```

### vs Promise.resolve().then()

```typescript
// Functionally equivalent but queueMicrotask is:
// - Slightly faster (no Promise object creation)
// - Cleaner intent signaling
// - Better for non-async code that just needs deferred execution

// Identical scheduling:
queueMicrotask(() => doWork());
Promise.resolve().then(() => doWork());

// queueMicrotask does NOT catch errors automatically
queueMicrotask(() => {
  throw new Error('uncaught'); // Goes to window.onerror / unhandledRejection
});

// Promise swallows and routes through rejection
Promise.resolve().then(() => {
  throw new Error('caught as unhandledRejection');
});
```

### Practical Use: Batching DOM Reads After Sync Mutations

```typescript
// Pattern: read DOM once after all synchronous writes settle
class DOMBatcher {
  private pendingReads: Array<() => void> = [];
  private scheduled = false;

  scheduleRead(fn: () => void): void {
    this.pendingReads.push(fn);
    if (!this.scheduled) {
      this.scheduled = true;
      queueMicrotask(() => this.flush());
    }
  }

  private flush(): void {
    const reads = this.pendingReads.splice(0);
    this.scheduled = false;
    // All reads happen in one microtask — single layout pass
    for (const fn of reads) fn();
  }
}

const batcher = new DOMBatcher();

// These fire synchronously in the same task:
element.style.height = '100px';
element.style.width = '200px';
// One microtask later — layout has been updated, reads are consistent
batcher.scheduleRead(() => console.log(element.offsetWidth));
```

## requestAnimationFrame

### Where rAF Lives in the Pipeline

`requestAnimationFrame` callbacks are called by the browser just before it performs a rendering update. This is tightly coupled to the display's refresh rate (typically 60 Hz = 16.67 ms per frame, or 120 Hz = 8.33 ms).

```
Timeline at 60 fps (16.67 ms budget):
  ├─── Task(s) + Microtasks ──────────┤ rAF ├── Style/Layout/Paint ─┤
  0ms                               14ms   15ms                    16.67ms
```

```typescript
// rAF callback receives a DOMHighResTimeStamp
// This timestamp is the same for all rAF callbacks in the same frame
requestAnimationFrame((timestamp: DOMHighResTimeStamp) => {
  console.log(timestamp); // e.g., 1234567.89 ms since page load
  // All rAF callbacks in this frame share the same timestamp
  // Use it for animation interpolation, NOT Date.now()
});
```

### Animation Loop Pattern

```typescript
interface AnimationState {
  x: number;
  velocity: number;
}

function createAnimationLoop(
  element: HTMLElement,
  target: number
): () => void {
  const state: AnimationState = { x: 0, velocity: 0 };
  let rafId: number;
  let lastTimestamp: DOMHighResTimeStamp | null = null;

  function frame(timestamp: DOMHighResTimeStamp): void {
    const dt = lastTimestamp === null ? 0 : timestamp - lastTimestamp;
    lastTimestamp = timestamp;

    // Physics-based spring animation
    const stiffness = 0.1;
    const damping = 0.8;
    const force = (target - state.x) * stiffness;
    state.velocity = state.velocity * damping + force;
    state.x += state.velocity * (dt / 16.67); // normalize to 60fps

    // Write to DOM — ONLY in rAF, never in event handlers
    element.style.transform = `translateX(${state.x}px)`;

    // Only keep looping if not settled
    if (Math.abs(state.velocity) > 0.01 || Math.abs(target - state.x) > 0.1) {
      rafId = requestAnimationFrame(frame);
    }
  }

  rafId = requestAnimationFrame(frame);

  // Return cancel function
  return () => cancelAnimationFrame(rafId);
}
```

### Read-Write Separation (Avoiding Layout Thrashing)

```typescript
// BAD: Alternating reads and writes force multiple layouts per frame
function badUpdate(elements: HTMLElement[]): void {
  requestAnimationFrame(() => {
    for (const el of elements) {
      const height = el.offsetHeight; // READ — forces layout
      el.style.height = `${height * 2}px`; // WRITE — invalidates layout
      // Next iteration's read forces ANOTHER layout
    }
  });
}

// GOOD: Batch reads first, then batch writes
function goodUpdate(elements: HTMLElement[]): void {
  requestAnimationFrame(() => {
    // Phase 1: Read all values (one layout computation)
    const heights = elements.map(el => el.offsetHeight);

    // Phase 2: Write all values (layout invalidated once)
    elements.forEach((el, i) => {
      el.style.height = `${heights[i] * 2}px`;
    });
  });
}
```

### Double rAF for Post-Paint Work

```typescript
// Single rAF fires BEFORE paint — element not yet visible
requestAnimationFrame(() => {
  element.classList.add('visible'); // Applied before paint ✓
});

// Double rAF: fires AFTER first paint — useful for transitions
// that need element to be in initial state before animating
requestAnimationFrame(() => {
  element.style.opacity = '0'; // Set initial state
  requestAnimationFrame(() => {
    // Now browser has painted the opacity:0 state
    element.style.transition = 'opacity 0.3s';
    element.style.opacity = '1'; // Animate from 0 → 1
  });
});
```

### rAF Throttling on Hidden Tabs

```typescript
// Browser throttles rAF to ~1fps on hidden tabs
// Use Page Visibility API to pause animations
let rafId: number;

function startAnimation(): void {
  function loop(ts: DOMHighResTimeStamp): void {
    // ... animation logic ...
    rafId = requestAnimationFrame(loop);
  }
  rafId = requestAnimationFrame(loop);
}

document.addEventListener('visibilitychange', () => {
  if (document.hidden) {
    cancelAnimationFrame(rafId); // Fully stop on hidden
  } else {
    startAnimation(); // Resume on visible
  }
});
```

## requestIdleCallback

### The Idle Period

After the browser has painted a frame and processed all urgent work, any remaining time before the next frame is "idle time." `requestIdleCallback` gives access to this spare capacity.

```
Frame budget: 16.67ms (60fps)
  ├── Tasks + Microtasks (5ms) ──┤ rAF (2ms) ├── Paint (3ms) ─┤ IDLE (6.67ms) ─┤
  0ms                           7ms          9ms             12ms             16.67ms

requestIdleCallback gets up to 6.67ms of idle time in this frame
If no idle time exists (long tasks), callback may be delayed many frames
```

```typescript
interface IdleDeadline {
  readonly didTimeout: boolean;
  timeRemaining(): DOMHighResTimeStamp; // Milliseconds left in idle period
}

requestIdleCallback((deadline: IdleDeadline) => {
  // Keep processing while time remains and work exists
  while (!workQueue.isEmpty() && deadline.timeRemaining() > 0) {
    const task = workQueue.dequeue();
    task.execute();
  }

  // If work remains, schedule another idle callback
  if (!workQueue.isEmpty()) {
    requestIdleCallback(/* this callback */);
  }
}, { timeout: 2000 }); // Force run after 2s even if not idle
```

### The timeout Option

```typescript
// Without timeout: may NEVER run during heavy animation/scroll
requestIdleCallback(lowPrioritySend); // Risky in production

// With timeout: guaranteed to run within N ms (at cost of possible jank)
requestIdleCallback(lowPrioritySend, { timeout: 5000 });

// When timeout fires, deadline.didTimeout === true
// The callback runs in a regular task, not idle — budget may be 0
requestIdleCallback((deadline) => {
  if (deadline.didTimeout) {
    // Running because of timeout, not idle — be careful
    // deadline.timeRemaining() may be 0 or very small
    // Only do critical work here
    sendCriticalAnalytics();
  } else {
    // True idle time — full budget available
    sendFullAnalyticsBatch();
  }
}, { timeout: 10000 });
```

### Background Work Patterns

```typescript
// Pattern 1: Precompute / warm caches
class RoutePreloader {
  private queue: string[] = [];
  private idleCallbackId: IdleCallbackID | null = null;

  preload(routes: string[]): void {
    this.queue.push(...routes);
    if (this.idleCallbackId === null) {
      this.scheduleWork();
    }
  }

  private scheduleWork(): void {
    this.idleCallbackId = requestIdleCallback(
      (deadline) => this.processQueue(deadline),
      { timeout: 5000 }
    );
  }

  private processQueue(deadline: IdleDeadline): void {
    this.idleCallbackId = null;
    while (this.queue.length > 0 && deadline.timeRemaining() > 1) {
      const route = this.queue.shift()!;
      this.prefetchRoute(route);
    }
    if (this.queue.length > 0) {
      this.scheduleWork();
    }
  }

  private prefetchRoute(route: string): void {
    const link = document.createElement('link');
    link.rel = 'prefetch';
    link.href = route;
    document.head.appendChild(link);
  }
}

// Pattern 2: Analytics batching
class AnalyticsBatcher {
  private events: object[] = [];
  private idleId: IdleCallbackID | null = null;

  track(event: object): void {
    this.events.push(event);
    if (this.idleId === null) {
      this.idleId = requestIdleCallback(
        () => this.flush(),
        { timeout: 30000 } // Force flush after 30s max
      );
    }
  }

  private flush(): void {
    this.idleId = null;
    if (this.events.length === 0) return;
    const batch = this.events.splice(0);
    navigator.sendBeacon('/api/analytics', JSON.stringify(batch));
  }
}
```

### Polyfill for Safari (No rIC Support)

```typescript
// Safari does not support requestIdleCallback (as of 2024)
// Polyfill using setTimeout with 0ms delay
const requestIdleCallbackPolyfill =
  window.requestIdleCallback ??
  ((cb: IdleRequestCallback, opts?: IdleRequestOptions) => {
    const start = Date.now();
    return setTimeout(() => {
      cb({
        didTimeout: false,
        timeRemaining: () => Math.max(0, 50 - (Date.now() - start)),
      });
    }, 1) as unknown as number;
  });

const cancelIdleCallbackPolyfill =
  window.cancelIdleCallback ?? clearTimeout;
```

## Ordering and Interaction

### Precise Sequence Demonstration

```typescript
// This example shows EXACT ordering
let log: string[] = [];

// Schedule everything from a synchronous script:
setTimeout(() => log.push('A: setTimeout'), 0);

requestAnimationFrame(() => {
  log.push('C: rAF');
  // Microtasks in rAF drain AFTER all rAF callbacks
  queueMicrotask(() => log.push('D: microtask inside rAF'));
});

queueMicrotask(() => log.push('B: microtask'));

requestIdleCallback(() => log.push('E: rIC'));

// Observed order:
// B: microtask          (microtask checkpoint after script)
// A: setTimeout         (next macrotask)
// C: rAF                (before next paint)
// D: microtask inside rAF (microtask checkpoint after rAF callbacks)
// E: rIC                (after paint, during idle)

// Note: A vs C order depends on browser; C is NOT guaranteed
// to come after A — rAF fires before paint regardless of tasks
```

### Common Misconception: rAF is NOT a Macrotask

```
❌ Incorrect mental model:
   MacroTask Queue: [setTimeout, setInterval, I/O, ..., rAF]

✓ Correct model:
   MacroTask Queue: [setTimeout, setInterval, I/O, ...]
   Rendering Steps (separate): [ResizeObserver, rAF, Paint]
   Idle Steps (separate):      [requestIdleCallback]

rAF is part of the "update the rendering" steps, which are
interleaved with the task queue but are NOT tasks themselves.
```

### Microtask Checkpoint Within rAF

```typescript
requestAnimationFrame(() => {
  // All rAF callbacks for this frame run first
  Promise.resolve().then(() => {
    // This microtask runs AFTER all rAF callbacks for this frame
    // but BEFORE the browser does style/layout/paint
    // Useful for final DOM mutations before paint
    element.style.color = 'red'; // Will be included in this paint
  });
});
```

## Practical Use Cases

### Use queueMicrotask When

```typescript
// 1. Signal-style reactive updates (batch multiple sync changes)
class Signal<T> {
  private _value: T;
  private subscribers = new Set<(v: T) => void>();
  private notifyScheduled = false;

  constructor(initial: T) {
    this._value = initial;
  }

  set value(v: T) {
    this._value = v;
    if (!this.notifyScheduled) {
      this.notifyScheduled = true;
      queueMicrotask(() => {
        this.notifyScheduled = false;
        this.subscribers.forEach(fn => fn(this._value));
      });
    }
  }

  get value(): T {
    return this._value;
  }
}

// 2. Converting sync APIs to async without extra task delay
function asyncify<T>(fn: () => T): Promise<T> {
  return new Promise((resolve) => {
    queueMicrotask(() => resolve(fn()));
  });
}

// 3. Post-mutation consistency checks
function updateWithCheck(el: HTMLElement, value: string): void {
  el.textContent = value;
  queueMicrotask(() => {
    // Verify the update took (e.g., sanitization may have changed it)
    console.assert(el.textContent === value);
  });
}
```

### Use requestAnimationFrame When

```typescript
// 1. CSS property animations not handled by CSS transitions
function smoothScroll(target: number, duration: number): void {
  const start = window.scrollY;
  const startTime = performance.now();

  requestAnimationFrame(function step(currentTime: DOMHighResTimeStamp) {
    const elapsed = currentTime - startTime;
    const progress = Math.min(elapsed / duration, 1);
    // Ease-in-out cubic
    const eased =
      progress < 0.5
        ? 4 * progress ** 3
        : 1 - (-2 * progress + 2) ** 3 / 2;

    window.scrollTo(0, start + (target - start) * eased);

    if (progress < 1) {
      requestAnimationFrame(step);
    }
  });
}

// 2. Canvas / WebGL rendering loop
function renderLoop(ctx: CanvasRenderingContext2D): void {
  function draw(timestamp: DOMHighResTimeStamp): void {
    ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);
    drawScene(ctx, timestamp);
    requestAnimationFrame(draw);
  }
  requestAnimationFrame(draw);
}

// 3. Synchronized DOM reads before paint (read layout values
//    at the precise moment after last write, before next paint)
function measureBeforeNextPaint(callback: (rect: DOMRect) => void): void {
  requestAnimationFrame(() => {
    callback(element.getBoundingClientRect()); // No forced reflow risk here
  });
}
```

### Use requestIdleCallback When

```typescript
// 1. Non-critical data persistence
function saveUserPreferences(prefs: object): void {
  requestIdleCallback(
    () => localStorage.setItem('prefs', JSON.stringify(prefs)),
    { timeout: 10000 }
  );
}

// 2. Pre-warming expensive computations
function prewarmParser(schemas: string[]): void {
  let index = 0;
  function processNext(deadline: IdleDeadline): void {
    while (index < schemas.length && deadline.timeRemaining() > 2) {
      JSONSchema.compile(schemas[index++]); // Cache compiled schema
    }
    if (index < schemas.length) {
      requestIdleCallback(processNext, { timeout: 3000 });
    }
  }
  requestIdleCallback(processNext);
}

// 3. Lazy loading non-critical resources
function lazyLoadImages(urls: string[]): void {
  requestIdleCallback((deadline) => {
    for (const url of urls) {
      if (deadline.timeRemaining() < 1) break;
      const img = new Image();
      img.src = url; // Starts fetching without blocking rendering
    }
  });
}
```

## Common Pitfalls

### Pitfall 1: Mutating DOM in requestIdleCallback

```typescript
// BAD: DOM mutations in rIC can cause layout thrashing
// rIC fires AFTER paint — mutations trigger an additional paint
requestIdleCallback(() => {
  element.style.color = 'blue'; // Forces unexpected repaint!
});

// GOOD: Stage changes in rIC, apply in rAF
let pendingMutation: (() => void) | null = null;

requestIdleCallback(() => {
  // Compute what needs changing
  pendingMutation = () => { element.style.color = 'blue'; };
});

requestAnimationFrame(() => {
  // Apply at the right time
  if (pendingMutation) {
    pendingMutation();
    pendingMutation = null;
  }
});
```

### Pitfall 2: Assuming rAF Fires at Exactly 60fps

```typescript
// BAD: Assuming 16.67ms delta
requestAnimationFrame(() => {
  position += speed * 16.67; // Wrong! Delta varies
  // On 120Hz displays: ~8.33ms
  // Under load: may be 33ms, 50ms, etc.
});

// GOOD: Use actual timestamp delta
let prevTimestamp: DOMHighResTimeStamp | null = null;

requestAnimationFrame(function loop(timestamp) {
  const delta = prevTimestamp === null ? 0 : timestamp - prevTimestamp;
  prevTimestamp = timestamp;

  // Cap delta to prevent huge jumps after tab switch
  const cappedDelta = Math.min(delta, 100);
  position += speed * (cappedDelta / 16.67); // Normalize to 60fps units

  requestAnimationFrame(loop);
});
```

### Pitfall 3: Memory Leaks from Uncancelled rAF

```typescript
class Component {
  private rafId: number | null = null;

  startAnimation(): void {
    const loop = (ts: DOMHighResTimeStamp) => {
      this.render(ts);
      this.rafId = requestAnimationFrame(loop);
    };
    this.rafId = requestAnimationFrame(loop);
  }

  // BAD: If component is destroyed without stopping, loop continues
  // and holds a closure reference to `this` preventing GC

  destroy(): void {
    // GOOD: Always cancel rAF on cleanup
    if (this.rafId !== null) {
      cancelAnimationFrame(this.rafId);
      this.rafId = null;
    }
  }
}

// React hook equivalent
function useAnimationLoop(callback: (ts: DOMHighResTimeStamp) => void): void {
  const cbRef = useRef(callback);
  cbRef.current = callback;

  useEffect(() => {
    let rafId: number;
    const loop = (ts: DOMHighResTimeStamp) => {
      cbRef.current(ts);
      rafId = requestAnimationFrame(loop);
    };
    rafId = requestAnimationFrame(loop);
    return () => cancelAnimationFrame(rafId); // Cleanup!
  }, []); // No deps — cbRef handles latest callback
}
```

### Pitfall 4: Blocking the Microtask Queue

```typescript
// BAD: Long microtask chain blocks all rendering
async function processLargeDataset(data: number[]): Promise<void> {
  for (const item of data) {
    await Promise.resolve(); // Each await is a microtask
    // 100,000 items = 100,000 microtasks before any paint!
    processItem(item);
  }
}

// GOOD: Yield to browser via macrotask every N items
async function processLargeDatasetSafe(data: number[]): Promise<void> {
  for (let i = 0; i < data.length; i++) {
    processItem(data[i]);
    // Yield to browser every 100 items
    if (i % 100 === 0) {
      await new Promise(resolve => setTimeout(resolve, 0));
    }
  }
}

// BETTER: Use requestIdleCallback for truly background work
function processLargeDatasetIdle(
  data: number[],
  onComplete: () => void
): void {
  let index = 0;
  function process(deadline: IdleDeadline): void {
    while (index < data.length && deadline.timeRemaining() > 1) {
      processItem(data[index++]);
    }
    if (index < data.length) {
      requestIdleCallback(process);
    } else {
      onComplete();
    }
  }
  requestIdleCallback(process);
}
```

## Performance Implications

### Choosing the Right Primitive

```
Decision tree for scheduling deferred work:

Is it a visual update (animation, DOM mutation visible to user)?
  YES → requestAnimationFrame
  NO  ↓

Does it need to happen before the next event/task sees updated state?
  YES → queueMicrotask (or Promise.resolve().then())
  NO  ↓

Does it need to happen within a bounded time window?
  YES → setTimeout(fn, maxDelay) or requestIdleCallback with timeout
  NO  ↓

Is it truly low-priority background work?
  YES → requestIdleCallback (no timeout)
  NO  → setTimeout(fn, 0) for simple task deferral
```

### Measurement and Profiling

```typescript
// Measure rAF callback duration to detect jank
const frameTimings: number[] = [];

requestAnimationFrame(function measure(ts) {
  const start = performance.now();

  // ... your frame work ...

  const duration = performance.now() - start;
  frameTimings.push(duration);

  if (duration > 10) {
    // Eating more than 10ms of a 16.67ms budget
    console.warn(`Slow rAF: ${duration.toFixed(2)}ms`);
  }

  requestAnimationFrame(measure);
});

// Use PerformanceObserver for Long Tasks (> 50ms total)
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.warn('Long task detected:', entry.duration, 'ms');
  }
});
observer.observe({ type: 'longtask', buffered: true });
```

---

## Supplementary: Scheduler API (`scheduler.postTask` / `scheduler.yield`)

The Scheduler API is the platform-level API for prioritized task scheduling — a more explicit alternative to `setTimeout(fn, 0)` and `requestIdleCallback`.

### `scheduler.postTask()`

```javascript
// Three priority levels — maps to browser task queues:
// 'user-blocking': same queue as user input (highest)
// 'user-visible':  default — visible rendering/animation work
// 'background':    idle priority — runs when browser is free

// Defer non-critical analytics without blocking anything
scheduler.postTask(() => sendAnalytics(event), {
  priority: 'background',
});

// Ensure a critical state update runs before the next paint
await scheduler.postTask(() => updateCriticalUI(), {
  priority: 'user-blocking',
});

// Cancel a task
const controller = new TaskController({ priority: 'background' });
const promise = scheduler.postTask(() => heavyWork(), {
  signal: controller.signal,
});
controller.abort(); // no-op if already started, cancels if queued

// Change priority before it runs
controller.setPriority('user-visible'); // promote
```

### `scheduler.yield()` — Breaking Long Tasks

`scheduler.yield()` is the key missing primitive for chunking work. It yields control back to the browser (allowing repaints and user interactions) and resumes at the same priority:

```javascript
// Before: 500ms Long Task, blocks all input
async function processLargeDataset(items) {
  for (const item of items) {
    process(item); // 1ms each × 500 items = 500ms block
  }
}

// After: yield every 50 items — browser stays responsive
async function processLargeDataset(items) {
  for (let i = 0; i < items.length; i++) {
    process(items[i]);
    if (i % 50 === 0) {
      await scheduler.yield(); // surrender to browser, resume next frame
    }
  }
}

// vs. setTimeout(fn, 0): yields but may take 4ms+ per yield (clamped)
// scheduler.yield(): yields at 'user-visible' priority — tighter timing
```

### Position in the Event Loop

```
Task Queue (user-blocking postTask tasks)
  ↓
Microtask Queue (drained after each task)
  ↓
RAF callbacks (before paint)
  ↓
Paint
  ↓
Task Queue (user-visible postTask tasks)
  ↓
Task Queue (background / idle postTask tasks)
  ↓
requestIdleCallback
```

### Browser Support
Chrome 94+ (`postTask`), Chrome 115+ (`yield`). Not yet in Firefox/Safari — polyfill with `requestIdleCallback` / `setTimeout` for background priority.

---

## Interview Questions

### Question 1: Where exactly does requestAnimationFrame fit in the event loop?

**Answer**: `requestAnimationFrame` callbacks are part of the browser's "update the rendering" steps, which are separate from the task queue. The browser runs rAF callbacks just before performing style recalculation, layout, and paint. Unlike `setTimeout`, rAF is not a macrotask — it fires on a rendering opportunity, which the browser schedules based on display refresh rate (usually 60 Hz). Microtasks queued inside an rAF callback are drained after all rAF callbacks for that frame complete, but before the actual paint.

### Question 2: What is the difference between queueMicrotask and Promise.resolve().then()?

**Answer**: Functionally they are identical — both schedule a callback in the microtask queue to run after the current task and any preceding microtasks. The differences are:
- `queueMicrotask` has no promise allocation overhead — slightly more performant
- `queueMicrotask` propagates errors as uncaught exceptions to `window.onerror`; promise rejections route through `unhandledrejection`
- `queueMicrotask` more clearly communicates intent: "schedule this microtask" rather than "resolve a void promise"
Both are equally dangerous if they schedule more microtasks recursively, which can starve rendering.

### Question 3: Why should you avoid DOM mutations inside requestIdleCallback?

**Answer**: `requestIdleCallback` fires AFTER the browser has already painted the current frame. Any DOM mutation made inside rIC invalidates layout/style and forces the browser to schedule an additional rendering pass that wasn't planned. This can cause extra repaints and actually reduce performance. The correct pattern is to compute changes during idle time and stage them for application in the next `requestAnimationFrame` callback.

### Question 4: How does requestIdleCallback behave when the page is under heavy load?

**Answer**: When the browser has no idle time between frames (e.g., during heavy animation or scrolling), rIC callbacks may be indefinitely postponed. Without a `timeout` option, the callback may never run during animation-heavy periods. The `timeout` option provides a deadline — the callback will be forced to run as a regular task after the specified milliseconds, even if the browser is busy. When forced this way, `deadline.didTimeout` is `true` and `deadline.timeRemaining()` returns 0.

### Question 5: How would you implement a cooperative task scheduler in JavaScript?

**Answer**: Use `requestIdleCallback` for time-sliced work with a work queue:

```typescript
class CooperativeScheduler {
  private queue: Array<{ task: () => void; priority: number }> = [];
  private scheduled = false;

  schedule(task: () => void, priority = 0): void {
    this.queue.push({ task, priority });
    this.queue.sort((a, b) => b.priority - a.priority);
    if (!this.scheduled) {
      this.scheduled = true;
      requestIdleCallback(
        (deadline) => this.run(deadline),
        { timeout: 5000 }
      );
    }
  }

  private run(deadline: IdleDeadline): void {
    this.scheduled = false;
    while (this.queue.length > 0 && deadline.timeRemaining() > 1) {
      this.queue.shift()!.task();
    }
    if (this.queue.length > 0) {
      this.scheduled = true;
      requestIdleCallback(
        (deadline) => this.run(deadline),
        { timeout: 5000 }
      );
    }
  }
}
```

## Key Takeaways

1. **Microtask queue drains completely** after every task and after every rendering step — new microtasks added during draining are also drained before control returns

2. **rAF is not a macrotask** — it's part of the rendering pipeline, tied to the display refresh cycle, not the task queue

3. **queueMicrotask is synchronous in effect** — it runs before any paint, before any next task; use it for state-consistency deferred work

4. **rAF callbacks receive a shared timestamp** — all rAF callbacks in the same frame share the same timestamp; use it for physics/animation math, not `Date.now()`

5. **requestIdleCallback fires after paint** — DOM mutations inside rIC cause extra repaints; compute in idle, apply in rAF

6. **Always cancel rAF and rIC on component teardown** — orphaned loops prevent GC and waste CPU

7. **Starving the microtask queue blocks rendering** — recursive microtask scheduling is equivalent to an infinite loop from the browser's perspective

8. **Use the `timeout` option for rIC in production** — without it, callbacks may never run during heavy animation phases

9. **Delta-time animation is always correct** — never assume a fixed frame rate; measure actual time elapsed per frame

10. **rIC is not supported in Safari** — always polyfill with a `setTimeout`-based fallback

## Resources

### Specifications
- [HTML Living Standard — Event Loop](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model)
- [requestIdleCallback specification (WICG)](https://www.w3.org/TR/requestidlecallback/)

### Articles
- "Using requestIdleCallback" — Paul Lewis (Google Developers)
- "Microtasks and the Event Loop" — Jake Archibald
- "The Anatomy of a Frame" — Paul Lewis (Aerotwist)

### Tools
- Chrome DevTools Performance panel → Frames view
- `PerformanceObserver` with `longtask` entry type
- `performance.mark()` / `performance.measure()` for custom spans
