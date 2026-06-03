# Caching: ETag, Cache-Control, and CDN Strategies

## Overview

HTTP caching is one of the highest-leverage performance improvements available to frontend engineers. Done correctly, it eliminates redundant network requests, reduces server load, and delivers near-instant repeat visits. Done incorrectly, users see stale data for hours or can't get critical updates. This guide covers Cache-Control directives, ETag-based conditional requests, stale-while-revalidate, and the interaction between browser caches and CDNs.

## The Caching Hierarchy

```
Request flow with caching:

Browser
  │
  ├─ Service Worker Cache (programmatic, highest priority)
  │
  ├─ Memory Cache (in-tab, fastest, lost on navigation)
  │
  ├─ Disk Cache (persistent, keyed by URL + Vary headers)
  │
  └─ HTTP request ──▶ CDN / Reverse Proxy Cache
                            │
                            └──▶ Origin Server
```

Each layer can serve the response independently. The browser disk cache is governed by HTTP cache headers. The CDN layer respects the same headers (with some vendor-specific extensions).

## Cache-Control Directives

`Cache-Control` is a response header (and request header) that controls caching behavior. Multiple directives are comma-separated:

```
Cache-Control: max-age=3600, stale-while-revalidate=86400, immutable
```

### Key Directives

```
Directive                  Meaning
──────────────────────────────────────────────────────────────────────────
max-age=N                  Cache valid for N seconds from response time
s-maxage=N                 Like max-age but for shared caches (CDN) only
no-store                   Never cache — not even on disk
no-cache                   Cache but revalidate with server before each use
must-revalidate            After max-age, must revalidate (no serving stale)
proxy-revalidate           must-revalidate for shared caches only
public                     Any cache (browser, CDN) may store the response
private                    Only the end user's browser may cache it (not CDN)
immutable                  Content will never change; skip revalidation check
stale-while-revalidate=N   Serve stale while fetching fresh in background
stale-if-error=N           Serve stale if origin returns error (5xx)
```

### Practical Cache-Control Recipes

```typescript
// Express middleware for setting cache headers

// Static assets with content hash (e.g., main.a1b2c3.js)
// Immutable — the hash guarantees content never changes for this URL
app.use('/static', express.static('dist', {
  setHeaders(res) {
    res.setHeader('Cache-Control', 'public, max-age=31536000, immutable');
    // 1 year — browser and CDN can cache indefinitely
  }
}));

// HTML pages — always revalidate (content changes, SPA needs fresh shell)
app.get('/', (req, res) => {
  res.setHeader('Cache-Control', 'no-cache');
  // no-cache means: cache it, but check with server before using
  // NOT the same as no-store (which means: don't cache at all)
  res.sendFile('index.html');
});

// API responses — short TTL with background revalidation
app.get('/api/products', (req, res) => {
  res.setHeader(
    'Cache-Control',
    'public, max-age=60, stale-while-revalidate=300'
  );
  // Fresh for 60s; serve stale for up to 5 minutes while fetching fresh
  res.json(products);
});

// User-specific data — private, short TTL
app.get('/api/profile', authMiddleware, (req, res) => {
  res.setHeader('Cache-Control', 'private, max-age=300');
  // CDN must not cache; browser can cache for 5 minutes
  res.json(req.user);
});

// Sensitive data — never cache
app.get('/api/payment-methods', authMiddleware, (req, res) => {
  res.setHeader('Cache-Control', 'no-store');
  res.json(paymentMethods);
});
```

## ETags and Conditional Requests

An ETag is an opaque identifier for a specific version of a resource. It enables conditional requests — the browser asks "is this resource still version X?" rather than downloading it unconditionally.

```
Conditional request flow:

Initial request:
  GET /api/posts/p1
  ← 200 OK
  ← ETag: "v3-abc123"
  ← Cache-Control: no-cache

  (browser stores response + ETag)

Subsequent request (after max-age expires or with no-cache):
  GET /api/posts/p1
  → If-None-Match: "v3-abc123"

  If resource unchanged:
  ← 304 Not Modified  (no body — just headers)
  ← ETag: "v3-abc123"

  If resource changed:
  ← 200 OK
  ← ETag: "v4-xyz789"
  ← (new body)
```

### ETag Implementation

```typescript
import crypto from 'crypto';
import { Request, Response } from 'express';

// Strong ETag — based on content hash
function generateETag(content: string | Buffer): string {
  const hash = crypto
    .createHash('sha256')
    .update(content)
    .digest('hex')
    .slice(0, 16);
  return `"${hash}"`;
}

// Weak ETag — based on last-modified time (less precise, but faster)
function generateWeakETag(lastModified: Date): string {
  return `W/"${lastModified.getTime()}"`;
}

// Middleware for ETag-based caching
async function withETag(req: Request, res: Response, getData: () => Promise<unknown>) {
  const data = await getData();
  const body = JSON.stringify(data);
  const etag = generateETag(body);

  // Check If-None-Match header
  if (req.headers['if-none-match'] === etag) {
    res.status(304).end(); // Not Modified — no body
    return;
  }

  res.setHeader('ETag', etag);
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Content-Type', 'application/json');
  res.send(body);
}

// Usage
app.get('/api/posts/:id', async (req, res) => {
  await withETag(req, res, () => db.posts.findById(req.params.id));
});
```

### Last-Modified Header (Alternative to ETag)

```typescript
// Last-Modified / If-Modified-Since flow
app.get('/api/articles', async (req, res) => {
  const lastModified = await db.articles.getLastModifiedDate();

  const ifModifiedSince = req.headers['if-modified-since'];
  if (ifModifiedSince) {
    const clientDate = new Date(ifModifiedSince);
    if (lastModified <= clientDate) {
      res.status(304).end();
      return;
    }
  }

  const articles = await db.articles.findAll();
  res.setHeader('Last-Modified', lastModified.toUTCString());
  res.setHeader('Cache-Control', 'no-cache');
  res.json(articles);
});
```

## stale-while-revalidate

`stale-while-revalidate` is one of the most impactful cache directives. It decouples "serve the response" from "fetch a fresh copy":

```
Timeline for: Cache-Control: max-age=60, stale-while-revalidate=300

0s  ─── Request arrives ─── Cache MISS ─── Fetch from origin ─── Store ───▶

0-60s    Fresh zone: serve from cache, no network request

60-360s  Stale zone: serve stale immediately (fast!),
         AND trigger background revalidation

360s+    Stale-if-error zone (if also set): serve stale only on origin error
         Otherwise: must wait for origin (slow again)
```

```typescript
// Ideal for frequently-read, occasionally-updated data
res.setHeader(
  'Cache-Control',
  'public, max-age=0, stale-while-revalidate=86400'
);
// max-age=0: always revalidate preference
// stale-while-revalidate=86400: but serve stale for up to 1 day while doing so

// This pattern is what Vercel uses for ISR (Incremental Static Regeneration)
// The CDN serves the stale page instantly while regenerating in the background
```

## Vary Header

`Vary` tells caches which request headers affect the response — caches must store separate responses for different header values:

```typescript
// Content negotiation — different response for different Accept headers
app.get('/api/export', (req, res) => {
  res.setHeader('Vary', 'Accept');
  // Cache stores separate entries for:
  // Accept: application/json
  // Accept: text/csv
  // Accept: application/xml

  if (req.accepts('csv')) {
    res.setHeader('Content-Type', 'text/csv');
    res.send(toCsv(data));
  } else {
    res.json(data);
  }
});

// Language-specific responses
app.get('/api/content', (req, res) => {
  res.setHeader('Vary', 'Accept-Language');
  res.setHeader('Cache-Control', 'public, max-age=3600');
  const lang = req.headers['accept-language']?.split(',')[0] ?? 'en';
  res.json(getContent(lang));
});

// ⚠️ Vary: Cookie or Vary: Authorization defeats CDN caching
// CDN can't cache per-user — use private or no-store for auth'd responses
```

## CDN vs Browser Cache

```
Cache responsibility split:

                         public           private
                    ┌────────────┐    ┌────────────┐
Static assets       │    CDN     │    │  Browser   │
(content-hashed)    │ (1 year)   │    │ (1 year)   │
                    └────────────┘    └────────────┘

API responses       ┌────────────┐    ┌────────────┐
(no auth)           │    CDN     │    │  Browser   │
                    │ (s-maxage) │    │ (max-age)  │
                    └────────────┘    └────────────┘

Authenticated       ┌────────────┐    ┌────────────┐
API responses       │  BYPASS    │    │  Browser   │
(private)           │ (private)  │    │ (max-age)  │
                    └────────────┘    └────────────┘
```

### CDN-Specific Headers

```typescript
// Cloudflare / Fastly cache control examples
res.setHeader('Cache-Control', 'public, max-age=60, s-maxage=3600');
// Browser caches 60s; CDN caches 3600s

// Surrogate keys / Cache tags (Fastly, Cloudflare Cache Rules)
res.setHeader('Surrogate-Key', `post:${postId} user:${userId}`);
// Allows targeted invalidation: purge all responses tagged post:p1

// Cloudflare Cache-Tag
res.setHeader('Cache-Tag', `category:${category}`);
```

## Cache Invalidation Strategies

```
"There are only two hard things in Computer Science:
 cache invalidation and naming things." — Phil Karlton
```

### Content-Hash URLs (Best for Static Assets)

```typescript
// webpack / Vite output filenames include content hash
// main.[contenthash].js → main.a1b2c3d4.js

// When content changes, the filename changes
// Old: main.a1b2c3d4.js (still cached — valid)
// New: main.e5f6g7h8.js (new URL — cache miss, fetches fresh)

// Result: immutable caching for all versioned assets
// The HTML file (no hash) uses no-cache to always fetch fresh
```

### Surrogate Keys / Cache Tags (CDN-Level Invalidation)

```typescript
// Tag responses with logical keys
// When data changes, purge by tag — not by URL

// Response to GET /api/posts?category=tech
res.setHeader('Surrogate-Key', 'posts posts:category:tech');

// When a tech post is updated, purge all tagged responses
await cloudflare.purgeByTag('posts:category:tech');
// All cached responses tagged with that key are invalidated atomically
```

### Versioned API URLs

```typescript
// Embedding version in URL forces cache miss on change
const apiBase = `/api/v2/posts`;

// Or use query params as cache buster (less clean)
const url = `/api/posts?v=${BUILD_VERSION}`;
```

## Fetch API Cache Modes

```typescript
// Browser fetch() cache modes
const response = await fetch('/api/data', {
  cache: 'default',        // Use HTTP cache headers normally
});

const fresh = await fetch('/api/data', {
  cache: 'no-cache',       // Revalidate with server before using cached copy
});

const forceNew = await fetch('/api/data', {
  cache: 'no-store',       // Don't use or store in cache at all
});

const forceCache = await fetch('/api/data', {
  cache: 'force-cache',    // Use cache even if stale (offline-first)
});

const onlyCache = await fetch('/api/data', {
  cache: 'only-if-cached', // Only return if in cache; error if not
  mode: 'same-origin',
});
```

## Common Mistakes

### 1. Confusing no-cache with no-store

```
no-cache  = "cache it, but always check with server first" (conditional GET)
no-store  = "never save this to any cache at all"

// ❌ Using no-store for HTML — forces full download every navigation
Cache-Control: no-store

// ✅ Use no-cache — allows caching + revalidation (304 saves bandwidth)
Cache-Control: no-cache
```

### 2. Setting max-age Without Content Hashing

```typescript
// ❌ Long max-age on a URL that can change content
res.setHeader('Cache-Control', 'public, max-age=86400');
// URL: /api/config.js
// When you deploy a new config, users are stuck with old version for 24h

// ✅ Use content hash in filename for long max-age
// /static/config.a1b2c3d4.js with max-age=31536000, immutable
// OR: use no-cache / short max-age for mutable URLs
```

### 3. Caching Authenticated Responses in CDN

```typescript
// ❌ Private data cached at CDN — user A sees user B's data
res.setHeader('Cache-Control', 'public, max-age=300');
res.json({ userId: req.user.id, balance: req.user.balance });

// ✅ Mark authenticated responses as private
res.setHeader('Cache-Control', 'private, max-age=300');
// CDN won't cache; each user's browser caches only their own data
```

### 4. Not Setting Vary for Content-Negotiated Responses

```typescript
// ❌ CDN caches one version regardless of Accept header
app.get('/export', (req, res) => {
  res.setHeader('Cache-Control', 'public, max-age=3600');
  // Missing Vary: Accept!
  if (req.accepts('csv')) res.send(csvData);
  else res.json(jsonData);
  // CDN may return CSV to a JSON client or vice versa
});

// ✅ Tell caches that Accept affects the response
app.get('/export', (req, res) => {
  res.setHeader('Cache-Control', 'public, max-age=3600');
  res.setHeader('Vary', 'Accept');
  if (req.accepts('csv')) res.send(csvData);
  else res.json(jsonData);
});
```

### 5. Forgetting stale-if-error for Resilience

```typescript
// ✅ Serve stale if origin is down (resilient to outages)
res.setHeader(
  'Cache-Control',
  'public, max-age=300, stale-while-revalidate=3600, stale-if-error=86400'
);
// If origin returns 5xx, CDN serves cached response for up to 24h
```

## Interview Questions

### 1. What is the difference between no-cache and no-store?

**Answer:** `no-cache` means the response can be stored in the cache, but the browser (or CDN) must revalidate with the server via a conditional request (sending `If-None-Match` or `If-Modified-Since`) before using the cached copy. If the server responds 304 Not Modified, the cached copy is used — saving bandwidth. `no-store` means never store the response in any cache whatsoever — no disk cache, no CDN, nothing. `no-store` is appropriate for sensitive data (payments, PII). `no-cache` is appropriate for HTML pages or resources that change on each deploy but can tolerate the 304 round-trip.

### 2. How does an ETag work and what does a 304 response mean?

**Answer:** An ETag is an opaque version identifier (usually a content hash or version number) the server includes in a response. The browser stores it alongside the cached response. On subsequent requests, the browser sends `If-None-Match: "etag-value"` in the request header. The server compares the ETag to the current version of the resource. If unchanged, it returns `304 Not Modified` with no body — just headers. The browser then uses its cached copy. This saves the cost of transferring the response body while still ensuring freshness. A `304` is effectively a "your cached copy is still valid" signal.

### 3. What is stale-while-revalidate and how does it improve perceived performance?

**Answer:** `stale-while-revalidate=N` tells the cache: serve the stale (expired) response immediately while asynchronously fetching a fresh copy in the background. This decouples latency from freshness. Instead of the user waiting for the network on every cache miss, they get an instant response from cache and the fresh content appears on the next interaction (or after the background fetch completes). Vercel's ISR (Incremental Static Regeneration) uses exactly this pattern at the CDN level. The tradeoff is that a user may briefly see slightly stale data — acceptable for most content but not for financial or real-time data.

### 4. Why should you use content-hashed filenames for static assets?

**Answer:** Content-hash filenames (`main.a1b2c3d4.js`) allow setting `Cache-Control: immutable, max-age=31536000` — telling every cache layer to hold the file for a year without ever revalidating. When code changes, the build produces a new hash (`main.e5f6g7h8.js`), which is a completely new URL — cache miss guaranteed. The old URL remains valid in caches (still correct for users who haven't refreshed). Only the HTML entry point (which references the hashed asset URLs) needs `no-cache` to pick up new filenames. This gives maximum cacheability for assets while ensuring instant updates.

### 5. How do CDN caches interact with Cache-Control headers?

**Answer:** CDNs respect `Cache-Control` like any shared cache. `public` allows CDN caching; `private` prohibits it. `s-maxage` overrides `max-age` specifically for shared caches (CDN) — useful when you want the CDN to cache longer than the browser. CDNs also support vendor-specific extensions: Cloudflare's `Cache-Tag`, Fastly's `Surrogate-Key` for targeted purges. When `Vary` lists a header like `Accept-Encoding`, CDNs store separate entries per value. Authorization invalidates CDN caching unless explicitly allowed — CDNs typically bypass their cache for requests with an `Authorization` header unless `Cache-Control: public` is set alongside it.
