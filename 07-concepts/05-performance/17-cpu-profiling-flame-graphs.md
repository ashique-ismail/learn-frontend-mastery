# CPU Profiling and Flame Graphs

## The Idea

**In plain English:** CPU profiling is the process of recording which parts of your code are running and for how long, so you can find the slow spots. A flame graph is the chart that visualizes this recording — each bar shows a function that ran, and wider bars mean that function took more time.

**Real-world analogy:** Imagine a chef preparing a multi-course meal and someone films the entire kitchen from above, timing every action. After the meal, you review the footage and notice the chef spent 40 of 60 minutes just chopping vegetables for one dish.

- The kitchen camera recording = the CPU profiler sampling the running code
- Each task the chef performs (chop, stir, plate) = each function call on the call stack
- The width of time spent on each task in the footage = the width of a bar in the flame graph
- Spotting the longest task (chopping) = finding the performance bottleneck to optimize

---

## Overview

CPU profiling measures where your JavaScript spends time. A flame graph is the visual representation: a stack of call frames, width proportional to time spent. Learning to read one is the skill that separates engineers who "added `useMemo`" from engineers who found the actual bottleneck.

---

## How CPU Sampling Works

The V8 profiler uses **sampling**: at regular intervals (typically every 100µs), it captures the current call stack. After thousands of samples, it aggregates them into a statistical picture of where time was spent.

This means:
- Results are statistical, not exact — short functions may not appear
- "Self time" = time in the function itself; "total time" = time including all callees
- Functions not on the stack when sampled are invisible — polling at high frequency catches more

---

## Chrome DevTools Performance Panel

### Recording a Profile

1. **DevTools → Performance tab**
2. Click record, interact with the page, stop
3. Look at the **flame chart** in the "Main" thread lane

Keyboard shortcut: `Ctrl+Shift+E` starts a recording from the console.

```javascript
// Programmatic profiling via console API
console.profile('my-operation');
runExpensiveOperation();
console.profileEnd('my-operation');
// Appears in DevTools Profiles tab
```

### Anatomy of the Performance Panel

```
┌─────────────────────────────────────────────────────────┐
│  FPS  ──────── frame rate bar (red = dropped frames)    │
│  CPU  ──────── CPU usage over time                      │
│  NET  ──────── network waterfall                        │
├─────────────────────────────────────────────────────────┤
│  Main thread (flame chart)                              │
│  ┌──────────────────────────────────────────────────┐   │
│  │ Task  │  Task  │ Long Task (red corner) │  Task  │   │
│  │───────────────────────────────────────────────── │   │
│  │ Event Handler                                    │   │
│  │   └─ React.render                               │   │
│  │        └─ reconcile                             │   │
│  │             └─ MyComponent (wide = slow)        │   │
│  │                  └─ expensiveCalc               │   │
│  └──────────────────────────────────────────────── ┘   │
├─────────────────────────────────────────────────────────┤
│  Timings  ── LCP, FCP, TTI markers                      │
│  Interactions ── user clicks, keyboard events           │
└─────────────────────────────────────────────────────────┘
```

### Reading the Flame Chart

- **X-axis** = time (left to right)
- **Y-axis** = call stack depth (callers on top, callees below)
- **Width** = time spent (self + children) — wider = slower
- **Red corner** on a task = Long Task (>50ms, blocks main thread)
- **Gray frames** = browser internals / native code
- **Bottom-most wide frames** = where to look for savings

```
# Example: finding a bottleneck

Task (120ms)                          ← Long Task
└─ onclick (120ms)
   └─ handleSearch (118ms)            ← all time is here
      ├─ setState (2ms)
      └─ filterProducts (116ms)       ← ← THIS IS THE HOT PATH
         ├─ JSON.parse (80ms)         ← unnecessarily parsing on every keystroke
         └─ Array.filter (36ms)
```

From this: `JSON.parse` inside `filterProducts` should be memoized or moved out of the hot path.

---

## Flame Graph Variants

### Top-Down (Call Tree)
Starts from the outermost caller, drills in. Shows **total time** per frame. Best for finding which entry point triggers expensive work.

### Bottom-Up
Starts from the deepest callee, rolls up. Shows **self time** — how much time a function consumed excluding its callees. Best for finding the actual leaf function that's expensive.

### Call Tree
Aggregates all instances of a function across the timeline. "How much total time did `Array.filter` consume across the entire recording?"

---

## Node.js / V8 CLI Profiling

```bash
# Generate a V8 profile
node --prof server.js
# Produces: isolate-0x....-v8.log

# Convert to human-readable
node --prof-process isolate-0x....-v8.log > profile.txt

# Key sections in output:
# [Summary]: % time in JS, C++, GC, etc.
# [Bottom up (heavy) profile]: hot paths — read this first
```

```bash
# Better: 0x — generates interactive flame graph as HTML
npm install -g 0x
0x server.js
# Opens flame graph in browser automatically
```

### Reading a 0x Flame Graph

```
Orientation: bottom = first call frame, top = deepest frame
Color (0x default):
  Red/orange = hot (>25% of samples)
  Yellow     = warm (5–25%)
  Blue/gray  = cold (<5%)

Wide + colored = bottleneck candidate
```

---

## Performance.mark / Performance.measure (Custom Spans)

Add custom markers that appear in the DevTools flame chart:

```javascript
// Mark points
performance.mark('filter:start');
const result = products.filter(expensiveFilter);
performance.mark('filter:end');

// Measure between marks
performance.measure('filter-duration', 'filter:start', 'filter:end');

// Read results
const [measure] = performance.getEntriesByName('filter-duration');
console.log(measure.duration); // ms

// Appears as a labeled bar in DevTools Performance timeline
// Also visible in PerformanceObserver

const observer = new PerformanceObserver((list) => {
  list.getEntries().forEach(entry => {
    console.log(entry.name, entry.duration);
  });
});
observer.observe({ entryTypes: ['measure'] });
```

---

## PerformanceObserver — Programmatic Profiling in Production

```javascript
// Measure Long Tasks in production (>50ms main thread blocks)
const ltObserver = new PerformanceObserver((list) => {
  list.getEntries().forEach(entry => {
    if (entry.duration > 100) {
      analytics.track('long_task', {
        duration:  entry.duration,
        startTime: entry.startTime,
        // entry.attribution[0] gives the script/function that caused it
        script:    entry.attribution?.[0]?.name,
      });
    }
  });
});
ltObserver.observe({ entryTypes: ['longtask'] });

// Measure LCP element
const lcpObserver = new PerformanceObserver((list) => {
  const entries = list.getEntries();
  const lcp = entries[entries.length - 1]; // last entry = final LCP candidate
  console.log('LCP:', lcp.startTime, lcp.element);
});
lcpObserver.observe({ entryTypes: ['largest-contentful-paint'] });

// Measure INP (Interaction to Next Paint)
const inpObserver = new PerformanceObserver((list) => {
  list.getEntries().forEach(entry => {
    const inp = entry.processingEnd - entry.startTime;
    if (inp > 200) { // INP threshold: good <200ms, needs improvement 200-500ms
      analytics.track('slow_interaction', {
        inp,
        type: entry.name, // 'click', 'keydown', etc.
      });
    }
  });
});
inpObserver.observe({ entryTypes: ['event'], durationThreshold: 16 });
```

---

## React-Specific Profiling

### React DevTools Profiler

1. DevTools → Profiler tab → Record
2. Interact
3. Stop — see flame chart of React render tree
4. Each bar = a component render; width = render time
5. Yellow bars = "why did this render?" — click to see

```jsx
// Wrap with Profiler API for programmatic measurement
import { Profiler } from 'react';

function onRender(id, phase, actualDuration, baseDuration) {
  // id: 'ProductList'
  // phase: 'mount' | 'update'
  // actualDuration: time for this render (ms)
  // baseDuration: estimated time without memoization
  if (actualDuration > 16) {
    analytics.track('slow_render', { component: id, duration: actualDuration });
  }
}

<Profiler id="ProductList" onRender={onRender}>
  <ProductList />
</Profiler>
```

### Why-Did-You-Render

```bash
npm install @welldone-software/why-did-you-render
```

```javascript
// dev-setup.js
import React from 'react';
import whyDidYouRender from '@welldone-software/why-did-you-render';

if (process.env.NODE_ENV === 'development') {
  whyDidYouRender(React, {
    trackAllPureComponents: true,
  });
}

// In a specific component:
MyComponent.whyDidYouRender = true;
// Now logs to console when it re-renders unnecessarily
```

---

## Identifying Common Patterns in Flame Graphs

### Pattern 1: Unnecessary Re-renders (React)

```
onClick (5ms)
└─ setState (4ms)
   └─ React render batch (60ms)   ← flame graph shows many components
      ├─ Header (1ms)             ← re-rendered but nothing changed
      ├─ Sidebar (1ms)            ← re-rendered but nothing changed
      └─ ProductList (58ms)       ← the one that actually needed updating
```

Fix: `React.memo` on Header/Sidebar, or context splitting.

### Pattern 2: Synchronous Computation in Render

```
React render (80ms)
└─ ProductList render (78ms)
   └─ products.filter().sort() (75ms)   ← heavy computation on every render
```

Fix: `useMemo(() => products.filter().sort(), [products, filters])`.

### Pattern 3: Layout Thrashing

```
Event handler (200ms)
├─ element.offsetHeight (read)   ← forces synchronous layout
├─ element.style.height = ...    ← write
├─ element.offsetHeight (read)   ← forces ANOTHER layout
└─ element.style.height = ...    ← write
```

Fix: batch reads before writes, or use `requestAnimationFrame`.

### Pattern 4: Heavy JSON Parsing

```
fetch callback (100ms)
└─ response.json() (90ms)   ← 5MB JSON payload parsed synchronously
```

Fix: paginate the API response, or use a streaming JSON parser.

---

## V8 Optimization and Deoptimization

V8 optimizes "hot" functions using TurboFan (JIT compiler). It can **deoptimize** a function if assumptions are violated:

```javascript
// V8 optimizes this: point always has { x: number, y: number }
function distance(point) {
  return Math.sqrt(point.x ** 2 + point.y ** 2);
}

// This call deoptimizes: unexpected shape
distance({ x: 1, y: 2, z: 3 }); // extra property → different hidden class

// In flame graphs: deoptimized functions show as gray "unknown" frames
// or appear in the "Deoptimizations" section of --prof-process output
```

Look for `[deoptimize]` markers in V8 logs. The fix is usually ensuring consistent object shapes (don't add properties dynamically).

---

## Interview Questions

**Q: What is a flame graph and how do you read it?**
A: A flame graph visualizes CPU sampling data. The X-axis is time, the Y-axis is call stack depth. Each bar is a function; width represents time spent. You find bottlenecks by looking for wide bars near the bottom of the stack — those are the callees consuming the most time. The topmost wide bars identify where in your code the expensive call originates.

**Q: What's the difference between "self time" and "total time" in a CPU profile?**
A: Self time is time spent executing the function's own instructions, excluding time in functions it calls. Total time includes all callees. A function with high total time but low self time is an aggregator — the hot path is in its children. A function with high self time is doing expensive work itself.

**Q: How do Long Tasks relate to INP?**
A: A Long Task (>50ms) blocks the main thread — any user interaction queued during it must wait to be processed. INP (Interaction to Next Paint) measures the delay from interaction to visual response. Long Tasks are the primary cause of high INP. Breaking Long Tasks with `scheduler.yield()`, `setTimeout(0)`, or `useTransition` reduces INP.

**Q: How would you profile a React performance problem in production (not dev tools)?**
A: Wrap suspect components in `<Profiler onRender={...}>` with a callback that sends slow renders (>16ms) to your analytics pipeline. Use `PerformanceObserver` for Long Tasks and INP events. Log the initiating user action from `PerformanceEventTiming.attribution`. Correlate with session replay (LogRocket, Sentry replay) to reproduce the exact interaction.
