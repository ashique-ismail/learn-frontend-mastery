# Production Failure: Performance Regression Scenarios

## The Idea

**In plain English:** A performance regression is when your app used to be fast but gradually (or suddenly) becomes slow — without breaking or crashing. It is a slowdown that sneaks in over time as developers add new code, and nobody notices until real users start complaining.

**Real-world analogy:** Imagine a highway that was built with 4 lanes and moves traffic smoothly. Over the years, city workers add toll booths, speed bumps, and detours for construction projects — each change seems minor, but together the drive now takes three times as long. Drivers don't see an error sign; the road still works, it's just painfully slow.

- The highway = your web app's JavaScript bundle and rendering pipeline
- Each new toll booth or speed bump = a new dependency, unoptimized re-render, or blocking data fetch added by a developer
- The traffic jam commuters experience = the slow page load or janky animation that real users suffer through

---

## Overview

Performance regressions are the sneakiest class of production issues. They don't throw errors, don't break tests, and often aren't noticed until users start complaining or a monitoring alert fires. These are the scenarios that separate engineers who "optimized once" from engineers who build systems that stay fast.

---

## Failure 1: Bundle Size Creep

### How It Happens

```
Week 1:  Bundle: 180KB  ✓
Week 4:  Bundle: 220KB  (someone added moment.js)
Week 8:  Bundle: 310KB  (charting library added)
Week 12: Bundle: 450KB  (three more dependencies)
Week 16: Bundle: 600KB  (nobody noticed)
LCP: 1.8s → 4.2s over 16 weeks
```

No single commit was obviously bad. But the cumulative effect cut conversion by 15%.

### Detection

**CI Size Budget (Webpack Bundle Analyzer / bundlewatch):**
```json
// bundlewatch.config.json
{
  "files": [
    { "path": "./dist/main.*.js",    "maxSize": "200kB" },
    { "path": "./dist/vendor.*.js",  "maxSize": "300kB" },
    { "path": "./dist/*.css",        "maxSize": "50kB"  }
  ]
}
```

```yaml
# CI: fail the build if bundles exceed budget
- name: Check bundle size
  run: npx bundlewatch
```

**Import Cost Tracking:**
```bash
# Find what's large
npx source-map-explorer dist/main.js

# Find what a specific import costs
npx bundlephobia-cli moment
# moment: 232.6kB minified, 71.9kB gzipped — reconsider this import
```

### Prevention Patterns

```typescript
// Replace moment.js (232kB) with date-fns tree-shaking (only what you use)
import { format, parseISO } from 'date-fns'; // ~3kB for these two functions

// Replace lodash (69kB) with individual lodash methods or native equivalents
import debounce from 'lodash/debounce'; // 1.7kB — NOT import _ from 'lodash'

// Dynamic import for heavy dependencies only needed on specific routes
const Chart = lazy(() => import('./HeavyChart')); // not in initial bundle
```

---

## Failure 2: Re-render Cascade from Atom Store

### How It Happens

You use Jotai or Recoil. You add a new atom. You connect it to a parent component. Now every component that reads from ANY atom in the dependency tree re-renders when this atom changes.

```typescript
// The problem: one global state change → thousands of re-renders
const userAtom = atom({ id: '1', name: 'Alice', theme: 'dark' });

// 500 components subscribe to userAtom to show the user's name
// User changes their theme → ALL 500 components re-render
// Each renders in ~0.3ms → 150ms of blocked main thread → jank
```

### Detection

React DevTools Profiler → record an interaction → look for the "why did this render?" badge. If hundreds of components show "atom changed" or "context changed" for an interaction that should affect only a few: you have a cascade.

### Fix: Granular Atoms

```typescript
// Split the atom so components subscribe to only what they read
const userNameAtom  = atom('Alice');
const userThemeAtom = atom('dark');
const userIdAtom    = atom('1');

// Component that only needs the name
function UserBadge() {
  const name = useAtomValue(userNameAtom); // only re-renders when name changes
  // theme changes do NOT cause this to re-render
}
```

```typescript
// Selector pattern for derived state
const userDisplayAtom = atom((get) => {
  const name = get(userNameAtom);
  const id   = get(userIdAtom);
  return `${name} (${id})`;
});
// Re-renders only when name OR id changes, not theme
```

---

## Failure 3: Synchronous Layout Read in Animation Loop

### How It Happens

```javascript
// Looks innocent — reads a DOM property in rAF
function animateElements(elements) {
  requestAnimationFrame(() => {
    elements.forEach(el => {
      const height = el.offsetHeight; // SYNC LAYOUT READ ← forces layout
      el.style.transform = `translateY(${-height}px)`; // write
      // Next iteration: read again → layout again → read → layout...
    });
    animateElements(elements);
  });
}

// 100 elements × 2 layout/paint cycles each = 200 forced layouts per frame
// Frame time: 16ms budget. 200 layouts: 80ms. Frame rate: 12fps.
```

### Detection

Chrome DevTools Performance panel → look for "Recalculate Style" + "Layout" occurring repeatedly inside a single frame. A purple "Layout" block immediately after a green "Scripting" block = forced synchronous layout (layout thrashing).

### Fix: Batch Reads Before Writes

```javascript
function animateElements(elements) {
  requestAnimationFrame(() => {
    // PHASE 1: All reads (no writes)
    const heights = elements.map(el => el.offsetHeight);

    // PHASE 2: All writes (using values from phase 1)
    elements.forEach((el, i) => {
      el.style.transform = `translateY(${-heights[i]}px)`;
    });

    animateElements(elements);
  });
}
```

---

## Failure 4: SSR Rendering Blocking on Slow Data

### How It Happens

```typescript
// Next.js page: all data fetched before page renders
export async function getServerSideProps() {
  const [user, products, recommendations] = await Promise.all([
    fetchUser(),        // 50ms
    fetchProducts(),    // 200ms  ← slowest
    fetchRecommendations(), // 150ms
  ]);

  return { props: { user, products, recommendations } };
}
```

Even though user data arrives in 50ms, the page doesn't render until `products` finishes at 200ms. TTFB is 200ms. If `fetchProducts()` spikes to 2 seconds (database slow query): page TTFB = 2 seconds, LCP = 4+ seconds.

### Fix: Stream Critical Content First

```tsx
// Next.js App Router: stream non-critical data with Suspense
export default async function Page() {
  // Fetch critical data — blocks shell render
  const user = await fetchUser(); // fast: 50ms

  return (
    <Shell user={user}>
      {/* Products stream in when ready — page renders immediately */}
      <Suspense fallback={<ProductSkeleton />}>
        <Products /> {/* fetches data inside component — doesn't block shell */}
      </Suspense>

      <Suspense fallback={<RecommendationSkeleton />}>
        <Recommendations />
      </Suspense>
    </Shell>
  );
}
```

```tsx
// Products component: fetches its own data
async function Products() {
  const products = await fetchProducts(); // 200ms — but page already sent shell
  return <ProductGrid products={products} />;
}
```

The shell (with user info, navigation) streams to the browser in ~50ms. Products stream in at ~200ms. Perceived performance is dramatically better even though total data fetch time is the same.

---

## Failure 5: Virtualization Misconfiguration

### How It Happens

You add virtualization to a list of 10,000 items to fix performance. But LCP gets *worse*.

```tsx
// Problem: overscanCount too high → renders more items than needed
<VirtualList
  itemCount={10000}
  overscanCount={50}  // renders 50 items above and below viewport
  // At 50px per item: 50 * 50px = 2500px of off-screen DOM
  // On low-end devices: initial render blocks for 400ms
/>
```

Or: `itemSize` is wrong (fixed size for variable-height items) → items overlap or leave gaps → users see broken layout → they think the app crashed.

```tsx
// Problem: fixed item size for variable content
<FixedSizeList itemSize={50}> {/* but items are 40–200px depending on content */}
```

### Fix

```tsx
// Correct overscan for most cases: 3–5 items, not 50
<VirtualList overscanCount={3} />

// Variable height: use dynamic measurement
import { VariableSizeList } from 'react-window';

const itemSizeCache = new Map<number, number>();
const getItemSize = (index: number) => itemSizeCache.get(index) ?? 60; // fallback

// Measure rendered items and update cache
const itemRef = (el: HTMLDivElement | null, index: number) => {
  if (el) {
    const height = el.getBoundingClientRect().height;
    if (itemSizeCache.get(index) !== height) {
      itemSizeCache.set(index, height);
      listRef.current?.resetAfterIndex(index); // recalculate from here
    }
  }
};
```

---

## Building a Performance Regression Detection System

### The Three Layers

```
Layer 1: CI — catch before deploy
  - Bundle size budgets (bundlewatch)
  - Lighthouse CI score thresholds
  - Import cost limits on PR

Layer 2: Synthetic monitoring — catch after deploy
  - Scheduled Lighthouse runs against production
  - WebPageTest from multiple regions
  - Alert if LCP > 2.5s or CLS > 0.1

Layer 3: RUM — catch for real users
  - PerformanceObserver: LCP, INP, CLS in production JS
  - Session replay (Sentry, LogRocket) to reproduce
  - Segment by: country, device type, connection speed
```

### Performance Budget in CI (Lighthouse CI)

```yaml
# lighthouserc.yml
ci:
  assert:
    assertions:
      first-contentful-paint: ['warn', { maxNumericValue: 2000 }]
      largest-contentful-paint: ['error', { maxNumericValue: 2500 }]
      total-blocking-time: ['error', { maxNumericValue: 300 }]
      cumulative-layout-shift: ['error', { maxNumericValue: 0.1 }]
      interactive: ['warn', { maxNumericValue: 3500 }]
```

---

## Interview Questions

**Q: LCP regressed from 1.8s to 3.2s after a deploy. Walk through your debugging approach.**
A: First, check the deployment: what changed? (bundle size, new dependencies, API response time, render strategy change). Then: (1) Run Lighthouse on both before/after — compare LCP element, which resource is slow. (2) Check network waterfall — is the LCP image deprioritized behind a new large JS file? (3) Check bundle diff — did bundle size increase? (4) Check server timing headers — is the TTFB worse? Isolate whether it's a network/render/resource issue before optimizing.

**Q: How do you prevent performance regressions from being silently introduced?**
A: Three-layer approach: (1) CI bundle size budget — PR fails if bundles exceed limits; (2) Lighthouse CI on every deploy — score thresholds block deployment; (3) RUM in production — PerformanceObserver tracks real LCP/INP/CLS, alerting on degradation. The key insight: you need automated gates because no human reviews performance on every PR.

**Q: We added virtualization to a 10,000-item list and it made performance worse. Why?**
A: Three likely causes: (1) `overscanCount` too high — rendering 50 off-screen items defeats the purpose; (2) Wrong `itemSize` for variable-height content causing layout thrashing as the list recalculates positions; (3) The list container doesn't have a fixed height — virtualization requires the container to have a defined height so it knows the viewport. Also possible: the original perf problem wasn't the long list itself but a re-render cascade — virtualization doesn't help with excessive re-renders of each item.
