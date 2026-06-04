# Caching Strategies

## Overview

Caching is one of the most effective performance optimization techniques, reducing server load, bandwidth usage, and page load times by storing and reusing previously fetched resources. A well-designed caching strategy can dramatically improve application performance, especially for repeat visitors. Modern web applications use multiple caching layers including browser cache, service worker cache, CDN cache, and application-level caching.

## Caching Layers

```
Web Caching Architecture
=========================

┌─────────────────────────────────────────────────┐
│                                                 │
│  User Browser                                   │
│  ┌───────────────────────────────────────────┐ │
│  │  1. Memory Cache (RAM)                    │ │
│  │     - Fastest, cleared on tab close       │ │
│  │     - Images, scripts from current page   │ │
│  └───────────────────────────────────────────┘ │
│  ┌───────────────────────────────────────────┐ │
│  │  2. Disk Cache (Browser Cache)            │ │
│  │     - Persistent across sessions          │ │
│  │     - Controlled by Cache-Control headers │ │
│  └───────────────────────────────────────────┘ │
│  ┌───────────────────────────────────────────┐ │
│  │  3. Service Worker Cache                  │ │
│  │     - Programmable, offline support       │ │
│  │     - Cache API for custom strategies     │ │
│  └───────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│  4. CDN Cache (Edge Servers)                    │
│     - Geographically distributed              │
│     - Reduces latency                         │
│     - Shared across users                     │
└──────────────────┬──────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────┐
│  5. Origin Server Cache                         │
│     - Redis, Memcached                        │
│     - Database query cache                    │
│     - Application-level cache                 │
└─────────────────────────────────────────────────┘
```

## HTTP Cache Headers

```javascript
// Express.js - Setting Cache-Control headers
const express = require('express');
const app = express();

// Static assets with long cache (immutable)
app.use('/static', express.static('public', {
  maxAge: '365d', // 1 year
  immutable: true,
  setHeaders: (res, path) => {
    if (path.endsWith('.html')) {
      // HTML files - no cache
      res.setHeader('Cache-Control', 'no-cache, no-store, must-revalidate');
    } else if (path.match(/\.(js|css)$/)) {
      // Versioned JS/CSS - long cache
      res.setHeader('Cache-Control', 'public, max-age=31536000, immutable');
    } else if (path.match(/\.(jpg|jpeg|png|gif|webp)$/)) {
      // Images - moderate cache
      res.setHeader('Cache-Control', 'public, max-age=2592000'); // 30 days
    }
  }
}));

// API responses with stale-while-revalidate
app.get('/api/data', (req, res) => {
  res.set({
    'Cache-Control': 'public, max-age=60, stale-while-revalidate=300',
    'ETag': generateETag(data),
    'Last-Modified': lastModified.toUTCString()
  });
  res.json(data);
});

// No cache for sensitive data
app.get('/api/user/profile', (req, res) => {
  res.set({
    'Cache-Control': 'private, no-cache, no-store, must-revalidate',
    'Pragma': 'no-cache',
    'Expires': '0'
  });
  res.json(userData);
});

// Cache with ETag validation
app.get('/api/article/:id', (req, res) => {
  const article = getArticle(req.params.id);
  const etag = generateETag(article);
  
  // Check If-None-Match header
  if (req.headers['if-none-match'] === etag) {
    return res.status(304).end(); // Not Modified
  }
  
  res.set({
    'Cache-Control': 'public, max-age=300', // 5 minutes
    'ETag': etag
  });
  res.json(article);
});

// Helper function to generate ETag
function generateETag(data) {
  const crypto = require('crypto');
  return crypto
    .createHash('md5')
    .update(JSON.stringify(data))
    .digest('hex');
}

// Next.js - Setting cache headers
// next.config.js
module.exports = {
  async headers() {
    return [
      {
        source: '/_next/static/:path*',
        headers: [
          {
            key: 'Cache-Control',
            value: 'public, max-age=31536000, immutable'
          }
        ]
      },
      {
        source: '/images/:path*',
        headers: [
          {
            key: 'Cache-Control',
            value: 'public, max-age=2592000' // 30 days
          }
        ]
      },
      {
        source: '/api/:path*',
        headers: [
          {
            key: 'Cache-Control',
            value: 'public, max-age=60, stale-while-revalidate=300'
          }
        ]
      }
    ];
  }
};
```

## Service Worker Caching Strategies

```typescript
// service-worker.ts - Comprehensive caching strategies
const CACHE_VERSION = 'v1';
const STATIC_CACHE = `static-${CACHE_VERSION}`;
const DYNAMIC_CACHE = `dynamic-${CACHE_VERSION}`;
const IMAGE_CACHE = `images-${CACHE_VERSION}`;

// Assets to cache on install
const STATIC_ASSETS = [
  '/',
  '/index.html',
  '/styles/main.css',
  '/scripts/app.js',
  '/manifest.json',
  '/offline.html'
];

// Install event - cache static assets
self.addEventListener('install', (event: ExtendableEvent) => {
  event.waitUntil(
    caches.open(STATIC_CACHE).then(cache => {
      console.log('Caching static assets');
      return cache.addAll(STATIC_ASSETS);
    })
  );
  // Force activation
  self.skipWaiting();
});

// Activate event - clean old caches
self.addEventListener('activate', (event: ExtendableEvent) => {
  event.waitUntil(
    caches.keys().then(cacheNames => {
      return Promise.all(
        cacheNames
          .filter(cacheName => {
            return cacheName.startsWith('static-') ||
                   cacheName.startsWith('dynamic-') ||
                   cacheName.startsWith('images-');
          })
          .filter(cacheName => {
            return cacheName !== STATIC_CACHE &&
                   cacheName !== DYNAMIC_CACHE &&
                   cacheName !== IMAGE_CACHE;
          })
          .map(cacheName => caches.delete(cacheName))
      );
    })
  );
  return self.clients.claim();
});

// Fetch event - apply caching strategies
self.addEventListener('fetch', (event: FetchEvent) => {
  const { request } = event;
  const url = new URL(request.url);

  // Strategy 1: Cache-first for images
  if (request.destination === 'image') {
    event.respondWith(cacheFirstStrategy(request, IMAGE_CACHE));
    return;
  }

  // Strategy 2: Network-first for API calls
  if (url.pathname.startsWith('/api/')) {
    event.respondWith(networkFirstStrategy(request, DYNAMIC_CACHE));
    return;
  }

  // Strategy 3: Stale-while-revalidate for pages
  if (request.mode === 'navigate') {
    event.respondWith(staleWhileRevalidateStrategy(request, DYNAMIC_CACHE));
    return;
  }

  // Strategy 4: Cache-first for static assets
  event.respondWith(cacheFirstStrategy(request, STATIC_CACHE));
});

// Cache-first strategy
async function cacheFirstStrategy(
  request: Request,
  cacheName: string
): Promise<Response> {
  const cachedResponse = await caches.match(request);
  if (cachedResponse) {
    return cachedResponse;
  }

  try {
    const networkResponse = await fetch(request);
    if (networkResponse.ok) {
      const cache = await caches.open(cacheName);
      cache.put(request, networkResponse.clone());
    }
    return networkResponse;
  } catch (error) {
    // Return offline page for navigation requests
    if (request.mode === 'navigate') {
      return caches.match('/offline.html');
    }
    throw error;
  }
}

// Network-first strategy
async function networkFirstStrategy(
  request: Request,
  cacheName: string
): Promise<Response> {
  try {
    const networkResponse = await fetch(request);
    if (networkResponse.ok) {
      const cache = await caches.open(cacheName);
      cache.put(request, networkResponse.clone());
    }
    return networkResponse;
  } catch (error) {
    const cachedResponse = await caches.match(request);
    if (cachedResponse) {
      return cachedResponse;
    }
    throw error;
  }
}

// Stale-while-revalidate strategy
async function staleWhileRevalidateStrategy(
  request: Request,
  cacheName: string
): Promise<Response> {
  const cachedResponse = await caches.match(request);
  
  const fetchPromise = fetch(request).then(networkResponse => {
    if (networkResponse.ok) {
      const cache = caches.open(cacheName);
      cache.then(c => c.put(request, networkResponse.clone()));
    }
    return networkResponse;
  });

  return cachedResponse || fetchPromise;
}

// Cache size management
async function trimCache(cacheName: string, maxItems: number) {
  const cache = await caches.open(cacheName);
  const keys = await cache.keys();
  
  if (keys.length > maxItems) {
    await cache.delete(keys[0]);
    trimCache(cacheName, maxItems); // Recursive call
  }
}

// Periodic cache cleanup (called from page)
self.addEventListener('message', (event: ExtendableMessageEvent) => {
  if (event.data.action === 'trimCaches') {
    event.waitUntil(
      Promise.all([
        trimCache(IMAGE_CACHE, 50),
        trimCache(DYNAMIC_CACHE, 30)
      ])
    );
  }
});
```

**React: Service Worker Registration**

```typescript
// serviceWorkerRegistration.ts
export function register() {
  if ('serviceWorker' in navigator) {
    window.addEventListener('load', () => {
      navigator.serviceWorker
        .register('/service-worker.js')
        .then(registration => {
          console.log('SW registered:', registration);
          
          // Check for updates periodically
          setInterval(() => {
            registration.update();
          }, 60000); // Check every minute
          
          // Handle updates
          registration.addEventListener('updatefound', () => {
            const newWorker = registration.installing;
            if (newWorker) {
              newWorker.addEventListener('statechange', () => {
                if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
                  // New version available
                  showUpdateNotification();
                }
              });
            }
          });
        })
        .catch(error => {
          console.error('SW registration failed:', error);
        });
    });
  }
}

export function unregister() {
  if ('serviceWorker' in navigator) {
    navigator.serviceWorker.ready
      .then(registration => {
        registration.unregister();
      })
      .catch(error => {
        console.error(error.message);
      });
  }
}

// React component for update notification
function UpdateNotification() {
  const [showUpdate, setShowUpdate] = useState(false);

  useEffect(() => {
    window.addEventListener('sw-update-available', () => {
      setShowUpdate(true);
    });
  }, []);

  const handleUpdate = () => {
    if ('serviceWorker' in navigator) {
      navigator.serviceWorker.ready.then(registration => {
        registration.waiting?.postMessage({ type: 'SKIP_WAITING' });
        window.location.reload();
      });
    }
  };

  if (!showUpdate) return null;

  return (
    <div className="update-notification">
      <p>A new version is available!</p>
      <button onClick={handleUpdate}>Update Now</button>
    </div>
  );
}

function showUpdateNotification() {
  window.dispatchEvent(new Event('sw-update-available'));
}
```

**Angular: Service Worker**

```typescript
// app.module.ts
import { ServiceWorkerModule } from '@angular/service-worker';
import { environment } from '../environments/environment';

@NgModule({
  imports: [
    ServiceWorkerModule.register('ngsw-worker.js', {
      enabled: environment.production,
      registrationStrategy: 'registerWhenStable:30000'
    })
  ]
})
export class AppModule {}

// ngsw-config.json - Angular service worker configuration
{
  "index": "/index.html",
  "assetGroups": [
    {
      "name": "app",
      "installMode": "prefetch",
      "resources": {
        "files": [
          "/favicon.ico",
          "/index.html",
          "/*.css",
          "/*.js"
        ]
      }
    },
    {
      "name": "assets",
      "installMode": "lazy",
      "updateMode": "prefetch",
      "resources": {
        "files": [
          "/assets/**",
          "/*.(eot|svg|cur|jpg|png|webp|gif|otf|ttf|woff|woff2|ani)"
        ]
      }
    }
  ],
  "dataGroups": [
    {
      "name": "api-cache",
      "urls": ["/api/**"],
      "cacheConfig": {
        "strategy": "freshness",
        "maxSize": 100,
        "maxAge": "1h",
        "timeout": "5s"
      }
    },
    {
      "name": "api-performance",
      "urls": ["/api/fast/**"],
      "cacheConfig": {
        "strategy": "performance",
        "maxSize": 100,
        "maxAge": "1d"
      }
    }
  ]
}

// update.service.ts - Handle updates
import { Injectable } from '@angular/core';
import { SwUpdate } from '@angular/service-worker';

@Injectable({
  providedIn: 'root'
})
export class UpdateService {
  constructor(private swUpdate: SwUpdate) {
    if (this.swUpdate.isEnabled) {
      // Check for updates periodically
      setInterval(() => {
        this.swUpdate.checkForUpdate();
      }, 60000);

      // Handle available updates
      this.swUpdate.versionUpdates.subscribe(event => {
        if (event.type === 'VERSION_READY') {
          if (confirm('New version available. Update now?')) {
            this.swUpdate.activateUpdate().then(() => {
              window.location.reload();
            });
          }
        }
      });
    }
  }
}
```

## Cache Invalidation Strategies

```typescript
// React: Cache invalidation with versioning
import { useState, useEffect } from 'react';

interface CacheConfig {
  version: string;
  maxAge: number;
}

function useCachedData<T>(
  key: string,
  fetcher: () => Promise<T>,
  config: CacheConfig
) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const fetchData = async () => {
      try {
        // Check cache first
        const cached = localStorage.getItem(key);
        if (cached) {
          const { data, timestamp, version } = JSON.parse(cached);
          const age = Date.now() - timestamp;
          
          // Use cache if valid
          if (age < config.maxAge && version === config.version) {
            setData(data);
            setLoading(false);
            return;
          }
        }

        // Fetch fresh data
        const freshData = await fetcher();
        
        // Update cache
        localStorage.setItem(
          key,
          JSON.stringify({
            data: freshData,
            timestamp: Date.now(),
            version: config.version
          })
        );
        
        setData(freshData);
      } catch (error) {
        console.error('Failed to fetch data:', error);
      } finally {
        setLoading(false);
      }
    };

    fetchData();
  }, [key, config.version, config.maxAge]);

  const invalidate = () => {
    localStorage.removeItem(key);
    setLoading(true);
  };

  return { data, loading, invalidate };
}

// Usage
function ProductList() {
  const { data: products, loading, invalidate } = useCachedData(
    'products',
    () => fetch('/api/products').then(res => res.json()),
    { version: '1.0', maxAge: 300000 } // 5 minutes
  );

  return (
    <div>
      <button onClick={invalidate}>Refresh</button>
      {loading ? <Spinner /> : <ProductGrid products={products} />}
    </div>
  );
}

// Stale-while-revalidate pattern
function useStaleWhileRevalidate<T>(
  key: string,
  fetcher: () => Promise<T>,
  maxAge: number
) {
  const [data, setData] = useState<T | null>(null);
  const [isStale, setIsStale] = useState(false);

  useEffect(() => {
    const fetchData = async () => {
      // Get cached data
      const cached = localStorage.getItem(key);
      if (cached) {
        const { data: cachedData, timestamp } = JSON.parse(cached);
        setData(cachedData);
        
        // Check if stale
        const age = Date.now() - timestamp;
        if (age > maxAge) {
          setIsStale(true);
          // Revalidate in background
          try {
            const freshData = await fetcher();
            setData(freshData);
            setIsStale(false);
            localStorage.setItem(
              key,
              JSON.stringify({ data: freshData, timestamp: Date.now() })
            );
          } catch (error) {
            console.error('Revalidation failed:', error);
          }
        }
      } else {
        // No cache, fetch fresh
        const freshData = await fetcher();
        setData(freshData);
        localStorage.setItem(
          key,
          JSON.stringify({ data: freshData, timestamp: Date.now() })
        );
      }
    };

    fetchData();
  }, [key, maxAge]);

  return { data, isStale };
}

// Cache with dependencies
function useCachedDataWithDeps<T>(
  key: string,
  fetcher: () => Promise<T>,
  dependencies: string[]
) {
  const [data, setData] = useState<T | null>(null);

  useEffect(() => {
    const fetchData = async () => {
      const cacheKey = `${key}-${dependencies.join('-')}`;
      const cached = localStorage.getItem(cacheKey);
      
      if (cached) {
        setData(JSON.parse(cached));
        return;
      }

      const freshData = await fetcher();
      localStorage.setItem(cacheKey, JSON.stringify(freshData));
      setData(freshData);
    };

    fetchData();
  }, [key, ...dependencies]);

  return data;
}
```

## CDN Caching

```javascript
// Cloudflare Worker - Custom caching logic
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  const url = new URL(request.url);
  const cache = caches.default;

  // Check cache first
  let response = await cache.match(request);
  if (response) {
    return response;
  }

  // Fetch from origin
  response = await fetch(request);

  // Cache based on path
  if (url.pathname.startsWith('/api/')) {
    // API responses - short cache
    const headers = new Headers(response.headers);
    headers.set('Cache-Control', 'public, max-age=60');
    response = new Response(response.body, {
      status: response.status,
      statusText: response.statusText,
      headers
    });
  } else if (url.pathname.match(/\.(js|css|jpg|png)$/)) {
    // Static assets - long cache
    const headers = new Headers(response.headers);
    headers.set('Cache-Control', 'public, max-age=31536000, immutable');
    response = new Response(response.body, {
      status: response.status,
      statusText: response.statusText,
      headers
    });
  }

  // Store in cache
  if (response.ok) {
    event.waitUntil(cache.put(request, response.clone()));
  }

  return response;
}

// Purge CDN cache programmatically
async function purgeCDNCache(urls) {
  const response = await fetch('https://api.cloudflare.com/client/v4/zones/ZONE_ID/purge_cache', {
    method: 'POST',
    headers: {
      'X-Auth-Email': 'your-email@example.com',
      'X-Auth-Key': 'your-api-key',
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ files: urls })
  });

  return response.json();
}

// Usage
await purgeCDNCache([
  'https://example.com/api/products',
  'https://example.com/styles/main.css'
]);
```

## Caching Strategy Decision Tree

```
Caching Strategy Selection
===========================

         What type of content?
                 │
      ┌──────────┼──────────┐
      │          │           │
   Static     Dynamic      API
   Assets      Pages     Responses
      │          │           │
      ▼          ▼           ▼
  
Static Assets:
├─ Versioned (hash in name)?
│  ├─ Yes: Cache-first, 1 year, immutable
│  └─ No: Cache-first, short TTL
│
Dynamic Pages:
├─ User-specific?
│  ├─ Yes: Private cache, no-store
│  └─ No: Stale-while-revalidate
│
API Responses:
├─ Update frequency?
│  ├─ Frequent: Network-first, short cache
│  ├─ Moderate: Stale-while-revalidate
│  └─ Rare: Cache-first, long TTL
```

## Common Mistakes

1. **Caching user-specific data publicly**
   - Problem: Data leaks to other users
   - Solution: Use `private` cache or no cache

2. **No cache busting strategy**
   - Problem: Users see old versions
   - Solution: Use content hashing in filenames

3. **Overly aggressive caching**
   - Problem: Stale content
   - Solution: Set appropriate TTLs

4. **Not using immutable flag**
   - Problem: Revalidation requests for unchanged assets
   - Solution: Add `immutable` to versioned assets

5. **Caching error responses**
   - Problem: Errors persist
   - Solution: Only cache successful responses

6. **No cache invalidation mechanism**
   - Problem: Can't force updates
   - Solution: Implement cache purging

7. **Ignoring Vary header**
   - Problem: Wrong cached version served
   - Solution: Set Vary header for negotiated content

8. **Not monitoring cache hit rates**
   - Problem: Unknown cache effectiveness
   - Solution: Log and analyze cache metrics

## Best Practices

1. **Use content hashing**: Version assets with hash in filename
2. **Set appropriate TTLs**: Balance freshness vs performance
3. **Implement stale-while-revalidate**: Best of both worlds
4. **Use CDN caching**: Reduce server load and latency
5. **Cache at multiple layers**: Browser, CDN, server
6. **Monitor cache metrics**: Track hit rates and effectiveness
7. **Implement cache invalidation**: Purge when content changes
8. **Test cache behavior**: Verify caching in production
9. **Use service workers**: Offline support and custom strategies
10. **Document caching strategy**: Clear guidelines for team

## When to Use

**Implement caching for:**
- Static assets (JS, CSS, images)
- API responses with low update frequency
- User-generated content
- Third-party resources
- Computed/expensive data

**Avoid caching:**
- Sensitive user data
- Real-time data
- Personalized content (use private cache)
- Authentication tokens
- Frequently changing data

## Interview Questions

1. **Explain the difference between Cache-Control and ETag.**
   - Cache-Control: Sets caching behavior and duration
   - ETag: Validation token for conditional requests
   - Cache-Control prevents requests, ETag enables 304 responses

2. **What is stale-while-revalidate and when to use it?**
   - Serves stale cache while fetching fresh data in background
   - Immediate response + eventual freshness
   - Good for content that changes occasionally
   - Trade-off: Possible stale data

3. **How do service workers differ from HTTP caching?**
   - SW: Programmable, offline support, custom strategies
   - HTTP: Declarative, no offline, follows standards
   - SW intercepts all requests, more control
   - HTTP cache is simpler, works everywhere

4. **Explain cache-first vs network-first strategies.**
   - Cache-first: Check cache, fallback to network (fast, possibly stale)
   - Network-first: Try network, fallback to cache (fresh, slower)
   - Choose based on content type and update frequency

5. **How would you implement cache invalidation?**
   - Version-based: Change URLs/filenames
   - Time-based: Set max-age, let expire
   - Event-based: Purge on updates
   - Tag-based: Group related resources
   - CDN purge API for manual invalidation

6. **What's the purpose of the immutable directive?**
   - Tells browser asset will never change
   - Prevents revalidation requests
   - Only use with content-hashed filenames
   - Improves performance for repeat visits

7. **How do you handle caching for authenticated users?**
   - Use `private` cache-control
   - Or `no-cache` for sensitive data
   - Include Vary: Cookie header
   - Consider short TTLs
   - Be careful with CDN caching

8. **Explain the difference between max-age and s-maxage.**
   - max-age: Browser and proxy cache duration
   - s-maxage: Shared/CDN cache duration only
   - s-maxage overrides max-age for proxies
   - Use s-maxage for CDN-specific rules

## Key Takeaways

1. Caching improves performance by storing and reusing previously fetched resources
2. Multiple caching layers: browser cache, service worker, CDN, server
3. Cache-Control header controls caching behavior and duration
4. Stale-while-revalidate provides immediate response with background updates
5. Service workers enable offline support and custom caching strategies
6. Content hashing enables aggressive caching without staleness
7. Use cache-first for static assets, network-first for dynamic content
8. Implement cache invalidation mechanism for forced updates
9. Monitor cache hit rates to measure effectiveness
10. Balance freshness vs performance based on content type

## Resources

- **Official Documentation**
  - [HTTP Caching MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)
  - [Cache-Control Header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control)
  - [Service Worker API](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API)

- **Tools**
  - [Workbox](https://developers.google.com/web/tools/workbox) - Service worker library
  - [sw-precache](https://github.com/GoogleChromeLabs/sw-precache)
  - [Cache API](https://developer.mozilla.org/en-US/docs/Web/API/Cache)

- **Articles**
  - [HTTP Caching Guide](https://web.dev/http-cache/)
  - [Service Worker Caching Strategies](https://web.dev/offline-cookbook/)
  - [Stale-While-Revalidate](https://web.dev/stale-while-revalidate/)
