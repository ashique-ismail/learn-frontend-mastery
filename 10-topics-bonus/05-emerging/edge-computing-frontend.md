# Edge Computing for Frontend Engineers

## Overview

Edge computing moves server-side logic from a single centralized data center to hundreds of nodes distributed globally, colocated near users. For frontend engineers, edge functions replace or augment traditional serverless functions, enabling middleware, personalization, A/B testing, authentication, and SSR with dramatically lower latency — often under 10ms cold start vs 200–500ms for Lambda.

```
Traditional serverless (Lambda/Cloud Functions):
  User in Tokyo ──────────────────────────────► us-east-1 (150ms)
  User in Paris ──────────────────────────────► us-east-1 (90ms)

Edge functions:
  User in Tokyo ──► Tokyo edge PoP (5ms)
  User in Paris ──► Paris edge PoP (3ms)

                                    ↓
                         Each PoP runs your code
```

## What Runs at the Edge vs at the Origin

Understanding this boundary is critical — edge environments are intentionally constrained.

```
Edge (V8 Isolates):               Origin (Node.js / full server):
─────────────────────             ──────────────────────────────
✓ Request/Response manipulation   ✓ Full Node.js APIs
✓ JSON parsing/generation         ✓ Native modules (bcrypt, sharp)
✓ JWT verification                ✓ TCP/UDP connections
✓ A/B testing                     ✓ Long-running processes
✓ Geolocation routing             ✓ Large memory allocations
✓ Header manipulation             ✓ File system access
✓ URL rewrites/redirects          ✓ Prisma / SQL databases (usually)
✓ KV store reads (Cloudflare KV)  ✓ Full ORMs
✓ HTML rewriting (HTMLRewriter)   ✓ Complex computations
✗ Node.js built-ins               ✗ (not needed at edge)
✗ fs, net, path modules           
✗ Many npm packages               
✗ Long execution (>30s)           
```

## Cloudflare Workers

Cloudflare Workers run in V8 isolates — not Node.js. They have access to the Web Platform APIs (fetch, crypto, streams, URL) but not Node.js APIs.

### Hello World Worker

```typescript
// worker.ts
export default {
  async fetch(
    request: Request,
    env: Env,
    ctx: ExecutionContext
  ): Promise<Response> {
    const url = new URL(request.url);

    if (url.pathname === '/api/hello') {
      return new Response(JSON.stringify({ message: 'Hello from the edge!' }), {
        headers: {
          'Content-Type': 'application/json',
          'X-Edge-Location': request.cf?.colo ?? 'unknown',
        },
      });
    }

    return new Response('Not Found', { status: 404 });
  },
};

interface Env {
  KV_NAMESPACE: KVNamespace;
  SECRET_KEY: string;
}
```

### Cloudflare KV — Edge Key-Value Storage

```typescript
// Reading from KV — globally replicated, eventually consistent
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const key = new URL(request.url).searchParams.get('key');
    if (!key) return new Response('Missing key', { status: 400 });

    // KV read — served from nearest data center
    const value = await env.MY_KV.get(key, 'text');

    if (!value) {
      return new Response('Not found', { status: 404 });
    }

    return new Response(value, {
      headers: { 'Cache-Control': 'public, max-age=60' },
    });
  },
};

// Writing to KV (usually from origin, not edge — KV has eventual consistency)
await env.MY_KV.put('feature-flags', JSON.stringify({ newUI: true }), {
  expirationTtl: 3600, // 1 hour
});
```

### Geolocation and Request Data

Cloudflare Workers have access to rich request metadata:

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const cf = request.cf;

    // Route based on geography
    const country = cf?.country ?? 'US';
    const timezone = cf?.timezone ?? 'UTC';
    const city = cf?.city ?? 'Unknown';

    if (country === 'DE' || country === 'FR') {
      // Redirect to EU data center
      return Response.redirect(
        `https://eu.myapp.com${new URL(request.url).pathname}`,
        302
      );
    }

    return new Response(
      JSON.stringify({ country, timezone, city }),
      { headers: { 'Content-Type': 'application/json' } }
    );
  },
};
```

### HTMLRewriter — Transform HTML on the Edge

```typescript
// Inject personalization into HTML without re-rendering the page
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const response = await fetch(request);

    // Get user session from cookie
    const sessionId = getCookie(request, 'session');
    const user = sessionId ? await env.KV.get(`session:${sessionId}`, 'json') as any : null;

    return new HTMLRewriter()
      .on('#user-name', {
        element(el) {
          if (user?.name) {
            el.setInnerContent(user.name);
          }
        },
      })
      .on('[data-auth="required"]', {
        element(el) {
          if (!user) {
            el.setAttribute('hidden', 'true');
          }
        },
      })
      .on('head', {
        element(el) {
          // Inject personalized meta tags
          el.append(
            `<meta name="user-id" content="${user?.id ?? ''}">`,
            { html: true }
          );
        },
      })
      .transform(response);
  },
};
```

## Vercel Edge Functions

Vercel Edge Functions use the same V8 isolate model but integrate tightly with the Next.js App Router.

### Edge Middleware (Next.js)

```typescript
// middleware.ts — runs on every request at the edge
import { NextRequest, NextResponse } from 'next/server';
import { jwtVerify } from 'jose';

export const config = {
  // Only run on these paths — don't match static files
  matcher: ['/dashboard/:path*', '/api/protected/:path*'],
};

export async function middleware(request: NextRequest) {
  const token = request.cookies.get('auth-token')?.value;

  if (!token) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  try {
    const secret = new TextEncoder().encode(process.env.JWT_SECRET!);
    const { payload } = await jwtVerify(token, secret);

    // Clone request and add user info to headers
    const requestHeaders = new Headers(request.headers);
    requestHeaders.set('x-user-id', payload.sub as string);
    requestHeaders.set('x-user-role', payload.role as string);

    return NextResponse.next({
      request: { headers: requestHeaders },
    });
  } catch {
    return NextResponse.redirect(new URL('/login', request.url));
  }
}
```

### Edge API Routes

```typescript
// app/api/geo/route.ts
export const runtime = 'edge'; // Opt into edge runtime

import { NextRequest } from 'next/server';

export async function GET(request: NextRequest) {
  // Vercel provides geo data
  const country = request.geo?.country ?? 'US';
  const city = request.geo?.city ?? 'Unknown';
  const region = request.geo?.region ?? '';

  // Available only in edge runtime
  const ip = request.ip ?? 'unknown';

  return Response.json({
    country,
    city,
    region,
    ip,
    timestamp: Date.now(),
  });
}
```

## Edge Limitations — What You CANNOT Do

These are the most common pitfalls when migrating to edge:

```typescript
// ✗ WRONG — Node.js built-ins don't exist at edge
import { createHash } from 'crypto'; // ReferenceError in V8 isolate
import path from 'path';              // Module not found
import { readFileSync } from 'fs';    // Module not found

// ✓ RIGHT — Use Web Crypto API
const hashBuffer = await crypto.subtle.digest(
  'SHA-256',
  new TextEncoder().encode(data)
);
const hashHex = Array.from(new Uint8Array(hashBuffer))
  .map((b) => b.toString(16).padStart(2, '0'))
  .join('');

// ✗ WRONG — Prisma doesn't run at edge (TCP connections)
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient(); // Won't work

// ✓ RIGHT — Use Prisma's edge-compatible client + connection pooler
import { PrismaClient } from '@prisma/client/edge';
import { withAccelerate } from '@prisma/extension-accelerate';
const prisma = new PrismaClient().$extends(withAccelerate());

// ✗ WRONG — npm packages that use Node.js APIs
import bcrypt from 'bcrypt'; // Uses native modules

// ✓ RIGHT — Use Web Crypto API equivalents
async function hashPassword(password: string): Promise<string> {
  const encoder = new TextEncoder();
  const salt = crypto.getRandomValues(new Uint8Array(16));
  const keyMaterial = await crypto.subtle.importKey(
    'raw',
    encoder.encode(password),
    'PBKDF2',
    false,
    ['deriveBits']
  );
  const bits = await crypto.subtle.deriveBits(
    { name: 'PBKDF2', hash: 'SHA-256', salt, iterations: 100000 },
    keyMaterial,
    256
  );
  // Encode salt + hash
  const combined = new Uint8Array([...salt, ...new Uint8Array(bits)]);
  return btoa(String.fromCharCode(...combined));
}
```

## A/B Testing at the Edge

Edge is perfect for A/B testing because you can split traffic before any HTML is generated:

```typescript
// middleware.ts
export async function middleware(request: NextRequest) {
  const url = request.nextUrl.clone();
  
  // Check if user already has a variant assigned
  const existingVariant = request.cookies.get('ab-variant')?.value;

  let variant: 'a' | 'b';

  if (existingVariant === 'a' || existingVariant === 'b') {
    variant = existingVariant;
  } else {
    // Deterministic assignment based on user ID (sticky sessions)
    const userId = request.cookies.get('user-id')?.value ?? crypto.randomUUID();
    variant = getVariantForUser(userId, 'homepage-cta-test');
  }

  // Rewrite URL to variant-specific page
  if (url.pathname === '/') {
    url.pathname = variant === 'b' ? '/variants/homepage-b' : '/';
  }

  const response = NextResponse.rewrite(url);
  
  // Set cookie for consistent experience
  response.cookies.set('ab-variant', variant, {
    maxAge: 60 * 60 * 24 * 30, // 30 days
    httpOnly: false, // Analytics scripts need to read this
    sameSite: 'lax',
  });

  // Track in edge analytics
  response.headers.set('x-ab-variant', variant);

  return response;
}

function getVariantForUser(userId: string, testName: string): 'a' | 'b' {
  // Deterministic hash — same user always gets same variant
  let hash = 0;
  const str = `${userId}:${testName}`;
  for (let i = 0; i < str.length; i++) {
    hash = ((hash << 5) - hash + str.charCodeAt(i)) | 0;
  }
  return Math.abs(hash) % 100 < 50 ? 'a' : 'b';
}
```

## Edge SSR — When It Makes Sense

```
Use edge SSR when:
  ✓ Page is personalized per user
  ✓ Content changes frequently
  ✓ You need request headers/cookies in the render
  ✓ Users are globally distributed

Avoid edge SSR when:
  ✗ Page is static — use CDN caching instead
  ✗ Page requires heavy computation
  ✗ Page queries a database directly (use origin + cache)
  ✗ Page requires Node.js APIs
```

### Caching at the Edge

```typescript
// app/api/products/route.ts
export const runtime = 'edge';

export async function GET(request: Request) {
  const url = new URL(request.url);
  const category = url.searchParams.get('category') ?? 'all';

  // Check edge cache first
  const cacheKey = `products:${category}`;
  const cache = caches.default; // Cloudflare/Vercel edge cache

  const cached = await cache.match(new Request(cacheKey));
  if (cached) {
    return new Response(await cached.text(), {
      headers: {
        'Content-Type': 'application/json',
        'X-Cache': 'HIT',
      },
    });
  }

  // Fetch from origin
  const data = await fetch(`https://api.example.com/products?category=${category}`);
  const products = await data.json();

  const response = new Response(JSON.stringify(products), {
    headers: {
      'Content-Type': 'application/json',
      'Cache-Control': 'public, s-maxage=60, stale-while-revalidate=300',
      'X-Cache': 'MISS',
    },
  });

  // Store in edge cache
  await cache.put(new Request(cacheKey), response.clone());

  return response;
}
```

## Durable Objects (Cloudflare)

For stateful edge logic (e.g., real-time counters, rate limiting, WebSocket coordination):

```typescript
// durable-object.ts — runs as a singleton at the edge
export class RateLimiter {
  private state: DurableObjectState;
  private requests: number = 0;
  private windowStart: number = Date.now();

  constructor(state: DurableObjectState) {
    this.state = state;
  }

  async fetch(request: Request): Promise<Response> {
    const now = Date.now();
    const windowMs = 60_000; // 1 minute

    // Reset window
    if (now - this.windowStart > windowMs) {
      this.requests = 0;
      this.windowStart = now;
    }

    this.requests++;

    if (this.requests > 100) {
      return new Response('Rate limit exceeded', {
        status: 429,
        headers: {
          'Retry-After': String(Math.ceil((this.windowStart + windowMs - now) / 1000)),
        },
      });
    }

    return new Response(JSON.stringify({
      allowed: true,
      remaining: 100 - this.requests,
    }));
  }
}

// Worker using Durable Object
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const ip = request.headers.get('CF-Connecting-IP') ?? 'unknown';

    // Each IP gets its own Durable Object instance
    const id = env.RATE_LIMITER.idFromName(ip);
    const limiter = env.RATE_LIMITER.get(id);

    return limiter.fetch(request);
  },
};
```

## Performance Comparison

```
Metric            | Lambda (us-east-1) | Cloudflare Worker | Vercel Edge
──────────────────|────────────────────|───────────────────|────────────
Cold start        | 200–500ms          | < 5ms             | < 10ms
P50 latency (EU)  | 110ms              | 8ms               | 12ms
P50 latency (APAC)| 160ms              | 6ms               | 15ms
Max memory        | 10GB               | 128MB             | 128MB
Max execution     | 15min              | 30ms (CPU time)   | 30s
Node.js support   | Full               | None (V8 only)    | None (V8 only)
Pricing           | Per GB-second      | Per request       | Included in plan
```

## Interview Questions

1. **Why do Cloudflare Workers use V8 isolates instead of containers or Node.js processes?**
   V8 isolates start in microseconds (vs hundreds of milliseconds for containers) because they don't require an OS process, virtual machine, or Node.js runtime initialization. Each isolate is a sandboxed JavaScript VM that shares the same V8 instance across many requests. Memory is isolated but the overhead is ~3MB per isolate vs ~128MB for a container. This makes cold starts essentially zero.

2. **What is the difference between edge middleware (Next.js) and edge API routes?**
   Middleware runs on every matching request before any rendering, making it ideal for auth, redirects, and request transformation. It cannot return custom response bodies with heavy data — it should redirect or rewrite. Edge API routes are full request handlers that return arbitrary responses. Both run at the edge in V8 isolates.

3. **How do you handle database access from edge functions given that most ORMs don't support edge environments?**
   Options: (1) Use HTTP-based data layers (Turso, Upstash Redis, Cloudflare KV/D1) designed for edge. (2) Use Prisma Accelerate, which proxies Prisma queries over HTTP. (3) Fetch from an origin API that handles the database. (4) Cache frequently-read data in KV/edge cache and write to origin asynchronously. The key constraint is that edge workers can't make raw TCP connections.

4. **When does edge rendering NOT make sense for performance?**
   When: (1) The page is fully static — CDN caching with `Cache-Control: s-maxage` is faster because no code runs. (2) The response requires a database round-trip to a single-region DB — the edge-to-DB RTT dominates. (3) The computation is CPU-intensive (edge has only 30ms CPU time). (4) The page doesn't vary by user — cache at the CDN level instead.

5. **Explain how A/B testing at the edge avoids the "layout shift" problem that client-side A/B testing has.**
   Client-side A/B testing: the browser renders variant A first, then JavaScript loads and switches to variant B, causing a flash of original content (FOOC) or layout shift. Edge A/B testing: before any HTML is sent, the middleware assigns the variant and rewrites the URL. The browser receives only variant B's HTML from the start — no layout shift, no double render, no visible flicker.

6. **What are Durable Objects, and what problems do they solve that KV cannot?**
   KV is eventually consistent — writes propagate across the globe in seconds. Durable Objects are strongly consistent, single-threaded actors that live in one specific location. They're used for: rate limiting (requires exact counts), real-time chat (single source of truth for room state), WebSocket coordination, and any logic that requires transactions or sequential state changes. Each Durable Object processes requests one at a time, eliminating race conditions.

7. **How would you implement feature flags at the edge without a round-trip to an external service?**
   Store feature flags in Cloudflare KV or Vercel Edge Config (which is specifically designed for this — reads from an in-memory store, not a network request). On flag updates, write to KV/Edge Config. The edge function reads the flags at request time with near-zero latency. Add a short TTL to the KV read to balance freshness vs performance.

8. **What security considerations are specific to edge functions?**
   (1) Secrets/env vars are injected at deploy time — don't log them. (2) No filesystem — can't accidentally expose server files. (3) Short execution time limits damage from runaway code. (4) Shared V8 instance means proper isolation is critical — never share memory between requests (all state must be per-request). (5) IP-based rate limiting is straightforward. (6) Be careful with `request.cf` data — it can be spoofed in testing but is authentic from Cloudflare's network.
