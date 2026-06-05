# Fetch Priority API (`fetchpriority`)

## The Idea

**In plain English:** When a browser loads a webpage, it downloads many files at once — images, fonts, scripts. Fetch Priority is a way for you to tell the browser which files matter most, so it loads the important ones first instead of treating everything equally.

**Real-world analogy:** Imagine a waiter at a busy restaurant holding a tray of orders to deliver. Some plates are for a VIP guest who ordered the main course, others are side dishes for other tables. You tell the waiter "deliver that main course first." The waiter reorders the tray accordingly.

- The waiter = the browser's network fetch queue
- Each plate = a file the browser is downloading (image, font, script)
- Telling the waiter which plate is most important = the `fetchpriority` attribute

---

## Overview

The Fetch Priority API lets developers hint to the browser how important a resource is relative to other resources at the same network priority level. The browser uses this to reorder its internal fetch queue, loading critical resources earlier and deferring non-critical ones. The primary use case is improving LCP (Largest Contentful Paint) by boosting the priority of the LCP image.

---

## The Problem

Browsers apply default priorities to resource types:

| Resource | Default priority |
|---|---|
| HTML document | Highest |
| CSS in `<head>` | Highest |
| Fonts | High |
| `<script>` (blocking) | High |
| LCP image (`<img>` above the fold) | High (auto-detected in modern browsers) |
| `<img>` below the fold | Low |
| `fetch()` / XHR | High |
| Prefetch (`<link rel="prefetch">`) | Lowest |

The problem: these defaults are imprecise. A carousel image gets the same priority as every other `<img>`. The hero/LCP image should be highest, but the browser can't always detect it. Background analytics fetches compete with critical API calls.

---

## The `fetchpriority` Attribute

### On `<img>`

```html
<!-- LCP image: boost to highest priority -->
<img src="/hero.webp" alt="Hero" fetchpriority="high" />

<!-- Below-fold carousel images: deprioritize -->
<img src="/carousel-1.webp" alt="" fetchpriority="low" />
<img src="/carousel-2.webp" alt="" fetchpriority="low" />

<!-- Avatar images in a list: explicitly low -->
<img src="/avatar-1.webp" alt="" width="40" height="40" fetchpriority="low" />
```

### On `<link rel="preload">`

```html
<!-- Preload the LCP image at high priority -->
<link rel="preload" as="image" href="/hero.webp" fetchpriority="high" />

<!-- Preload a font — already high by default, but explicit -->
<link rel="preload" as="font" href="/fonts/body.woff2" crossorigin fetchpriority="high" />

<!-- Preload a non-critical script — defer its loading -->
<link rel="preload" as="script" href="/analytics.js" fetchpriority="low" />
```

### On `<script>`

```html
<!-- Defer non-critical script loading in the priority queue -->
<script src="/analytics.js" fetchpriority="low" async></script>

<!-- Boost a critical script -->
<script src="/payment-sdk.js" fetchpriority="high"></script>
```

---

## `fetch()` API — Priority Option

```javascript
// Critical API call — boost its priority
const data = await fetch('/api/critical-data', {
  priority: 'high',
});

// Background sync — don't compete with user-visible requests
fetch('/api/analytics', {
  method: 'POST',
  body: JSON.stringify(event),
  priority: 'low',
});

// Default — browser decides (equivalent to omitting)
fetch('/api/data', { priority: 'auto' });
```

Priority values: `'high'`, `'low'`, `'auto'` (default).

---

## Impact on LCP

LCP is the render time of the largest visible element — usually the hero image. Boosting its fetch priority reduces the time between the browser discovering the image URL and starting to download it.

### Without `fetchpriority`

```html
<!-- Browser discovers these in order and queues them equally -->
<img src="/product-1.webp" />  <!-- gets High priority -->
<img src="/product-2.webp" />  <!-- gets High priority -->
<img src="/product-3.webp" />  <!-- gets High priority -->
<img src="/hero.webp" />       <!-- also High — competes with others -->
```

### With `fetchpriority`

```html
<img src="/product-1.webp" fetchpriority="low" />
<img src="/product-2.webp" fetchpriority="low" />
<img src="/product-3.webp" fetchpriority="low" />
<img src="/hero.webp" fetchpriority="high" />  <!-- fetched first in queue -->
```

Google's research shows `fetchpriority="high"` on the LCP image can improve LCP by 30–40ms on fast connections and 2+ seconds on slow connections.

---

## Priority Hints vs `loading="lazy"`

These are complementary, not alternatives:

```html
<!-- Hero image: high priority + eager loading (default) -->
<img src="/hero.webp" fetchpriority="high" />

<!-- Below-fold images: low priority + lazy loading -->
<img src="/below-fold.webp" fetchpriority="low" loading="lazy" />

<!-- Preloaded resources still benefit from priority hints -->
<link rel="preload" as="image" href="/hero.webp" fetchpriority="high" />
<img src="/hero.webp" /> <!-- browser matches preload, doesn't refetch -->
```

---

## Priority Hints vs `preload`

```html
<!-- preload: tells browser to fetch this resource NOW, before it's discovered -->
<!-- fetchpriority: tells browser how to rank this vs other resources already queued -->
<!-- They compose: preload a resource AND give it a priority hint -->

<link rel="preload" as="image" href="/hero.webp" fetchpriority="high" />
```

`preload` changes *when* the browser starts fetching (earlier than natural discovery).
`fetchpriority` changes *how urgently* it's fetched relative to other queued resources.

---

## HTTP/2 Priority vs `fetchpriority`

HTTP/2 stream prioritization allows servers to weight streams. `fetchpriority` feeds into this — the browser signals the priority of each request to the HTTP/2 server, which can reorder responses.

```
Browser → Server
  GET /hero.webp       (stream 1, weight: 256 = high)
  GET /carousel-1.webp (stream 3, weight: 36  = low)
  GET /carousel-2.webp (stream 5, weight: 36  = low)

Server prioritizes stream 1 → hero loads first even if carousel started downloading
```

This only works if the server respects HTTP/2 priorities (not all do — check your CDN/server config).

---

## Measuring Impact

```javascript
// PerformanceObserver to measure LCP element and its load time
const observer = new PerformanceObserver((list) => {
  const entries = list.getEntries();
  const lcp = entries[entries.length - 1];

  console.log({
    element:   lcp.element,   // the DOM node that is the LCP element
    startTime: lcp.startTime, // when it was rendered (ms from navigation start)
    url:       lcp.url,       // resource URL (if image)
    loadTime:  lcp.loadTime,  // when the resource finished loading
    renderTime: lcp.renderTime, // when it was painted (0 if cross-origin without Timing-Allow-Origin)
  });
});
observer.observe({ entryTypes: ['largest-contentful-paint'] });
```

```javascript
// Measure resource timing to compare prioritized vs non-prioritized
performance.getEntriesByType('resource')
  .filter(r => r.initiatorType === 'img')
  .map(r => ({
    name:         r.name,
    // Time from navigation start to when fetch started
    fetchStart:   r.fetchStart,
    // Time from navigation start to when download completed
    responseEnd:  r.responseEnd,
    duration:     r.duration,
  }));
```

---

## `priority` in React / Next.js

Next.js `<Image>` exposes this via the `priority` prop:

```tsx
import Image from 'next/image';

// LCP image — sets fetchpriority="high" + preloads
<Image
  src="/hero.webp"
  alt="Hero"
  width={1200}
  height={600}
  priority        // ← adds fetchpriority="high" and <link rel="preload">
/>

// Non-critical images — default (lazy loading)
<Image src="/product.webp" alt="Product" width={300} height={300} />
```

---

## Browser Support and Feature Detection

```javascript
// Feature detect (all modern browsers support it)
const supported = 'fetchPriority' in HTMLImageElement.prototype;

// In fetch():
// priority option is available in Chrome 101+, Firefox 132+, Safari 17.2+
// Falls back gracefully — ignored if unsupported
```

---

## Common Patterns

```html
<!-- Pattern 1: E-commerce product page -->
<link rel="preload" as="image" href="/main-product.webp" fetchpriority="high" />
<img src="/main-product.webp" alt="Main product" fetchpriority="high" />
<img src="/thumbnail-1.webp" alt="" fetchpriority="low" />
<img src="/thumbnail-2.webp" alt="" fetchpriority="low" />

<!-- Pattern 2: News article -->
<img src="/hero-article.webp" alt="Article hero" fetchpriority="high" />
<img src="/author-avatar.webp" alt="" width="48" height="48" fetchpriority="low" />

<!-- Pattern 3: Analytics — don't compete with page resources -->
<script>
  fetch('/api/pageview', {
    method: 'POST',
    priority: 'low',
    keepalive: true, // survives page unload
    body: JSON.stringify({ url: location.href }),
  });
</script>

<!-- Pattern 4: Critical font + non-critical font -->
<link rel="preload" as="font" href="/fonts/body.woff2" crossorigin fetchpriority="high" />
<link rel="preload" as="font" href="/fonts/display.woff2" crossorigin fetchpriority="low" />
```

---

## Interview Questions

**Q: What problem does `fetchpriority` solve that `loading="lazy"` doesn't?**
A: `loading="lazy"` controls *whether* an image is fetched eagerly or deferred until near-viewport. `fetchpriority` controls *urgency* among resources already being fetched. An above-fold carousel image will be fetched eagerly (no `loading="lazy"`) but may compete with the LCP hero image. Adding `fetchpriority="high"` to the LCP image and `fetchpriority="low"` to carousel images resolves this competition without delaying the carousel images entirely.

**Q: How can `fetchpriority` improve LCP?**
A: It signals to the browser (and the HTTP/2 server) that the LCP image should be fetched before other images in the queue. The browser allocates more bandwidth to it, the HTTP/2 server deprioritizes other streams, and the image bytes arrive sooner — reducing the time from navigation start to LCP render.

**Q: Should you add `fetchpriority="high"` to every important resource?**
A: No — priority is relative. If you mark five resources as `high`, they all compete equally and the hint becomes meaningless. Use `fetchpriority="high"` only on the single most critical resource (usually the LCP image). Use `fetchpriority="low"` on known non-critical resources (analytics, below-fold images) to free up bandwidth for critical ones.
