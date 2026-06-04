# Edge Rendering

## Overview

Edge Rendering is a modern rendering strategy where pages are server-side rendered at CDN edge locations geographically close to users, rather than at a centralized origin server. This approach combines the benefits of SSR (dynamic content) with the low latency of edge networks, providing fast, personalized experiences globally.

## How Edge Rendering Works

```
Traditional SSR:
┌──────────────────────────────────────────────────────┐
│  User in Tokyo → Origin Server in US → Response     │
│  Latency: ~200ms base + rendering time              │
├──────────────────────────────────────────────────────┤
│  ┌─────────┐         ┌───────────┐                  │
│  │ Tokyo   │────────▶│ US Server │                  │
│  │ User    │  200ms  │  Renders  │                  │
│  │         │◀────────│   Page    │                  │
│  └─────────┘  200ms  └───────────┘                  │
│  Total: 400ms + render time                         │
└──────────────────────────────────────────────────────┘

Edge Rendering:
┌──────────────────────────────────────────────────────┐
│  User in Tokyo → Tokyo Edge → Response               │
│  Latency: ~10ms base + rendering time                │
├──────────────────────────────────────────────────────┤
│  ┌─────────┐         ┌───────────┐                  │
│  │ Tokyo   │────────▶│Tokyo Edge │                  │
│  │ User    │  10ms   │  Renders  │                  │
│  │         │◀────────│   Page    │                  │
│  └─────────┘  10ms   └───────────┘                  │
│  Total: 20ms + render time                          │
│  20x faster baseline!                               │
└──────────────────────────────────────────────────────┘

Global Edge Network:
        ┌─────────────────────────────────┐
        │       Global CDN Network         │
        ├─────────────────────────────────┤
        │                                  │
        │  🌍 London    🌏 Tokyo          │
        │  🌎 NYC       🌏 Sydney         │
        │  🌎 SF        🌍 Mumbai         │
        │  🌎 Brazil    🌍 Frankfurt      │
        │                                  │
        │  Each edge can render SSR!      │
        └─────────────────────────────────┘
```

## Next.js Edge Runtime

### Basic Edge Function

```typescript
// app/api/hello/route.ts
import { NextResponse } from 'next/server';

export const runtime = 'edge'; // Enable edge runtime

export async function GET(request: Request) {
  return NextResponse.json({
    message: 'Hello from the edge!',
    location: request.headers.get('x-vercel-ip-city') || 'Unknown',
    timestamp: Date.now(),
  });
}
```

### Edge-Rendered Page

```typescript
// app/product/[id]/page.tsx
import { NextResponse } from 'next/server';

// Enable edge runtime for this page
export const runtime = 'edge';

interface PageProps {
  params: { id: string };
}

export default async function ProductPage({ params }: PageProps) {
  // Fetch data at the edge
  const product = await fetchProduct(params.id);

  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <p className="price">${product.price}</p>
      <button>Add to Cart</button>
    </div>
  );
}

async function fetchProduct(id: string) {
  const response = await fetch(`https://api.example.com/products/${id}`, {
    headers: {
      'Content-Type': 'application/json',
    },
  });

  if (!response.ok) {
    throw new Error('Failed to fetch product');
  }

  return response.json();
}
```

### Personalized Edge Rendering

```typescript
// app/dashboard/page.tsx
import { cookies } from 'next/headers';

export const runtime = 'edge';

export default async function Dashboard() {
  // Access user data at the edge
  const cookieStore = cookies();
  const userId = cookieStore.get('userId')?.value;

  if (!userId) {
    return <div>Please log in</div>;
  }

  // Fetch user-specific data
  const userData = await fetchUserData(userId);

  return (
    <div>
      <h1>Welcome, {userData.name}!</h1>
      
      <div className="stats">
        <StatCard title="Views" value={userData.stats.views} />
        <StatCard title="Revenue" value={`$${userData.stats.revenue}`} />
        <StatCard title="Orders" value={userData.stats.orders} />
      </div>

      <RecentActivity activities={userData.recentActivity} />
    </div>
  );
}

async function fetchUserData(userId: string) {
  const response = await fetch(
    `https://api.example.com/users/${userId}/dashboard`,
    {
      headers: {
        'X-API-Key': process.env.API_KEY!,
      },
    }
  );

  return response.json();
}

function StatCard({ title, value }: { title: string; value: string | number }) {
  return (
    <div className="stat-card">
      <h3>{title}</h3>
      <p className="value">{value}</p>
    </div>
  );
}

function RecentActivity({ activities }: { activities: any[] }) {
  return (
    <div className="activity">
      <h2>Recent Activity</h2>
      {activities.map((activity) => (
        <div key={activity.id}>
          <p>{activity.description}</p>
          <time>{new Date(activity.timestamp).toLocaleString()}</time>
        </div>
      ))}
    </div>
  );
}
```

## Cloudflare Workers

### Basic Worker

```typescript
// worker.ts
export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);

    // Route handling
    if (url.pathname === '/api/hello') {
      return new Response(
        JSON.stringify({
          message: 'Hello from Cloudflare Edge',
          location: request.cf?.colo || 'Unknown',
          country: request.cf?.country || 'Unknown',
        }),
        {
          headers: {
            'Content-Type': 'application/json',
          },
        }
      );
    }

    // Default response
    return new Response('Not Found', { status: 404 });
  },
};
```

### SSR with Cloudflare Workers

```typescript
// worker-ssr.ts
import { renderToString } from 'react-dom/server';
import React from 'react';
import App from './App';

interface Env {
  ASSETS: any;
  API_URL: string;
  API_KEY: string;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    // Serve static assets
    if (url.pathname.startsWith('/static/')) {
      return env.ASSETS.fetch(request);
    }

    // SSR for pages
    try {
      const html = await renderPage(request, env);

      return new Response(html, {
        headers: {
          'Content-Type': 'text/html',
          'Cache-Control': 'public, max-age=60',
        },
      });
    } catch (error) {
      return new Response('Error rendering page', { status: 500 });
    }
  },
};

async function renderPage(request: Request, env: Env): Promise<string> {
  // Fetch data at the edge
  const data = await fetch(`${env.API_URL}/data`, {
    headers: {
      'X-API-Key': env.API_KEY,
    },
  }).then((r) => r.json());

  // Render React component
  const appHtml = renderToString(React.createElement(App, { data }));

  // Return complete HTML
  return `
    <!DOCTYPE html>
    <html>
      <head>
        <meta charset="utf-8">
        <title>Edge SSR App</title>
        <link rel="stylesheet" href="/static/styles.css">
      </head>
      <body>
        <div id="root">${appHtml}</div>
        <script>
          window.__INITIAL_DATA__ = ${JSON.stringify(data)};
        </script>
        <script src="/static/client.js"></script>
      </body>
    </html>
  `;
}
```

### Geolocation-Based Rendering

```typescript
// geo-worker.ts
export default {
  async fetch(request: Request): Promise<Response> {
    const cf = request.cf as any;
    const country = cf?.country || 'US';
    const region = cf?.region || 'Unknown';
    const city = cf?.city || 'Unknown';

    // Fetch region-specific content
    const content = await fetchRegionalContent(country, region);

    // Render page with regional content
    const html = renderPage({
      content,
      location: {
        country,
        region,
        city,
      },
    });

    return new Response(html, {
      headers: {
        'Content-Type': 'text/html',
        'Cache-Control': 'public, max-age=300',
      },
    });
  },
};

async function fetchRegionalContent(country: string, region: string) {
  // Fetch content tailored to user's location
  const response = await fetch(
    `https://api.example.com/content?country=${country}&region=${region}`
  );

  return response.json();
}

function renderPage(data: any): string {
  return `
    <!DOCTYPE html>
    <html>
      <head>
        <title>Welcome from ${data.location.city}</title>
      </head>
      <body>
        <h1>Hello from ${data.location.city}, ${data.location.country}!</h1>
        <div>${data.content.html}</div>
      </body>
    </html>
  `;
}
```

## Deno Deploy Edge Functions

### Basic Deno Deploy Function

```typescript
// main.ts
import { serve } from 'https://deno.land/std@0.140.0/http/server.ts';

serve(async (request: Request) => {
  const url = new URL(request.url);

  if (url.pathname === '/api/hello') {
    return new Response(
      JSON.stringify({
        message: 'Hello from Deno Deploy Edge',
        region: Deno.env.get('DENO_REGION') || 'Unknown',
        timestamp: Date.now(),
      }),
      {
        headers: {
          'Content-Type': 'application/json',
        },
      }
    );
  }

  return new Response('Not Found', { status: 404 });
});
```

### React SSR on Deno Deploy

```typescript
// ssr.tsx
import { serve } from 'https://deno.land/std@0.140.0/http/server.ts';
import React from 'https://esm.sh/react@18';
import { renderToString } from 'https://esm.sh/react-dom@18/server';

function App({ data }: { data: any }) {
  return (
    <div>
      <h1>Edge SSR with Deno</h1>
      <p>Rendered at: {new Date().toISOString()}</p>
      <pre>{JSON.stringify(data, null, 2)}</pre>
    </div>
  );
}

serve(async (request: Request) => {
  // Fetch data
  const data = await fetch('https://api.example.com/data')
    .then((r) => r.json())
    .catch(() => ({ error: 'Failed to fetch data' }));

  // Render React component
  const html = renderToString(<App data={data} />);

  // Return HTML
  return new Response(
    `
    <!DOCTYPE html>
    <html>
      <head>
        <meta charset="utf-8">
        <title>Deno Edge SSR</title>
      </head>
      <body>
        <div id="root">${html}</div>
      </body>
    </html>
  `,
    {
      headers: {
        'Content-Type': 'text/html',
      },
    }
  );
});
```

## Edge Middleware

### Next.js Edge Middleware

```typescript
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const response = NextResponse.next();

  // Get user's location
  const country = request.geo?.country || 'US';
  const city = request.geo?.city || 'Unknown';

  // Add location headers
  response.headers.set('x-user-country', country);
  response.headers.set('x-user-city', city);

  // Redirect based on location
  if (country === 'CN' && request.nextUrl.pathname === '/') {
    return NextResponse.redirect(new URL('/cn', request.url));
  }

  // A/B testing at the edge
  const variant = Math.random() > 0.5 ? 'a' : 'b';
  response.cookies.set('ab-test-variant', variant);

  // Rate limiting
  const ip = request.ip || 'unknown';
  const rateLimitResult = checkRateLimit(ip);

  if (!rateLimitResult.allowed) {
    return new NextResponse('Too Many Requests', { status: 429 });
  }

  return response;
}

export const config = {
  matcher: [
    /*
     * Match all request paths except:
     * - api (API routes)
     * - _next/static (static files)
     * - _next/image (image optimization files)
     * - favicon.ico (favicon file)
     */
    '/((?!api|_next/static|_next/image|favicon.ico).*)',
  ],
};

function checkRateLimit(ip: string): { allowed: boolean } {
  // Simplified rate limiting
  // In production, use KV storage or similar
  return { allowed: true };
}
```

### Authentication at the Edge

```typescript
// middleware-auth.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';
import { jwtVerify } from 'jose';

const SECRET = new TextEncoder().encode(process.env.JWT_SECRET);

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // Public routes
  if (pathname.startsWith('/login') || pathname.startsWith('/public')) {
    return NextResponse.next();
  }

  // Get token from cookie
  const token = request.cookies.get('auth-token')?.value;

  if (!token) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  try {
    // Verify JWT at the edge
    const { payload } = await jwtVerify(token, SECRET);

    // Add user info to request headers
    const response = NextResponse.next();
    response.headers.set('x-user-id', payload.userId as string);
    response.headers.set('x-user-role', payload.role as string);

    return response;
  } catch (error) {
    // Invalid token, redirect to login
    return NextResponse.redirect(new URL('/login', request.url));
  }
}
```

## Edge Caching Strategies

### Cache Configuration

```typescript
// lib/edge-cache.ts
export const cacheConfig = {
  // Static assets - long cache
  static: {
    'Cache-Control': 'public, max-age=31536000, immutable',
  },

  // API responses - short cache
  api: {
    'Cache-Control': 'public, max-age=60, s-maxage=300',
  },

  // User-specific - no cache
  private: {
    'Cache-Control': 'private, no-cache, no-store, must-revalidate',
  },

  // Stale-while-revalidate
  swr: {
    'Cache-Control': 'public, max-age=60, stale-while-revalidate=3600',
  },
};

// Usage in edge function
export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);

    if (url.pathname.startsWith('/api/')) {
      const data = await fetchData();
      return new Response(JSON.stringify(data), {
        headers: {
          'Content-Type': 'application/json',
          ...cacheConfig.api,
        },
      });
    }

    // ... other routes
  },
};
```

### Cache Tags and Purging

```typescript
// edge-cache-tags.ts
export default {
  async fetch(request: Request, env: any): Promise<Response> {
    const url = new URL(request.url);

    if (url.pathname.startsWith('/products/')) {
      const productId = url.pathname.split('/')[2];
      const product = await fetchProduct(productId);

      return new Response(JSON.stringify(product), {
        headers: {
          'Content-Type': 'application/json',
          'Cache-Control': 'public, max-age=3600',
          'Cache-Tag': `product-${productId},products`,
        },
      });
    }

    // Purge cache endpoint
    if (url.pathname === '/api/purge' && request.method === 'POST') {
      const { tags } = await request.json();

      // Purge cache by tags
      await purgeCacheTags(tags, env);

      return new Response('Cache purged', { status: 200 });
    }

    return new Response('Not Found', { status: 404 });
  },
};

async function purgeCacheTags(tags: string[], env: any): Promise<void> {
  // Implementation depends on CDN provider
  // Example: Cloudflare cache purge
  await fetch(`https://api.cloudflare.com/client/v4/zones/${env.ZONE_ID}/purge_cache`, {
    method: 'POST',
    headers: {
      'X-Auth-Email': env.CF_EMAIL,
      'X-Auth-Key': env.CF_API_KEY,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ tags }),
  });
}
```

## Edge Database Integration

### Edge KV Storage

```typescript
// kv-storage.ts
interface Env {
  KV: KVNamespace;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    // Read from KV
    if (url.pathname === '/api/config') {
      const config = await env.KV.get('app-config', 'json');

      return new Response(JSON.stringify(config), {
        headers: { 'Content-Type': 'application/json' },
      });
    }

    // Write to KV
    if (url.pathname === '/api/config' && request.method === 'POST') {
      const config = await request.json();
      await env.KV.put('app-config', JSON.stringify(config));

      return new Response('Config saved', { status: 200 });
    }

    // Counter example
    if (url.pathname === '/api/counter') {
      const count = await env.KV.get('counter');
      const newCount = (parseInt(count || '0') + 1).toString();
      await env.KV.put('counter', newCount);

      return new Response(JSON.stringify({ count: newCount }), {
        headers: { 'Content-Type': 'application/json' },
      });
    }

    return new Response('Not Found', { status: 404 });
  },
};
```

### Edge Database Queries

```typescript
// edge-db.ts
import { neon } from '@neondatabase/serverless';

interface Env {
  DATABASE_URL: string;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const sql = neon(env.DATABASE_URL);
    const url = new URL(request.url);

    if (url.pathname === '/api/users') {
      const users = await sql`SELECT id, name, email FROM users LIMIT 10`;

      return new Response(JSON.stringify(users), {
        headers: { 'Content-Type': 'application/json' },
      });
    }

    if (url.pathname.startsWith('/api/users/')) {
      const userId = url.pathname.split('/')[3];
      const [user] = await sql`
        SELECT id, name, email, created_at
        FROM users
        WHERE id = ${userId}
      `;

      if (!user) {
        return new Response('User not found', { status: 404 });
      }

      return new Response(JSON.stringify(user), {
        headers: { 'Content-Type': 'application/json' },
      });
    }

    return new Response('Not Found', { status: 404 });
  },
};
```

## Performance Monitoring

### Edge Metrics

```typescript
// edge-metrics.ts
interface Metrics {
  responseTime: number;
  region: string;
  cacheHit: boolean;
  timestamp: number;
}

export default {
  async fetch(request: Request, env: any): Promise<Response> {
    const startTime = Date.now();
    const region = env.REGION || 'unknown';

    try {
      // Process request
      const response = await handleRequest(request, env);

      // Record metrics
      const metrics: Metrics = {
        responseTime: Date.now() - startTime,
        region,
        cacheHit: response.headers.get('cf-cache-status') === 'HIT',
        timestamp: Date.now(),
      };

      // Send to analytics
      await recordMetrics(metrics, env);

      // Add timing header
      response.headers.set('X-Response-Time', `${metrics.responseTime}ms`);
      response.headers.set('X-Edge-Region', region);

      return response;
    } catch (error) {
      // Record error
      await recordError(error, region, env);

      return new Response('Internal Server Error', { status: 500 });
    }
  },
};

async function recordMetrics(metrics: Metrics, env: any): Promise<void> {
  // Send to analytics service
  if (env.ANALYTICS_ENDPOINT) {
    fetch(env.ANALYTICS_ENDPOINT, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(metrics),
    }).catch(() => {
      // Ignore analytics failures
    });
  }
}

async function recordError(error: any, region: string, env: any): Promise<void> {
  const errorData = {
    message: error.message,
    stack: error.stack,
    region,
    timestamp: Date.now(),
  };

  if (env.ERROR_TRACKING_ENDPOINT) {
    fetch(env.ERROR_TRACKING_ENDPOINT, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(errorData),
    }).catch(() => {
      // Ignore error tracking failures
    });
  }
}

async function handleRequest(request: Request, env: any): Promise<Response> {
  return new Response('OK');
}
```

## Common Mistakes

### 1. Using Node.js APIs

```typescript
// BAD: Node.js APIs don't work at the edge
import fs from 'fs';
import path from 'path';

export const runtime = 'edge';

export async function GET() {
  // ❌ This will fail!
  const data = fs.readFileSync('data.json');
  return Response.json(data);
}

// GOOD: Use Web APIs
export const runtime = 'edge';

export async function GET() {
  // ✓ Fetch from external source
  const data = await fetch('https://api.example.com/data').then(r => r.json());
  return Response.json(data);
}
```

### 2. Heavy Computation at Edge

```typescript
// BAD: CPU-intensive work at edge
export const runtime = 'edge';

export async function GET() {
  // ❌ This will timeout!
  const result = heavyComputation(); // Takes 10 seconds
  return Response.json(result);
}

// GOOD: Keep edge functions fast
export const runtime = 'edge';

export async function GET() {
  // ✓ Fetch pre-computed results
  const result = await fetch('https://api.example.com/computed').then(r => r.json());
  return Response.json(result);
}
```

### 3. Not Handling Errors

```typescript
// BAD: No error handling
export default {
  async fetch(request: Request): Promise<Response> {
    const data = await fetch('https://api.example.com/data').then(r => r.json());
    return Response.json(data);
  },
};

// GOOD: Proper error handling
export default {
  async fetch(request: Request): Promise<Response> {
    try {
      const data = await fetch('https://api.example.com/data').then(r => r.json());
      return Response.json(data);
    } catch (error) {
      console.error('Edge function error:', error);
      return new Response('Internal Server Error', { status: 500 });
    }
  },
};
```

## Best Practices

### 1. Keep Functions Fast

```typescript
// Edge functions should complete in < 50ms
export const runtime = 'edge';

export async function GET(request: Request) {
  const startTime = Date.now();

  // Fast operations only
  const data = await fetch('https://api.example.com/data', {
    signal: AbortSignal.timeout(5000), // 5s timeout
  }).then(r => r.json());

  const duration = Date.now() - startTime;

  if (duration > 50) {
    console.warn(`Slow edge function: ${duration}ms`);
  }

  return Response.json(data);
}
```

### 2. Implement Proper Caching

```typescript
export default {
  async fetch(request: Request, env: any): Promise<Response> {
    const url = new URL(request.url);
    const cacheKey = new Request(url.toString(), request);
    const cache = caches.default;

    // Try cache first
    let response = await cache.match(cacheKey);

    if (!response) {
      // Generate response
      response = await generateResponse(request, env);

      // Cache it
      await cache.put(cacheKey, response.clone());
    }

    return response;
  },
};
```

### 3. Use Appropriate Edge Regions

```typescript
// Deploy to regions close to your data sources
// and user base

// Example: Regional configuration
const config = {
  regions: [
    'iad1', // US East (Ashburn)
    'sfo1', // US West (San Francisco)
    'fra1', // Europe (Frankfurt)
    'sin1', // Asia (Singapore)
  ],
  dataSource: {
    primary: 'us-east-1',
    replicas: ['eu-west-1', 'ap-southeast-1'],
  },
};
```

## When to Use Edge Rendering

### Ideal Use Cases

1. **Global applications** - Users worldwide
2. **Personalized content** - User-specific rendering
3. **A/B testing** - Split traffic at edge
4. **Geolocation** - Location-based content
5. **Authentication** - Fast auth checks
6. **API gateways** - Route/transform requests

### Not Recommended For

1. **Heavy computation** - CPU-intensive tasks
2. **Large bundles** - Limited memory/CPU
3. **File system operations** - No FS access
4. **Long-running tasks** - Execution time limits
5. **Complex databases** - Better on origin server

## Interview Questions

1. **What is edge rendering?**
   - SSR at CDN edge locations
   - Closer to users geographically
   - Lower latency than origin SSR

2. **How does it differ from traditional SSR?**
   - Traditional: single origin server
   - Edge: distributed across globe
   - Edge: uses Web APIs, not Node.js

3. **What are the benefits?**
   - Lower latency
   - Global scalability
   - Better user experience
   - Reduced origin load

4. **What are the limitations?**
   - Execution time limits (50-500ms)
   - Memory constraints
   - No Node.js APIs
   - No file system access

5. **Which platforms support edge rendering?**
   - Vercel Edge Functions
   - Cloudflare Workers
   - Deno Deploy
   - AWS Lambda@Edge
   - Netlify Edge Functions

## Key Takeaways

1. Edge rendering brings SSR closer to users
2. Dramatically reduces latency globally
3. Use Web APIs instead of Node.js APIs
4. Keep edge functions fast (< 50ms)
5. Implement proper caching strategies
6. Handle errors gracefully
7. Great for personalization and geo-targeting
8. Not suitable for heavy computation
9. Monitor performance across regions
10. Choose appropriate edge platform

## Resources

- [Vercel Edge Functions](https://vercel.com/docs/concepts/functions/edge-functions)
- [Cloudflare Workers](https://developers.cloudflare.com/workers/)
- [Deno Deploy](https://deno.com/deploy)
- [Next.js Edge Runtime](https://nextjs.org/docs/api-reference/edge-runtime)
- [AWS Lambda@Edge](https://aws.amazon.com/lambda/edge/)
