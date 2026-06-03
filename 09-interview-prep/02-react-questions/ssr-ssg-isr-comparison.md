# SSR vs SSG vs ISR — Rendering Strategy Comparison

## CSR (Client-Side Rendering) — The Baseline Problem

The browser receives an empty HTML shell and downloads JS to render:

```
Server: <div id="root"></div> + app.js (500 KB)
Browser: Download JS → Parse → Execute → Fetch data → Render
```

**Problems:** Blank page until JS loads (poor FCP), no content for SEO crawlers, slow on low-end devices.

---

## SSR (Server-Side Rendering)

HTML is generated **per request** on the server with real data:

```
Request → Server fetches data + renders HTML → Full HTML sent → Hydration
```

```tsx
// Next.js App Router (always SSR unless you opt out)
export default async function ProductPage({ params }) {
  const product = await db.getProduct(params.id);  // every request
  return <ProductView product={product} />;
}
```

**Pros:**
- Fresh data on every request
- Full HTML for SEO and FCP
- Personalized content (user-specific pages)

**Cons:**
- Higher TTFB (server must compute before responding)
- More server load
- Slower than serving static files from CDN

**Best for:** Pages with frequently changing, user-specific, or real-time data (dashboards, shopping carts, logged-in views).

---

## SSG (Static Site Generation)

HTML is generated **at build time** and served from a CDN:

```
Build time → Fetch data + render → Save .html files → CDN
Request → CDN serves pre-built HTML instantly
```

```tsx
// Next.js App Router — opt into SSG with no dynamic behavior
export default async function BlogPost({ params }) {
  const post = await getPost(params.slug); // runs at build time only
  return <Article post={post} />;
}

export async function generateStaticParams() {
  const posts = await getAllPosts();
  return posts.map(p => ({ slug: p.slug }));
}
```

**Pros:**
- Near-instant response (CDN delivery)
- No server load at runtime
- Perfect Lighthouse scores

**Cons:**
- Data becomes stale immediately after build
- Full rebuild required for updates
- Slow builds for thousands of pages

**Best for:** Marketing pages, blog posts, documentation, landing pages — content that changes infrequently.

---

## ISR (Incremental Static Regeneration)

SSG pages are **regenerated in the background** after a specified revalidation window:

```tsx
// Next.js — regenerate every 60 seconds
export const revalidate = 60; // seconds

export default async function ProductPage({ params }) {
  const product = await db.getProduct(params.id);
  return <ProductView product={product} />;
}
```

**How it works:**
1. First request after expiry: serve stale page, trigger background rebuild
2. Next request: serve freshly rebuilt page
3. (stale-while-revalidate semantics)

On-demand revalidation (purge a specific path):
```ts
// Server action or route handler
import { revalidatePath } from 'next/cache';
revalidatePath('/products/[id]', 'page');
```

**Pros:** Static performance + relatively fresh data + no full rebuild

**Cons:** Data may be up to `revalidate` seconds stale; "first visitor after expiry" gets stale content

**Best for:** E-commerce product pages, news articles, sports scores — content that updates periodically but not per-user.

---

## PPR — Partial Prerendering (Next.js 14+)

The newest model: static shell with dynamic Suspense holes:

```tsx
export default function Page() {
  return (
    <main>
      <StaticHeader />           {/* pre-rendered at build time */}
      <Suspense fallback={<Skeleton />}>
        <DynamicPersonalizedFeed /> {/* rendered dynamically per request */}
      </Suspense>
    </main>
  );
}
```

The static shell is CDN-cached; dynamic parts stream in. Best of both worlds.

---

## Decision Matrix

| Strategy | Data freshness | Performance | Server cost | Best for |
|----------|---------------|-------------|-------------|----------|
| CSR | Real-time | Slow FCP | Low | Behind-auth dashboards |
| SSR | Per-request | Medium | High | User-specific pages |
| SSG | Build time | Fastest | Near-zero | Marketing, docs, blogs |
| ISR | Configurable | Fast (CDN) | Low | Product catalogs, news |
| PPR | Mixed | Fast + fresh | Low | Most app pages |

---

## Common Interview Questions

**Q: When does SSR make sense over SSG?**
When content is personalized per user (auth state, preferences, user-specific data), real-time (stock prices, live scores), or too numerous/dynamic to pre-generate at build time.

**Q: What's the TTFB difference between SSR and SSG?**
SSG: ~10-50ms (CDN cache hit). SSR: 100-500ms+ depending on DB query time and server location. SSR adds the full server computation time to TTFB.

**Q: Can you mix strategies in Next.js?**
Yes. Each page/route segment chooses its own strategy. The App Router defaults to SSR unless you use `generateStaticParams` (SSG) or `revalidate` (ISR).

**Q: What is hydration and why does it matter?**
After the server sends HTML, React "hydrates" it — attaches event handlers and makes it interactive. During hydration, the page looks functional but isn't interactive. Hydration is the cost of SSR over pure CSR.
