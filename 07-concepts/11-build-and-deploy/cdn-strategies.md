# CDN Strategies

## Overview

A Content Delivery Network (CDN) is a geographically distributed network of servers that caches and serves content from locations physically close to users. CDNs are the single most impactful infrastructure change for reducing page load times for geographically distributed users — serving a file from a CDN edge node 20ms away is fundamentally different from serving it from an origin server 200ms away. This guide covers CDN architecture, cache-busting with content hashes, invalidation strategies, and the distinction between static asset CDNs and API/edge CDNs.

## How CDNs Work

```
Without CDN:
  User in Tokyo ──────────────────────────── Origin (us-east-1)
  200ms RTT                                   (300ms TTFB typical)

With CDN:
  User in Tokyo ── Tokyo CDN node ── Origin (us-east-1)
  20ms                cache HIT → 20ms TTFB
                      cache MISS → 20ms + 300ms (origin) + caches for next user

Request flow:
  1. User requests https://cdn.example.com/app.js
  2. DNS resolves to nearest CDN edge node (GeoDNS / Anycast)
  3. Edge checks cache:
     HIT  → Returns cached response (fast path)
     MISS → Edge fetches from origin, caches, returns to user
  4. Next user in same region hits edge → cache HIT
```

## CDN Cache Hierarchy

```
Multi-tier CDN architecture:

  Browser cache (disk)
       │
  CDN Edge Node (closest POP)
       │
  CDN Regional Cache (mid-tier)
       │
  CDN Origin Shield (shield layer — protects origin from traffic spikes)
       │
  Origin Server

Edge Node → cache miss → Regional Cache (faster than going to origin)
Regional Cache → cache miss → Origin Shield
Origin Shield → cache miss → Origin (only one request to origin per cache miss,
                                     not one per edge node)
```

## Cache-Busting with Content Hashes

The most reliable cache-busting strategy: embed a hash of the file content in the filename. When the content changes, the filename changes — the old URL remains cached (correctly, for users who haven't refreshed) while the new URL misses the cache and fetches fresh.

```
Content hash naming:

Before:
  app.js      → cached at CDN for 1 year → user gets stale code

After:
  app.a1b2c3d4.js  → cached at CDN for 1 year (immutable)
  When code changes:
  app.e5f6g7h8.js  → new URL → cache miss → fresh download
  Old URL (app.a1b2c3d4.js) remains valid for in-progress sessions
```

### Webpack Content Hash

```javascript
// webpack.config.js
module.exports = {
  output: {
    filename: '[name].[contenthash:8].js',    // main.a1b2c3d4.js
    chunkFilename: '[name].[contenthash:8].js', // lazy chunks
    assetModuleFilename: 'assets/[name].[hash:8][ext]', // images, fonts
    path: path.resolve(__dirname, 'dist'),
  },
};
// contenthash changes only when that file's content changes
// (unlike chunkhash which changes when any chunk in the compilation changes)
```

### Vite Content Hash

```typescript
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        // Content hash in filename
        entryFileNames: 'assets/[name].[hash].js',
        chunkFileNames: 'assets/[name].[hash].js',
        assetFileNames: 'assets/[name].[hash][extname]',
      },
    },
  },
});
```

### The HTML Entry Point Problem

The HTML file (`index.html`) references hashed assets. It must always be fresh:

```html
<!-- dist/index.html — references hashed assets -->
<link rel="stylesheet" href="/assets/index.a1b2c3d4.css">
<script src="/assets/index.e5f6g7h8.js"></script>
```

The HTML file cannot have a content hash in its filename (it's the entry point). Set it with `no-cache` so browsers and CDNs always revalidate:

```
dist/index.html        Cache-Control: no-cache          (always revalidate)
dist/assets/*.js       Cache-Control: public, max-age=31536000, immutable
dist/assets/*.css      Cache-Control: public, max-age=31536000, immutable
dist/assets/images/*   Cache-Control: public, max-age=31536000, immutable
```

```typescript
// Express / static server configuration
app.use('/assets', express.static('dist/assets', {
  setHeaders(res) {
    // Hashed filenames → immutable, 1 year
    res.setHeader('Cache-Control', 'public, max-age=31536000, immutable');
  },
}));

app.get('*', (req, res) => {
  res.setHeader('Cache-Control', 'no-cache');
  res.sendFile(path.join(__dirname, 'dist', 'index.html'));
});
```

## CDN Invalidation

Even with content hashes, some resources don't have hashes in their URLs and occasionally need explicit invalidation:

```
When you need explicit invalidation:
  - index.html (no hash — must be invalidated on every deploy)
  - API responses cached at the CDN
  - Versioned but un-hashed URLs (/api/v1/config)
  - Images/videos with predictable paths (uploaded user content)
```

### Cloudflare Cache Invalidation

```typescript
// Purge specific URLs via Cloudflare API
async function purgeCloudflareCache(urls: string[]): Promise<void> {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/zones/${process.env.CF_ZONE_ID}/purge_cache`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.CF_API_TOKEN}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ files: urls }),
    }
  );

  const result = await response.json();
  if (!result.success) {
    throw new Error(`Cache purge failed: ${JSON.stringify(result.errors)}`);
  }
}

// Purge index.html on every deploy
await purgeCloudflareCache([
  'https://example.com/',
  'https://example.com/index.html',
]);
```

### Cache Tags (Surrogate Keys) for Targeted Invalidation

```typescript
// Tag responses with logical keys — purge by tag instead of URL
// Cloudflare: Cache-Tag header
// Fastly: Surrogate-Key header

// When serving a product page:
res.setHeader('Cache-Tag', `product:${productId} category:${categoryId}`);
res.setHeader('Cache-Control', 'public, s-maxage=3600');

// When product is updated:
async function onProductUpdate(productId: string) {
  await cloudflare.purgeByTag(`product:${productId}`);
  // Invalidates ALL cached responses tagged with this product
  // regardless of URL — product pages, API responses, thumbnails
}

// Benefits over URL-based purge:
// - One API call invalidates all URLs for a logical entity
// - Works for paginated lists, search results, related content
// - No need to know which URLs are cached
```

## CDN Configuration for SPAs

```typescript
// Vercel — vercel.json
{
  "headers": [
    {
      "source": "/assets/(.*)",
      "headers": [
        { "key": "Cache-Control", "value": "public, max-age=31536000, immutable" }
      ]
    },
    {
      "source": "/(.*).html",
      "headers": [
        { "key": "Cache-Control", "value": "no-cache" }
      ]
    }
  ],
  "rewrites": [
    {
      // SPA fallback — serve index.html for all routes
      "source": "/((?!assets/).*)",
      "destination": "/index.html"
    }
  ]
}
```

```nginx
# nginx — CDN origin or self-hosted
location /assets/ {
  # Content-hashed files — immutable
  add_header Cache-Control "public, max-age=31536000, immutable";
  add_header Vary "Accept-Encoding";
}

location / {
  # HTML — always revalidate
  add_header Cache-Control "no-cache";
  try_files $uri $uri/ /index.html;  # SPA fallback
}
```

## Static CDN vs API/Edge CDN

```
Static CDN:
  Purpose: Cache static files (JS, CSS, images, fonts)
  TTL: Long (1 year with content hashes)
  Invalidation: Rare (change URL to bust cache)
  Examples: AWS CloudFront, Fastly, Cloudflare CDN, jsDelivr

API/Edge CDN (with compute):
  Purpose: Cache API responses, run serverless at edge
  TTL: Short-medium (seconds to minutes)
  Invalidation: Frequent (tag-based purge on data change)
  Examples: Cloudflare Workers, Vercel Edge, AWS Lambda@Edge,
            Fastly Compute@Edge

API CDN caching pattern:
  GET /api/products?category=electronics
  → Cache-Control: public, s-maxage=300, stale-while-revalidate=3600
  → Surrogate-Key: products products:category:electronics
  → Served from CDN for 300s
  → Background revalidation for 3600s
  → On product update: purge tag products:category:electronics
```

## Multi-CDN Strategy

```
Multi-CDN for resilience:

DNS failover:
  Primary CDN (Cloudflare) → fails → DNS switches to secondary (Fastly)

Weighted routing:
  80% of users → CDN A
  20% of users → CDN B
  Gradually shift when migrating CDN providers

Real User Monitoring (RUM) based routing:
  Measure actual CDN performance per user region
  Route each user to the fastest CDN for their location

Tools:
  - NS1 (traffic management)
  - Cloudflare Load Balancing
  - AWS Global Accelerator
```

## CDN Security

```typescript
// Hot-link protection — only allow requests from your domain
// Cloudflare: Hotlink Protection (UI setting)
// Nginx: valid_referers
location /uploads/ {
  valid_referers none blocked ~.example.com;
  if ($invalid_referer) {
    return 403;
  }
}

// Signed URLs for private content (time-limited access)
import { getSignedUrl } from '@aws-sdk/cloudfront-signer';

const url = getSignedUrl({
  url: `https://cdn.example.com/private/${key}`,
  dateLessThan: new Date(Date.now() + 3600 * 1000).toISOString(), // 1 hour
  privateKey: process.env.CF_PRIVATE_KEY!,
  keyPairId: process.env.CF_KEY_PAIR_ID!,
});
// Only users with this signed URL can access the file for 1 hour

// WAF (Web Application Firewall) at CDN layer
// Cloudflare, AWS WAF — blocks malicious traffic before reaching origin
// Rate limiting, bot protection, DDoS mitigation
```

## Common Mistakes

### 1. Not Using Content Hashes

```
❌ app.js without content hash
   Cache-Control: public, max-age=86400

  User visits page → gets app.js → browser caches for 24h
  You deploy a critical fix → user's browser still uses old cached version
  Users must hard-refresh to get fix

✅ app.a1b2c3d4.js with content hash
   Cache-Control: public, max-age=31536000, immutable

  You deploy fix → new filename app.e5f6g7h8.js
  index.html references new filename
  Users get new file immediately (new URL → cache miss)
```

### 2. Caching HTML Files Long-Term

```
❌ index.html with long max-age
  Cache-Control: public, max-age=86400

  Users' browsers cache index.html for 24h
  After a deploy with new asset hashes, users still see old index.html
  → points to old asset URLs → stale content for up to 24h

✅ index.html with no-cache
  Cache-Control: no-cache
  → Browser checks with server on every load
  → 304 Not Modified if unchanged (still fast)
  → New content immediately when deployed
```

### 3. Not Setting Vary: Accept-Encoding

```
❌ Serving both gzip and no-gzip from same cache entry
  → Some users may receive non-gzip content that was gzip-compressed

✅ Always set Vary: Accept-Encoding for compressible resources
   CDN stores separate entries for each encoding
```

### 4. Purging Everything on Deploy

```
❌ Purge all cache on every deploy
  → Every user fetches fresh content from origin → origin overloaded spike
  → Negates caching benefit for resources that didn't change

✅ Only invalidate what changed:
  → index.html (always, on every deploy)
  → Any unversioned resources that actually changed
  → Leave content-hashed assets alone — they're already versioned by URL
```

## Interview Questions

### 1. What is the difference between the CDN edge cache and the browser cache?

**Answer:** Both respect HTTP cache headers (Cache-Control, ETag). The CDN edge cache is a shared cache — one cached copy serves all users in that geographic region. The browser cache is per-user — each user has their own copy. `Cache-Control: public` allows CDN caching; `Cache-Control: private` is browser-only. `s-maxage` overrides `max-age` for shared caches (CDN) specifically. For static assets with content hashes, both can cache for 1 year (max-age=31536000). For HTML, `no-cache` tells both to revalidate — but the CDN serves as a request terminator if it has a cached 200 with a valid ETag, returning 304 without hitting the origin.

### 2. Why do you need content hashes in filenames and what problem do they solve?

**Answer:** Without content hashes, you face a dilemma: long cache TTLs mean users see stale code after deploys; short TTLs mean every page load incurs CDN revalidation overhead. Content hashes solve this by making cache-busting automatic — when `app.js` becomes `app.a1b2c3d4.js`, the URL itself changes. The old URL remains validly cached (browsers serving the old version before refresh are correct). The new URL is a cache miss and fetches fresh. This allows immutable caching (Cache-Control: max-age=31536000, immutable) for all hashed assets while ensuring users always get the latest code when they load or refresh the page (because index.html, which is always fresh, references the new hashed URLs).

### 3. What is cache tag-based invalidation and when would you use it over URL-based purging?

**Answer:** Cache tags (Surrogate-Key/Cache-Tag headers) label cached responses with logical identifiers. When data changes, you purge by tag — all cached responses bearing that tag are invalidated atomically, regardless of URL. URL-based purging requires knowing every URL to invalidate: `/products/p1`, `/products?category=electronics&page=1`, `/api/products/p1`, etc. Tag-based purging replaces this with one call: purge tag `product:p1` — hits all pages, all API responses, all search result pages that displayed that product. Use tag-based for dynamic content (product pages, API responses, user-generated content). Use URL-based purging only for simple cases with predictable, finite URLs like index.html on every deploy.

### 4. How would you configure CDN caching for a React SPA?

**Answer:** Three asset classes with different strategies: (1) Content-hashed JS/CSS/images in `/assets/` — `Cache-Control: public, max-age=31536000, immutable`. These never need invalidation because a new deploy produces new hashes. (2) `index.html` and any un-hashed HTML — `Cache-Control: no-cache` so the CDN always revalidates; 304 Not Modified responses keep this efficient. (3) SPA client-side routing — configure the CDN or origin to serve `index.html` for all paths that don't match a static file (the "SPA fallback"). On Vercel this is a rewrite rule; on nginx `try_files $uri /index.html`. The combination ensures users always get the latest entry point while assets are cached maximally.
