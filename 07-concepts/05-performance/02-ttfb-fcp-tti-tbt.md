# TTFB, FCP, TTI, and TBT Performance Metrics

## The Idea

**In plain English:** These are four stopwatch measurements that record exactly when different things happen as a webpage loads — when the server starts replying, when you first see anything on screen, when you can actually click and use the page, and how much time the browser was stuck doing heavy work instead of responding to you.

**Real-world analogy:** Imagine ordering food at a restaurant. You sit down, order, and then wait. First the waiter acknowledges your order (that's the first signal back). Then your appetizer arrives and you can see food on the table. Then your main course arrives and you can finally eat a proper meal. During the wait, the kitchen was occasionally so slammed with one big order that all other orders were put on hold for a bit — add up all those hold-pauses and you get a total "kitchen blocked" time.

- The waiter's first acknowledgement = TTFB (Time to First Byte — the server's first response reaches your browser)
- The appetizer arriving = FCP (First Contentful Paint — you see the first text or image on screen)
- The main course arriving and being able to eat = TTI (Time to Interactive — you can actually click and use the page)
- The total time the kitchen was blocked by big orders = TBT (Total Blocking Time — the sum of time JavaScript was hogging the browser and blocking your clicks)

---

## Overview

Web performance metrics give precise, measurable meaning to "the page is slow." Google's Core Web Vitals and the extended performance metrics — Time to First Byte (TTFB), First Contentful Paint (FCP), Time to Interactive (TTI), and Total Blocking Time (TBT) — each capture a distinct dimension of the user experience. Understanding what drives each metric and how to measure it is essential for making defensible performance improvements.

## The Metrics Timeline

```
Page load timeline:

Navigation starts
│
├─── TTFB ────────────────────────────────────────────────────────────┐
│    First byte of response received                                  │
│                                                                     │
│         FCP ───────────────────────────────────────────────────┐    │
│         First Contentful Paint (text/image visible)            │    │
│                                                                │    │
│                   LCP (tracked separately)                     │    │
│                   Largest Contentful Paint                     │    │
│                                                                │    │
│    ┌────────────────────────────────────────────────────────┐  │    │
│    │ TBT: sum of blocking time from long tasks (>50ms each) │  │    │
│    │ ██░░░░░████░░░░░░████░░████                            │  │    │
│    └────────────────────────────────────────────────────────┘  │    │
│                                                                │    │
│                                         TTI ──────────────────┘    │
│                                         Page reliably interactive   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## TTFB — Time to First Byte

**Definition:** Time from the navigation request being sent to the first byte of the HTTP response being received by the browser.

**What it measures:** Network latency + server processing time. Everything before the browser can start rendering.

### TTFB Breakdown

```
TTFB = DNS lookup + TCP connection + TLS handshake + Request + Wait (server processing)

0ms ──DNS──┬──TCP──┬──TLS──┬──Request──┬──Server processing──┬── First byte
           │       │       │           │                     │
          20ms   40ms    60ms         70ms                  280ms = TTFB
```

### Good/Needs Improvement/Poor Thresholds

```
TTFB:  ≤ 800ms = Good
       800ms–1800ms = Needs Improvement
       > 1800ms = Poor
```

### What Affects TTFB

1. **Server processing time** — database queries, template rendering, auth checks
2. **Geographic distance** — request must travel to origin server (CDN helps)
3. **Network latency** — TLS handshake, TCP slow start
4. **Server cold starts** — serverless functions, containers scaling up
5. **Resource contention** — overloaded servers, connection limits

### Measuring TTFB

```typescript
// PerformanceObserver — works in real browsers
const observer = new PerformanceObserver((list) => {
  const entries = list.getEntriesByType('navigation');
  for (const entry of entries) {
    const nav = entry as PerformanceNavigationTiming;
    const ttfb = nav.responseStart - nav.requestStart;
    console.log(`TTFB: ${ttfb}ms`);
  }
});
observer.observe({ type: 'navigation', buffered: true });

// web-vitals library (recommended)
import { onTTFB } from 'web-vitals';
onTTFB((metric) => {
  console.log(`TTFB: ${metric.value}ms`);
  sendToAnalytics({ name: metric.name, value: metric.value });
});
```

### Optimizing TTFB

```typescript
// 1. CDN — move origin content close to users
// Most effective: static assets cached at edge → near-zero TTFB

// 2. Server-side response caching
import { createHash } from 'crypto';
import { LRUCache } from 'lru-cache';

const responseCache = new LRUCache<string, string>({ max: 1000, ttl: 60_000 });

app.get('/api/popular-products', async (req, res) => {
  const cacheKey = `products:${req.query.category}`;
  const cached = responseCache.get(cacheKey);

  if (cached) {
    res.setHeader('X-Cache', 'HIT');
    res.setHeader('Content-Type', 'application/json');
    return res.send(cached);
  }

  const products = await db.products.findBestSellers(req.query.category);
  const body = JSON.stringify(products);
  responseCache.set(cacheKey, body);
  res.setHeader('X-Cache', 'MISS');
  res.json(products);
});

// 3. Database query optimization
// Slow: N+1 query pattern
const posts = await db.posts.findAll();
const withAuthors = await Promise.all(
  posts.map((p) => db.users.findById(p.authorId)) // N extra queries
);

// Fast: JOIN or batch load
const postsWithAuthors = await db.posts.findAll({
  include: [{ model: db.users }], // single query
});

// 4. Edge functions for simple personalization
// Instead of routing to origin for header-based redirects,
// handle in a 2ms edge function
```

## FCP — First Contentful Paint

**Definition:** Time until the browser renders the first piece of content — text, image, canvas, or non-white SVG.

**What it measures:** How quickly users see that the page is loading (vs staring at a blank screen).

### FCP Thresholds

```
FCP:  ≤ 1.8s = Good
      1.8s–3s = Needs Improvement
      > 3s = Poor
```

### What Affects FCP

1. **Render-blocking resources** — scripts and stylesheets in `<head>` that delay parsing
2. **Server response time (TTFB)** — FCP can't happen before TTFB
3. **Large HTML payload** — browser must parse before it can render
4. **Slow resource loading** — blocking CSS with large font/image imports

### Measuring FCP

```typescript
import { onFCP } from 'web-vitals';

onFCP((metric) => {
  sendToAnalytics({
    name: 'FCP',
    value: metric.value,
    rating: metric.rating, // 'good' | 'needs-improvement' | 'poor'
  });
});

// Browser Performance API
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.name === 'first-contentful-paint') {
      console.log(`FCP: ${entry.startTime}ms`);
    }
  }
});
observer.observe({ type: 'paint', buffered: true });
```

### Optimizing FCP

```html
<!-- 1. Eliminate render-blocking resources -->
<!-- ❌ Blocking stylesheet load -->
<link rel="stylesheet" href="/styles.css">

<!-- ✅ Inline critical CSS; async-load non-critical -->
<style>/* critical CSS for above-the-fold */</style>
<link rel="stylesheet" href="/styles.css" media="print" onload="this.media='all'">

<!-- 2. Preload critical resources -->
<link rel="preload" as="font" href="/fonts/main.woff2" crossorigin>
<link rel="preload" as="image" href="/hero.webp">

<!-- 3. Async/defer scripts that aren't needed for initial render -->
<!-- ❌ Blocks HTML parsing -->
<script src="/analytics.js"></script>

<!-- ✅ Deferred: executes after HTML parsed, before DOMContentLoaded -->
<script defer src="/analytics.js"></script>

<!-- ✅ Async: executes as soon as downloaded (unordered) -->
<script async src="/analytics.js"></script>
```

```typescript
// 4. Server-Side Rendering for FCP
// CSR: FCP delayed until JS executes
// SSR: FCP = TTFB + first render (server sends real HTML)

// 5. Font display optimization
// CSS font-display: swap → shows fallback font while custom font loads
// font-display: optional → uses system font if custom not cached
```

## TTI — Time to Interactive

**Definition:** The time until the page is reliably interactive — the main thread is idle, the network is mostly quiet, and event handlers can respond within 50ms.

**What it measures:** The gap between "page looks loaded" (FCP) and "page actually works." A high TTI means users click buttons that don't respond yet — one of the most frustrating user experiences.

### TTI Definition (Formal)

```
TTI calculation:
1. Start at FCP
2. Look forward for a 5-second quiet window:
   - No long tasks (tasks > 50ms) on the main thread
   - No more than 2 in-flight network requests
3. TTI = start of that 5-second quiet window
4. Walk backward to the last long task before that window — TTI = end of that task

  FCP                                               TTI
   │                                                 │
   │  [long task][  ][long task][ quiet 5s window ]  │
   └─────────────────────────────────────────────────┘
                               ▲
                       end of last long task = TTI
```

### TTI Thresholds

```
TTI:  ≤ 3.8s = Good
      3.8s–7.3s = Needs Improvement
      > 7.3s = Poor
```

### TTI vs FCP: The "Uncanny Valley"

```
The danger zone:

FCP good (1s) │████████████████████████████████│
              │ User sees content, tries to click│
              │ [JS parsing, hydration running]  │
              │ [clicks register — no response!] │
TTI bad (6s)  │                         ████████│
              │           ↑ Zombie page zone     │
```

This is worse than a slow FCP because the user thinks the page is ready but it isn't. Clicks pile up ("rage clicks") and then fire all at once when JS loads.

### Optimizing TTI

```typescript
// 1. Code splitting — don't ship all JS upfront
// Before:
import AdminPanel from './AdminPanel';   // always loaded

// After:
const AdminPanel = React.lazy(() => import('./AdminPanel'));
// Only loads when AdminPanel is rendered

// 2. Reduce JavaScript execution time
// Check Chrome DevTools → Performance tab → Main thread activity

// 3. Break up long tasks with scheduler
// ❌ One long task blocks main thread for 500ms
function processItems(items: Item[]) {
  items.forEach((item) => expensiveProcess(item));
}

// ✅ Yield between chunks — keeps main thread responsive
async function processItemsYielding(items: Item[]) {
  const CHUNK_SIZE = 50;
  for (let i = 0; i < items.length; i += CHUNK_SIZE) {
    const chunk = items.slice(i, i + CHUNK_SIZE);
    chunk.forEach((item) => expensiveProcess(item));

    // Yield control to the browser between chunks
    await new Promise((r) => setTimeout(r, 0));
    // Or: await scheduler.yield() (newer browsers)
  }
}

// 4. Defer non-critical JavaScript
// Third-party chat widgets, analytics, non-critical features
// Load after TTI to not block interactivity
window.addEventListener('load', () => {
  setTimeout(() => {
    loadNonCriticalScripts();
  }, 3000); // Wait until page is interactive
});
```

## TBT — Total Blocking Time

**Definition:** The sum of "blocking time" for every long task (>50ms) between FCP and TTI. Blocking time for a task = task duration − 50ms.

**What it measures:** The severity of main thread blockage during the "loading but interactive" phase. TBT is the most actionable metric because it directly reflects JavaScript execution cost.

### TBT Calculation

```
Long tasks between FCP and TTI:

Task A: 70ms  → blocking time = 70 - 50 = 20ms
Task B: 200ms → blocking time = 200 - 50 = 150ms
Task C: 80ms  → blocking time = 80 - 50 = 30ms

TBT = 20 + 150 + 30 = 200ms
```

### TBT Thresholds

```
TBT:  ≤ 200ms = Good
      200ms–600ms = Needs Improvement
      > 600ms = Poor
```

### Measuring TBT (in Lighthouse / Lab)

```typescript
// TBT is primarily a lab metric (Lighthouse, WebPageTest)
// In real users you measure via Long Tasks API

const observer = new PerformanceObserver((list) => {
  let totalBlockingTime = 0;

  for (const entry of list.getEntries()) {
    if (entry.duration > 50) {
      totalBlockingTime += entry.duration - 50;
    }
  }

  console.log(`Total blocking time so far: ${totalBlockingTime}ms`);
});

observer.observe({ type: 'longtask', buffered: true });
```

### What Creates Long Tasks

```
Common long task sources:

1. Parsing large JS bundles
   → Solution: code split, lazy load, reduce bundle size

2. Hydration of large component trees (React SSR)
   → Solution: streaming SSR, selective hydration, Suspense boundaries

3. Synchronous API calls or data transformation
   → Solution: async/yield, Web Workers

4. Layout/reflow thrashing
   → Solution: batch DOM reads/writes, use requestAnimationFrame

5. Third-party scripts (analytics, ads, chat)
   → Solution: defer loading, use Partytown to move to Web Worker
```

### Web Workers for Heavy Computation

```typescript
// Move CPU-intensive work off the main thread
// worker.ts
self.addEventListener('message', (event: MessageEvent) => {
  const { data, type } = event.data;

  if (type === 'PROCESS_LARGE_DATASET') {
    // This runs in a worker thread — main thread stays responsive
    const result = heavyComputation(data);
    self.postMessage({ type: 'RESULT', result });
  }
});

// main.ts
const worker = new Worker(new URL('./worker.ts', import.meta.url));

worker.postMessage({ type: 'PROCESS_LARGE_DATASET', data: largeArray });
worker.addEventListener('message', (event) => {
  if (event.data.type === 'RESULT') {
    updateUI(event.data.result);
  }
});
```

## Collecting Metrics from Real Users (RUM)

```typescript
import { onTTFB, onFCP, onLCP, onTBT, onINP, onCLS } from 'web-vitals';

function sendToAnalytics(metric: Metric) {
  // Use navigator.sendBeacon for reliability on page unload
  const body = JSON.stringify({
    name: metric.name,
    value: metric.value,
    rating: metric.rating,
    id: metric.id,
    navigationType: metric.navigationType,
    url: location.href,
    userAgent: navigator.userAgent,
    timestamp: Date.now(),
  });

  navigator.sendBeacon('/analytics/vitals', body);
}

// Collect all metrics
onTTFB(sendToAnalytics);
onFCP(sendToAnalytics);
onLCP(sendToAnalytics);
onTBT(sendToAnalytics);  // lab only — not available from real users
onINP(sendToAnalytics);  // Interaction to Next Paint (replaced FID in CWV)
onCLS(sendToAnalytics);
```

## Common Mistakes

### 1. Optimizing Lab Metrics Without Measuring RUM

```typescript
// ❌ Lighthouse score improved but real users unaffected
// Lighthouse uses a simulated device/network; real users vary wildly
// Some optimizations (reduce image size) help both; some (script order) only affect lab

// ✅ Collect real-user metrics and segment by device/connection type
// A p75 field metric of 4s LCP on mobile is more important than
// a Lighthouse score of 95 on desktop fast 4G
```

### 2. Conflating FCP with LCP

```
FCP = any content painted (a loading spinner counts!)
LCP = the largest contentful element

If your spinner appears at 200ms (good FCP) but the hero image appears at 4s (bad LCP),
your FCP looks great but users see a spinner for 4 seconds.

Optimize LCP — it's in Core Web Vitals. FCP is secondary.
```

### 3. Ignoring the FCP-TTI Gap

```typescript
// ❌ SSR with massive client JS bundle
// FCP: 1.2s (great — server sends HTML)
// TTI: 8.5s (terrible — browser hydrates 2MB of JS)
// TBT: 1800ms

// The server rendering bought you a good FCP but your users
// experience a zombie page for 7 seconds

// ✅ Match rendering strategy to JS budget
// If you can't reduce hydration cost, consider Islands or streaming SSR
```

### 4. render-blocking third-party scripts

```html
<!-- ❌ Third-party analytics blocks rendering -->
<head>
  <script src="https://cdn.analytics.com/track.js"></script>
</head>

<!-- ✅ Defer third-party scripts -->
<head>
  <script defer src="https://cdn.analytics.com/track.js"></script>
  <!-- Or load after interactive event -->
</head>
```

## Interview Questions

### 1. What is the difference between TTFB, FCP, TTI, and TBT?

**Answer:** TTFB (Time to First Byte) measures server/network latency — time from request to first byte of response; it's all pre-rendering. FCP (First Contentful Paint) measures when the user first sees any content — the first pixel of text or image. TTI (Time to Interactive) measures when the page becomes reliably responsive — the main thread is free of long tasks and event handlers fire within 50ms. TBT (Total Blocking Time) measures total main-thread blocking (long tasks minus 50ms threshold) between FCP and TTI — a proxy for JavaScript execution cost. They form a chain: fix TTFB to improve FCP, fix TBT to improve TTI.

### 2. Why is a good FCP with a bad TTI worse than both being slow?

**Answer:** When FCP is fast but TTI is slow, the user sees a seemingly-loaded page and starts interacting with it — but JavaScript is still parsing and hydrating. Clicks appear to do nothing (the handlers aren't attached yet). This "zombie page" effect causes rage clicks and a strong negative perception that's worse than a consistent slow load. Users who wait through a blank screen understand they're waiting; users who see content and click a button that does nothing think the site is broken.

### 3. How does TBT relate to TTI and why is TBT more actionable?

**Answer:** TBT and TTI are correlated — reducing TBT reduces TTI. TBT is more actionable because it's specific: it quantifies exactly how much time JavaScript is blocking the main thread. A TBT of 600ms with three long tasks tells you to find and break up those tasks. TTI just gives you a timestamp. TBT is also more robust — it's less sensitive to random jitter in network timing. In Lighthouse, TBT is measured in the lab (simulated throttling) and correlates strongly with field INP (Interaction to Next Paint), which is now a Core Web Vital.

### 4. What are the most effective ways to improve TTI?

**Answer:** (1) Code split aggressively — use `React.lazy`, dynamic imports, and route-based splitting so only the code needed for the initial view is parsed. (2) Reduce bundle size — tree shake, remove unused dependencies, audit your bundle with webpack-bundle-analyzer. (3) Break up long tasks — use `setTimeout(fn, 0)` or `scheduler.yield()` to yield between processing chunks. (4) Move computation to Web Workers — CPU-intensive work (data transformation, CSV parsing, image processing) off the main thread entirely. (5) Defer non-critical scripts — analytics, chat widgets, A/B testing libraries should load after TTI, not before.

### 5. How do you measure performance metrics from real users?

**Answer:** Use the `web-vitals` library, which wraps `PerformanceObserver` with correct metric calculation. Call `onFCP`, `onLCP`, `onINP`, `onCLS`, `onTTFB` and send readings to your analytics endpoint via `navigator.sendBeacon` (reliable even on page unload). Segment by connection type (`navigator.connection.effectiveType`), device category, and page type to find where real users suffer. Lab tools (Lighthouse, WebPageTest) are invaluable for debugging specific regressions but field data from real devices/networks reveals the true distribution — especially the p75 and p95 which Lighthouse misses.
