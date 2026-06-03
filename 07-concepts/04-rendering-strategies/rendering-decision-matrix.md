# Rendering Strategy Decision Matrix

## Overview

Choosing the wrong rendering strategy is one of the most expensive architectural mistakes in frontend development — it affects performance, SEO, infrastructure cost, and developer experience. This guide provides a systematic decision framework covering CSR, SSR, SSG, ISR, PPR, Edge rendering, Islands architecture, and Resumability — with concrete tradeoffs and real-world examples for each.

## The Rendering Landscape

```
Rendering strategies — where does the work happen?

  Build time                Server runtime           Browser
  ──────────────────────────────────────────────────────────
  SSG (build)               SSR (per request)        CSR
  ISR (on-demand rebuild)   Edge (CDN worker)
  PPR (shell at build,      SSR + Islands
       dynamic at runtime)
                            Streaming SSR
                            Resumability (Qwik)
```

## Strategy Reference

### CSR — Client-Side Rendering

The server sends a minimal HTML shell; all rendering happens in the browser via JavaScript:

```
Client-Side Rendering flow:

Browser        CDN/Server
  │                │
  ├─ GET /page ───▶│
  │◀─ HTML shell ──┤  (empty <div id="root">, script tags)
  │                │
  ├─ GET app.js ──▶│
  │◀─ bundle ──────┤
  │                │
  [JS executes, fetches data, renders DOM]
  │                │
  ├─ GET /api ────▶│─▶ Origin
  │◀─ JSON ────────┤
  │                │
  [FCP after all this — high TTI, low TTFB]
```

**Best for:**
- Dashboards and admin tools behind authentication (SEO irrelevant)
- Highly interactive apps (drawing tools, complex editors, games)
- Real-time collaborative apps where server rendering would be stale immediately
- SaaS apps where repeat visitors are the norm (shell cached after first load)

**Avoid for:** Marketing sites, content-heavy pages, SEO-critical pages, slow devices/connections.

### SSR — Server-Side Rendering

Every request triggers HTML generation on the server:

```
Server-Side Rendering flow:

Browser        Server
  │                │
  ├─ GET /page ───▶│
  │                ├─ fetch data (DB / API)
  │                ├─ render HTML
  │◀─ Full HTML ───┤  (visible content, good FCP)
  │                │
  ├─ GET app.js ──▶│
  │◀─ bundle ──────┤
  │                │
  [Hydration — React takes over]
  │                │
  [TTI after hydration — FCP good, TTI gap]
```

**Best for:**
- Content that changes per-request or per-user (personalized feeds, stock prices)
- Pages where search crawlers need fully rendered HTML
- Apps where authentication gates access to dynamic content
- First-load performance on content that can't be statically generated

**Avoid for:** Ultra-high traffic pages (every request hits server — expensive), pages with only public static content, real-time data that changes faster than a server render cycle.

### SSG — Static Site Generation

Pages rendered at build time; served as static files:

```
SSG flow:

Build time:          Serve time:
─────────────        ─────────────────
npm run build        Browser
  │                    │
  ├─ Fetch all data     ├─ GET /page ───▶ CDN
  ├─ Render HTML        │◀─ Pre-built HTML (near-instant)
  ├─ Write .html files  │
  └─ Deploy to CDN      ├─ GET app.js ──▶ CDN
                        │◀─ bundle
                        │
                        [Hydration]
```

**Best for:**
- Marketing sites, landing pages, blogs, documentation
- Content that changes infrequently (daily deploys acceptable)
- Maximum performance (CDN-served static HTML, no server compute)
- Sites where hosting cost is a priority

**Avoid for:** User-specific content, real-time data, frequently updating content (rebuilding on every change becomes untenable at scale).

### ISR — Incremental Static Regeneration

Hybrid: pre-render at build time, regenerate individual pages in the background:

```
ISR flow (Next.js):

Build time: pre-render high-traffic pages
           ↓
Serve time:
  Request for /products/p1
    ├─ Within max-age → serve stale (instant)
    └─ After max-age → serve stale + trigger background regeneration
                          └─ Next request gets fresh page

  Request for /products/p99 (never built)
    ├─ First request → SSR (builds + caches)
    └─ Subsequent → serve cached (instant)
```

**Best for:**
- E-commerce product pages (thousands of pages, updated occasionally)
- News articles (published once, rarely edited)
- Content where slight staleness is acceptable
- When SSG build time becomes prohibitive (millions of pages)

**Avoid for:** Real-time data, per-user personalization, content where stale serving is unacceptable (prices, inventory).

### PPR — Partial Pre-Rendering (Next.js 15+)

Static shell pre-rendered at build time; dynamic "holes" filled via streaming SSR:

```
PPR flow:

Build time:
  ├─ Render static shell (nav, layout, above-fold content)
  └─ Mark <Suspense> boundaries as "dynamic holes"

Request time:
  Browser ── GET /page ──▶ CDN ──▶ Edge
                             │
                             ├─ Serve static shell instantly (from cache)
                             │
                             └─ Stream dynamic content as it resolves:
                                ├─ User cart (personalised) ← from origin
                                └─ Recommendations ← from origin
```

```typescript
// Next.js PPR usage
// app/page.tsx
import { Suspense } from 'react';

export default function ProductPage({ params }: { params: { id: string } }) {
  return (
    <div>
      {/* Static — pre-rendered at build time */}
      <ProductHeader id={params.id} />

      {/* Dynamic — streamed at request time */}
      <Suspense fallback={<CartSkeleton />}>
        <UserCart />         {/* reads cookies — dynamic */}
      </Suspense>

      <Suspense fallback={<RecommendationsSkeleton />}>
        <Recommendations productId={params.id} />  {/* personalised */}
      </Suspense>
    </div>
  );
}
```

**Best for:** E-commerce, media sites — pages with a mix of static product info and dynamic personalization where you want the best of both SSG (fast first byte) and SSR (fresh dynamic content).

### Edge Rendering

SSR at CDN edge nodes — geographically close to the user:

```
Edge Rendering:

User in Tokyo               User in London
     │                           │
     ▼                           ▼
Tokyo CDN Node            London CDN Node
  (renders HTML)            (renders HTML)
     │                           │
     └────────────▶ Origin ◀─────┘
                (only for data/cache miss)

vs. SSR at single origin:
  User anywhere ──────────▶ us-east-1 ──▶ User  (high latency for far users)
```

**Constraints:** Edge runtimes have limited APIs — no Node.js APIs (`fs`, `crypto`, some `Buffer` operations), short execution time limits, limited memory.

```typescript
// Next.js edge runtime declaration
export const runtime = 'edge';

export default async function handler(request: Request) {
  const geo = request.geo;                          // Cloudflare-specific
  const country = geo?.country ?? 'US';

  const content = await fetch(`/api/content?locale=${country}`);
  const html = renderToString(<Page data={await content.json()} />);

  return new Response(html, {
    headers: { 'Content-Type': 'text/html' },
  });
}
```

**Best for:** Personalization based on geolocation/cookies, A/B testing, auth redirects, middleware transforms — any logic that needs server-side execution but benefits enormously from geographic distribution.

**Avoid for:** Heavy computation, operations requiring Node.js-only APIs, database queries requiring full connection pools.

### Islands Architecture

Static HTML with interactive "islands" of JavaScript hydrated independently:

```
Islands Architecture:

┌─────────────────────────────────────────────────┐
│  Static HTML (no JS)                            │
│                                                 │
│  ┌──────────────┐   ┌──────────────────────┐   │
│  │   Island 1   │   │       Island 2        │   │
│  │  (nav menu)  │   │  (product carousel)   │   │
│  │  [hydrated]  │   │     [hydrated]        │   │
│  └──────────────┘   └──────────────────────┘   │
│                                                 │
│  Static content (text, images) — zero JS cost   │
└─────────────────────────────────────────────────┘
```

```typescript
// Astro islands example
---
// page.astro
import StaticContent from './StaticContent.astro';
import InteractiveCounter from './Counter.react';
import ImageCarousel from './Carousel.vue';
---

<html>
  <body>
    <StaticContent />  <!-- Zero JS, just HTML -->

    <!-- client:load = hydrate immediately -->
    <InteractiveCounter client:load />

    <!-- client:visible = hydrate when in viewport -->
    <ImageCarousel client:visible />

    <!-- client:idle = hydrate when browser is idle -->
    <NewsletterForm client:idle />
  </body>
</html>
```

**Best for:** Content-heavy sites (blogs, marketing, docs) with small interactive zones. Dramatically reduces JavaScript payload — most pages ship zero JS.

**Avoid for:** Highly interactive apps where most of the UI is interactive (React SPA is better — islands overhead outweighs benefit).

### Resumability (Qwik)

Eliminates hydration entirely — serializes all component state and event listeners to HTML, resuming execution on user interaction:

```
Traditional hydration vs Resumability:

Hydration (React):
  Server renders HTML
  Client downloads all JS
  Client re-executes all component trees to attach listeners
  → Expensive startup cost proportional to app size

Resumability (Qwik):
  Server renders HTML + serializes all state to DOM
  Client downloads NO JS upfront
  User interacts → lazy-load only the handler for that interaction
  → O(1) startup regardless of app size
```

**Best for:** Extremely performance-sensitive, SEO-critical applications where TTI on slow networks is paramount. Qwik is the only mainstream framework implementing this model.

**Avoid for:** Teams unfamiliar with Qwik's programming model (different mental model than React), applications where interaction-heavy early usage makes lazy-loading JS inefficient anyway.

## Decision Tree

```
Is this page behind authentication?
  YES → CSR or SSR
    │
    ├─ Highly interactive (dashboard, editor)?
    │    YES → CSR
    │    NO  → SSR with private cache
    │
    └─ Real-time data?
         YES → CSR + WebSocket/SSE
         NO  → SSR

  NO → public page
    │
    ├─ Does content change?
    │    NEVER/RARELY → SSG
    │    PERIODICALLY (hours/days) → ISR
    │    PER REQUEST / REAL-TIME → SSR or Edge
    │
    ├─ Highly interactive?
    │    YES → SSR + hydration (React/Next.js)
    │    Mostly static with small interactive bits → Islands (Astro)
    │
    ├─ Global audience, latency critical?
    │    YES → Edge rendering or SSG + CDN
    │
    └─ Performance budget extremely tight?
         YES → Islands or Resumability (Qwik)
```

## Tradeoffs Table

```
Strategy    TTFB    FCP     TTI     SEO    Freshness  Infra Cost  Best Use Case
─────────────────────────────────────────────────────────────────────────────────
CSR         Fast*   Slow    Slow    Poor   Always     Low         Auth'd dashboards
SSR         Med     Fast    Med     Good   Always     High        Personalized pages
SSG         Fast    Fast    Med     Good   Build-time Minimal     Marketing, blogs
ISR         Fast    Fast    Med     Good   Near-live  Low         E-commerce, news
PPR         Fast    Fast    Med     Good   Dynamic    Med         Mixed static+dyn
Edge SSR    Fast    Fast    Med     Good   Always     Med         Global, personalized
Islands     Fast    Fast    Fast    Good   Build-time Low         Content + small UI
Resumable   Fast    Fast    Fast    Good   Build/SSR  Med         Perf-critical public

* Fast TTFB for CSR because CDN serves empty HTML shell; FCP is slow because JS must run
```

## Real-World Examples

### E-commerce Product Page (ISR + PPR)

```
amazon.com/product/B001
├─ Product title, images, specs ─── SSG/ISR (changes rarely)
├─ Price ────────────────────────── SSR/PPR dynamic (changes often)
├─ "In stock" ───────────────────── SSR/PPR dynamic (real-time)
├─ User's cart count ────────────── CSR / PPR island (per-user)
└─ Reviews ──────────────────────── ISR (updates on new review)
```

### News Article (SSG + Islands)

```
nytimes.com/article/123
├─ Article text ─────── SSG (never changes after publish)
├─ Article images ───── SSG
├─ Share buttons ────── Islands (client:load)
├─ Comment count ────── Islands (client:visible, fetches count)
└─ "Read next" ──────── SSG (pre-computed recommendations)
```

### SaaS Dashboard (CSR)

```
app.linear.app/issues
├─ Everything ─── CSR
   (behind auth, highly interactive, real-time updates via WebSocket)
   SSG/SSR provides no benefit here — content is entirely per-user and dynamic
```

### Marketing Site (SSG + Edge for A/B)

```
stripe.com
├─ All pages ──── SSG (lightning fast, CDN-served)
├─ Pricing page ─ Edge middleware (A/B test variant from cookie)
└─ Docs ────────── SSG with search as CSR island
```

## Common Mistakes

### 1. Using SSR for Static Content

```typescript
// ❌ Running SSR for a page that never changes
export async function getServerSideProps() {
  return { props: { content: await fetchStaticContent() } };
}
// Every request hits the server; content could be pre-built

// ✅ SSG — build once, serve from CDN
export async function getStaticProps() {
  return { props: { content: await fetchStaticContent() } };
}
```

### 2. Using SSG for Real-Time Data

```typescript
// ❌ SSG for a page showing live inventory — users see stale stock count
export async function getStaticProps() {
  const inventory = await fetchInventory(); // correct at build time only
  return { props: { inventory }, revalidate: 3600 };
}

// ✅ Hybrid: SSG for product info, CSR for live inventory
export default function ProductPage({ product }: Props) {
  const { data: inventory } = useSWR(`/api/inventory/${product.id}`);
  return (
    <>
      <ProductInfo product={product} />     {/* from SSG */}
      <InventoryBadge inventory={inventory} /> {/* from SWR/CSR */}
    </>
  );
}
```

### 3. Hydration Mismatch

```typescript
// ❌ Server-rendered HTML differs from client render
function Component() {
  return <div>{new Date().toISOString()}</div>;
  // Server renders "2026-01-01T00:00:00.000Z"
  // Client renders "2026-01-01T00:00:01.234Z" → hydration mismatch!
}

// ✅ Use useEffect for values that differ between server and client
function Component() {
  const [time, setTime] = useState<string | null>(null);
  useEffect(() => setTime(new Date().toISOString()), []);
  return <div>{time ?? 'Loading...'}</div>;
}
```

### 4. Ignoring TTFB vs FCP vs TTI in Strategy Choice

```
Common confusion:
  "SSG is always fastest" → SSG has fast TTFB AND FCP, but TTI still requires JS hydration
  "CSR is fine for perf" → Fast TTFB (empty shell served quickly), but FCP is terrible

For SEO and Core Web Vitals:
  FCP matters most for perceived load (SSG/SSR win)
  TTI matters for interactivity (Islands/Resumability win)
  CLS matters for layout stability (all strategies must handle this)
```

## Interview Questions

### 1. When would you choose ISR over SSG?

**Answer:** ISR when you have too many pages to rebuild entirely on each change (e-commerce with 100,000 product pages), or when content updates frequently enough that a full build per change is impractical. ISR lets you pre-render the most popular pages at build time and generate less popular pages on first request, then regenerate pages in the background at a `revalidate` interval. SSG is preferable when content is truly static, builds are fast, and you want the simplicity of a pure static deployment — no server/edge needed at all.

### 2. What is the hydration problem and how do Islands and Resumability solve it?

**Answer:** Hydration is the process where React (after SSR) downloads the full JavaScript bundle and re-executes all component trees to attach event listeners — even though the HTML is already rendered. For large apps this is expensive CPU work that delays interactivity. Islands architecture solves it by making most of the page static HTML (no JS at all) and only hydrating explicitly marked "island" components. Resumability (Qwik) eliminates hydration entirely — it serializes all event listener references into the HTML so the browser can resume execution at the exact point of user interaction, downloading only the specific handler for that interaction.

### 3. What are the constraints of Edge rendering versus traditional SSR?

**Answer:** Edge runtimes (Vercel Edge, Cloudflare Workers) run in a V8 isolate with no Node.js APIs. You can't use `fs`, native modules, most `crypto` APIs, or libraries that depend on Node.js internals. Execution time is strictly limited (typically 10-50ms CPU time). Memory is limited. Cold starts are near-zero but the compute ceiling is lower. The benefit is geographic distribution — edge nodes run within ~20ms of most users globally, versus 50-200ms to a single-region origin. Edge rendering is ideal for lightweight transformations, auth checks, A/B routing, and geolocation-based personalization — not for heavy database queries or CPU-intensive work.

### 4. How do you decide between SSR and CSR for an authenticated dashboard?

**Answer:** Default to CSR for dashboards: the content is fully per-user (CDN caching provides no benefit), the user logs in once and then navigates heavily (the JS bundle is cached after first load), and the interactive complexity makes heavy hydration unavoidable anyway. Use SSR for authenticated dashboards when: first-load performance is critical on slow devices/networks (SSR ships pre-rendered HTML faster to FCP), SEO of authenticated pages matters (uncommon), or when you need server-only secrets to fetch data. The SSR hydration cost is only worth paying if the FCP improvement meaningfully affects the user's experience or conversion rate.

### 5. What is PPR (Partial Pre-Rendering) and what problem does it solve?

**Answer:** PPR pre-renders the static shell of a page at build time (layout, above-fold content, structure) while leaving dynamic "holes" marked by Suspense boundaries. At request time, the static shell is served instantly from CDN edge cache, and the dynamic content is streamed from the server as it resolves. This solves the classic SSR vs ISR dilemma for mixed pages — a product page's title, images, and description are static (ISR), but the price, stock status, and personalized recommendations are dynamic (SSR). PPR serves the static shell at CDN speed and streams the dynamic parts, giving near-SSG TTFB/FCP with near-SSR freshness for the dynamic portions.
