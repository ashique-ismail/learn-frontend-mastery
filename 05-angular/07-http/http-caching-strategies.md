# HTTP Caching Strategies in Angular

> Senior-level reference — patterns, tradeoffs, and pitfalls that distinguish candidates who have shipped production caching from those who have only read about it.

---

## Table of Contents

1. [In-Memory Caching with Services and shareReplay(1)](#1-in-memory-caching-with-services-and-sharereplay1)
2. [HTTP Cache-Control Headers and Browser Caching](#2-http-cache-control-headers-and-browser-caching)
3. [ETag and Conditional Requests with HttpClient](#3-etag-and-conditional-requests-with-httpclient)
4. [Implementing a Cache Interceptor](#4-implementing-a-cache-interceptor)
5. [Cache Invalidation Strategies](#5-cache-invalidation-strategies)
6. [Stale-While-Revalidate Pattern in Angular](#6-stale-while-revalidate-pattern-in-angular)
7. [TanStack Query vs Manual Caching in Angular](#7-tanstack-query-vs-manual-caching-in-angular)
8. [Common Pitfalls](#8-common-pitfalls)
9. [Interview Q&A](#9-interview-qa)

---

## 1. In-Memory Caching with Services and shareReplay(1)

### The Core Idea

Angular services are singletons within their injector scope. This makes them the natural place to hold shared, cached state. The RxJS `shareReplay` operator multicasts a source observable and replays the last N emissions to late subscribers — which is exactly what a cache needs.

### Basic Pattern

```typescript
// products.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, shareReplay } from 'rxjs';
import { Product } from './product.model';

@Injectable({ providedIn: 'root' })
export class ProductsService {
  private http = inject(HttpClient);
  private readonly API = '/api/products';

  // The observable is created once and shared.
  // shareReplay(1) keeps the last emitted value and replays it to every
  // new subscriber without issuing a second HTTP request.
  private products$: Observable<Product[]> = this.http
    .get<Product[]>(this.API)
    .pipe(shareReplay(1));

  getAll(): Observable<Product[]> {
    return this.products$;
  }
}
```

```typescript
// component usage — three components, one HTTP request
@Component({ ... })
export class ProductListComponent {
  private svc = inject(ProductsService);
  products$ = this.svc.getAll(); // replays cached value
}
```

### Why shareReplay(1) and Not share() or BehaviorSubject?

| Operator / Type | Replays to late subscriber? | Keeps last value? | New HTTP call on subscribe? |
|---|---|---|---|
| `share()` | No | No | Yes (if no current subscriber) |
| `shareReplay(1)` | Yes | Yes | No (replays) |
| `BehaviorSubject` | Yes | Yes | Manual — you push values yourself |
| `ReplaySubject(1)` | Yes | Yes | Manual — you push values yourself |

`shareReplay(1)` is the minimum-ceremony option for read-only, single-fetch caches. For mutable state, `BehaviorSubject` gives you imperative `.next()` control.

### shareReplay with refCount

There are two flavors of `shareReplay`. This distinction is critical:

```typescript
// bufferSize, windowTime, scheduler — all optional
shareReplay(1)                          // refCount: false (default pre-RxJS 6.4)
shareReplay({ bufferSize: 1, refCount: true })  // refCount: true
```

- `refCount: false` (default until recent RxJS): the multicasted subject **never unsubscribes** from the source even when all downstream subscribers unsubscribe. The HTTP request stays "alive" and the cache is permanent for the service lifetime.
- `refCount: true`: the subject unsubscribes from the source when the subscriber count drops to zero. If a new subscriber arrives later, a **new HTTP request fires**. This is the correct choice for short-lived data but a footgun if you assume the value stays cached.

```typescript
// Safe production default — permanent cache, no re-fetch:
private products$ = this.http.get<Product[]>(this.API).pipe(
  shareReplay({ bufferSize: 1, refCount: false })
);
```

### Adding TTL to shareReplay

`shareReplay` has no built-in expiry. Combine with `timer` to expire:

```typescript
import { timer, switchMap, shareReplay } from 'rxjs';

private readonly CACHE_TTL_MS = 60_000; // 1 minute

private products$ = timer(0, this.CACHE_TTL_MS).pipe(
  switchMap(() => this.http.get<Product[]>(this.API)),
  shareReplay({ bufferSize: 1, refCount: false })
);
```

`timer(0, TTL)` emits at 0 ms then every TTL ms. `switchMap` cancels the previous inner observable and fires a new request each interval. Any subscriber at any point in the interval window gets the latest cached response.

---

## 2. HTTP Cache-Control Headers and Browser Caching

The browser's HTTP cache is the most efficient cache layer — no JavaScript involved. Angular's `HttpClient` is a thin wrapper over `XMLHttpRequest`/`fetch`, so standard HTTP caching semantics apply fully.

### Cache-Control Directives Reference

```
Cache-Control: max-age=3600           # Cache for 1 hour
Cache-Control: no-cache               # Must revalidate with server before use
Cache-Control: no-store               # Never cache (auth tokens, PII)
Cache-Control: private                # Only browser cache, not CDN/proxy
Cache-Control: public                 # CDN + browser
Cache-Control: must-revalidate        # Stale entries must be revalidated
Cache-Control: stale-while-revalidate=60  # Serve stale for 60s while refreshing
Cache-Control: immutable              # Never revalidate (fingerprinted assets)
```

### Server-Side Headers Angular Relies On

When your API sets these headers, the browser caches the response automatically:

```
HTTP/1.1 200 OK
Cache-Control: public, max-age=300
ETag: "abc123"
Last-Modified: Tue, 03 Jun 2025 10:00:00 GMT
Vary: Accept-Encoding, Authorization
```

**Angular has no special hook needed** — the browser/CDN handles caching transparently. A second `HttpClient.get()` to the same URL within the `max-age` window will be served from the disk cache with status `200 (from disk cache)`, never reaching the network.

### Bypassing the Browser Cache from Angular

Sometimes you need to force a fresh fetch (e.g., after a POST that mutates server state):

```typescript
import { HttpParams } from '@angular/common/http';

// Option 1: Cache-busting query param
const params = new HttpParams().set('_t', Date.now().toString());
this.http.get<Product[]>('/api/products', { params });

// Option 2: Cache-Control request header
this.http.get<Product[]>('/api/products', {
  headers: { 'Cache-Control': 'no-cache', 'Pragma': 'no-cache' }
});
```

Note: `Cache-Control: no-cache` on a **request** tells the browser and any intermediate caches to revalidate, not to skip caching on the response. Use query params for guaranteed cache bypass.

### Vary Header — The CDN/Proxy Trap

The `Vary` header tells shared caches which request headers define cache key uniqueness. If your API returns `Vary: Authorization`, every unique `Authorization` token gets its own cache entry — this is correct for user-specific responses, but it means CDN caching is effectively disabled for authenticated routes (each user's token is different).

```typescript
// Wrong: identical URL, different data per user, no Vary protection
// Correct server pattern for user-specific data:
// Cache-Control: private, max-age=300
// (no Vary needed — "private" already prevents CDN caching)
```

---

## 3. ETag and Conditional Requests with HttpClient

ETags allow the server to say "nothing changed — use your cached copy" with a `304 Not Modified`. The browser saves full response bandwidth while still hitting the server to check freshness.

### How ETags Work

```
Client → GET /api/products
Server → 200 OK
         ETag: "v1-abc123"
         body: [...]

Client → GET /api/products
         If-None-Match: "v1-abc123"

Server → 304 Not Modified   ← no body, client uses cached response
     OR  200 OK              ← new ETag, new body
         ETag: "v2-def456"
```

### Browser-Level ETag (Zero Angular Code)

If the server returns `ETag` and `Cache-Control: no-cache` (or `must-revalidate`), the **browser handles conditional requests automatically**. You write zero Angular code:

```typescript
// This is all you need — the browser sends If-None-Match on subsequent calls
this.http.get<Product[]>('/api/products').subscribe(console.log);
```

### Manual ETag Management with HttpClient

When you need fine-grained control — for example, when `shareReplay` holds the last response and you want to revalidate on user action:

```typescript
// etag-cache.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpResponse } from '@angular/common/http';
import { Observable, of, tap } from 'rxjs';
import { map, switchMap } from 'rxjs/operators';

interface CacheEntry<T> {
  data: T;
  etag: string;
}

@Injectable({ providedIn: 'root' })
export class EtagCacheService {
  private http = inject(HttpClient);
  private cache = new Map<string, CacheEntry<unknown>>();

  get<T>(url: string): Observable<T> {
    const entry = this.cache.get(url) as CacheEntry<T> | undefined;

    const headers: Record<string, string> = {};
    if (entry?.etag) {
      headers['If-None-Match'] = entry.etag;
    }

    return this.http
      .get<T>(url, { observe: 'response', headers })
      .pipe(
        switchMap((response: HttpResponse<T>) => {
          if (response.status === 304 && entry) {
            // Server confirmed: data unchanged, return cached value
            return of(entry.data);
          }

          const newEtag = response.headers.get('ETag') ?? '';
          const body = response.body as T;

          this.cache.set(url, { data: body, etag: newEtag });
          return of(body);
        })
      );
  }

  invalidate(url: string): void {
    this.cache.delete(url);
  }
}
```

### Last-Modified Conditional Requests

If the server doesn't support ETags but does return `Last-Modified`:

```typescript
// Use If-Modified-Since header
headers['If-Modified-Since'] = entry.lastModified;
// Server returns 304 if unchanged, 200 + new Last-Modified if changed
```

ETags are preferred over `Last-Modified` when both are available — ETags handle sub-second changes and server restarts that reset file timestamps.

---

## 4. Implementing a Cache Interceptor

An interceptor is the cleanest place to apply caching transparently — components and services stay unaware of cache logic.

### Full Cache Interceptor with TTL

```typescript
// cache.interceptor.ts
import { Injectable, inject } from '@angular/core';
import {
  HttpRequest,
  HttpHandlerFn,
  HttpEvent,
  HttpResponse,
  HttpContext,
  HttpContextToken,
} from '@angular/common/http';
import { Observable, of, tap } from 'rxjs';

// ── Opt-in token — components pass this to control caching per-request ──────
export const CACHE_TTL = new HttpContextToken<number>(() => 0); // 0 = no cache
export const CACHE_BYPASS = new HttpContextToken<boolean>(() => false);

// ── Internal cache entry ──────────────────────────────────────────────────────
interface CacheEntry {
  response: HttpResponse<unknown>;
  expiresAt: number;
}

// ── The cache store (lives with the interceptor, reset on service destroy) ───
class HttpCacheStore {
  private store = new Map<string, CacheEntry>();

  get(key: string): HttpResponse<unknown> | null {
    const entry = this.store.get(key);
    if (!entry) return null;
    if (Date.now() > entry.expiresAt) {
      this.store.delete(key);
      return null;
    }
    return entry.response;
  }

  set(key: string, response: HttpResponse<unknown>, ttlMs: number): void {
    this.store.set(key, { response, expiresAt: Date.now() + ttlMs });
  }

  delete(key: string): void {
    this.store.delete(key);
  }

  deleteByPrefix(prefix: string): void {
    for (const key of this.store.keys()) {
      if (key.startsWith(prefix)) this.store.delete(key);
    }
  }

  clear(): void {
    this.store.clear();
  }
}

// Singleton store shared across all interceptor invocations
const cacheStore = new HttpCacheStore();
export { cacheStore as httpCacheStore }; // export for manual invalidation

// ── Functional interceptor (Angular 15+) ──────────────────────────────────────
export function cacheInterceptor(
  req: HttpRequest<unknown>,
  next: HttpHandlerFn
): Observable<HttpEvent<unknown>> {
  // Only cache GET requests
  if (req.method !== 'GET') return next(req);

  const ttl = req.context.get(CACHE_TTL);
  const bypass = req.context.get(CACHE_BYPASS);

  // No TTL configured or bypass requested — pass through
  if (ttl === 0 || bypass) return next(req);

  const cacheKey = req.urlWithParams;
  const cached = cacheStore.get(cacheKey);

  if (cached) {
    // Return a clone — HttpResponse objects are mutable
    return of(cached.clone());
  }

  return next(req).pipe(
    tap((event) => {
      if (event instanceof HttpResponse && event.status === 200) {
        cacheStore.set(cacheKey, event, ttl);
      }
    })
  );
}
```

### Registering the Interceptor

```typescript
// app.config.ts
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { cacheInterceptor } from './cache.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(withInterceptors([cacheInterceptor]))
  ]
};
```

### Using the Interceptor from a Service

```typescript
// products.service.ts
import { HttpContext } from '@angular/common/http';
import { CACHE_TTL, CACHE_BYPASS, httpCacheStore } from './cache.interceptor';

@Injectable({ providedIn: 'root' })
export class ProductsService {
  private http = inject(HttpClient);

  getAll(): Observable<Product[]> {
    return this.http.get<Product[]>('/api/products', {
      context: new HttpContext().set(CACHE_TTL, 60_000) // 1 min TTL
    });
  }

  // Bypass cache for force-refresh
  refresh(): Observable<Product[]> {
    return this.http.get<Product[]>('/api/products', {
      context: new HttpContext()
        .set(CACHE_TTL, 60_000)
        .set(CACHE_BYPASS, true)
    });
  }

  // Manual invalidation
  invalidateCache(): void {
    httpCacheStore.deleteByPrefix('/api/products');
  }
}
```

### Interceptor Cache Key Design

The cache key must uniquely identify a resource. `req.urlWithParams` is usually sufficient, but consider:

```typescript
// Include relevant headers if responses vary by them
const cacheKey = `${req.urlWithParams}|${req.headers.get('Accept-Language') ?? ''}`;

// For user-specific data — include a user ID, not auth token
// (tokens rotate; user IDs are stable)
const userId = authService.currentUserId;
const cacheKey = `${req.urlWithParams}|user:${userId}`;
```

---

## 5. Cache Invalidation Strategies

> "There are only two hard things in computer science: cache invalidation and naming things." — Phil Karlton

### TTL-Based Invalidation

The simplest strategy: every cache entry has a fixed lifetime.

```typescript
// Service-level TTL with timer-based refresh
@Injectable({ providedIn: 'root' })
export class TtlCacheService {
  private http = inject(HttpClient);
  private cache = new Map<string, { data$: Observable<unknown>; expiresAt: number }>();

  get<T>(url: string, ttlMs: number): Observable<T> {
    const existing = this.cache.get(url);

    if (existing && Date.now() < existing.expiresAt) {
      return existing.data$ as Observable<T>;
    }

    const data$ = this.http.get<T>(url).pipe(
      shareReplay({ bufferSize: 1, refCount: false })
    );

    this.cache.set(url, { data$, expiresAt: Date.now() + ttlMs });
    return data$;
  }
}
```

**Tradeoffs:**
- Simple to reason about
- Stale data guaranteed for up to TTL duration
- Poor for high-frequency mutation scenarios

### Manual Eviction

Explicit invalidation triggered by write operations:

```typescript
@Injectable({ providedIn: 'root' })
export class ProductsService {
  private http = inject(HttpClient);
  private cache = new Map<string, Observable<unknown>>();

  getAll(): Observable<Product[]> {
    if (!this.cache.has('products')) {
      this.cache.set(
        'products',
        this.http.get<Product[]>('/api/products').pipe(
          shareReplay({ bufferSize: 1, refCount: false })
        )
      );
    }
    return this.cache.get('products') as Observable<Product[]>;
  }

  create(product: Partial<Product>): Observable<Product> {
    return this.http.post<Product>('/api/products', product).pipe(
      tap(() => this.cache.delete('products')) // invalidate on success
    );
  }

  update(id: string, changes: Partial<Product>): Observable<Product> {
    return this.http.patch<Product>(`/api/products/${id}`, changes).pipe(
      tap(() => this.cache.delete('products'))
    );
  }
}
```

### Event-Driven Invalidation

When multiple services or modules can trigger invalidation, use a shared event bus:

```typescript
// cache-events.service.ts
import { Injectable } from '@angular/core';
import { Subject } from 'rxjs';
import { filter } from 'rxjs/operators';

export type CacheEvent =
  | { type: 'INVALIDATE'; key: string }
  | { type: 'INVALIDATE_PREFIX'; prefix: string }
  | { type: 'CLEAR_ALL' };

@Injectable({ providedIn: 'root' })
export class CacheEventsService {
  private events$ = new Subject<CacheEvent>();

  emit(event: CacheEvent): void {
    this.events$.next(event);
  }

  on(type: CacheEvent['type']) {
    return this.events$.pipe(filter((e) => e.type === type));
  }

  invalidate(key: string): void {
    this.emit({ type: 'INVALIDATE', key });
  }

  invalidatePrefix(prefix: string): void {
    this.emit({ type: 'INVALIDATE_PREFIX', prefix });
  }
}
```

```typescript
// products.service.ts — listens to events
@Injectable({ providedIn: 'root' })
export class ProductsService implements OnDestroy {
  private http = inject(HttpClient);
  private cacheEvents = inject(CacheEventsService);
  private cache = new Map<string, Observable<unknown>>();
  private destroy$ = new Subject<void>();

  constructor() {
    // Any module can emit an invalidation event
    this.cacheEvents
      .on('INVALIDATE_PREFIX')
      .pipe(takeUntil(this.destroy$))
      .subscribe((event) => {
        if (event.type === 'INVALIDATE_PREFIX') {
          this.evictByPrefix(event.prefix);
        }
      });
  }

  private evictByPrefix(prefix: string): void {
    for (const key of this.cache.keys()) {
      if (key.startsWith(prefix)) this.cache.delete(key);
    }
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### WebSocket-Driven Invalidation

For real-time apps, invalidate cache entries when the server pushes update notifications:

```typescript
@Injectable({ providedIn: 'root' })
export class RealtimeCacheService {
  private ws = inject(WebSocketService); // your WS abstraction
  private productsService = inject(ProductsService);
  private destroy$ = new Subject<void>();

  constructor() {
    this.ws
      .on<{ resource: string; id: string }>('resource:updated')
      .pipe(takeUntil(this.destroy$))
      .subscribe(({ resource }) => {
        if (resource === 'products') {
          this.productsService.invalidateCache();
        }
      });
  }
}
```

---

## 6. Stale-While-Revalidate Pattern in Angular

Stale-while-revalidate (SWR) serves the cached (possibly stale) value immediately, then silently fetches a fresh copy in the background. The UI shows data instantly and updates when fresh data arrives — no loading spinners for secondary navigations.

### Implementation with BehaviorSubject

```typescript
// swr-cache.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { BehaviorSubject, Observable } from 'rxjs';
import { tap, filter } from 'rxjs/operators';

interface SwrEntry<T> {
  data: T;
  fetchedAt: number;
}

@Injectable({ providedIn: 'root' })
export class SwrCacheService {
  private http = inject(HttpClient);

  // Map of URL → BehaviorSubject holding the current value
  private subjects = new Map<string, BehaviorSubject<unknown>>();
  private meta = new Map<string, { fetchedAt: number }>();

  get<T>(url: string, staleMs = 30_000): Observable<T> {
    if (!this.subjects.has(url)) {
      // First request — no cached value yet, use null as placeholder
      this.subjects.set(url, new BehaviorSubject<T | null>(null));
    }

    const subject$ = this.subjects.get(url) as BehaviorSubject<T | null>;
    const meta = this.meta.get(url);
    const isStale = !meta || Date.now() - meta.fetchedAt > staleMs;

    if (isStale) {
      // Fire background revalidation — non-blocking
      this.revalidate<T>(url);
    }

    // Return the stream, filtering out the initial null (no data yet)
    return subject$.pipe(filter((v): v is T => v !== null));
  }

  private revalidate<T>(url: string): void {
    this.http.get<T>(url).subscribe({
      next: (data) => {
        const subject$ = this.subjects.get(url) as BehaviorSubject<T>;
        subject$.next(data);
        this.meta.set(url, { fetchedAt: Date.now() });
      },
      error: (err) => {
        // On error: keep serving stale data, log the failure
        console.error(`SWR revalidation failed for ${url}`, err);
      }
    });
  }

  invalidate(url: string): void {
    this.meta.delete(url);
    // Next call to get() will trigger immediate revalidation
  }
}
```

### Usage — Always Immediate, Always Fresh

```typescript
// Component gets instant data, then silently receives updates
@Component({
  template: `
    @if (products$ | async; as products) {
      <product-list [products]="products" />
    } @else {
      <skeleton-loader />
    }
  `
})
export class ProductsComponent {
  private swr = inject(SwrCacheService);
  products$ = this.swr.get<Product[]>('/api/products', 60_000);
}
```

### SWR with Angular Signals (Angular 17+)

```typescript
import { Injectable, inject, signal, effect } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Injectable({ providedIn: 'root' })
export class SwrSignalService {
  private http = inject(HttpClient);

  createResource<T>(url: string, staleMs = 30_000) {
    const data = signal<T | null>(null);
    const isLoading = signal(false);
    const error = signal<unknown>(null);
    let lastFetch = 0;

    const fetch = () => {
      isLoading.set(true);
      this.http.get<T>(url).subscribe({
        next: (value) => {
          data.set(value);
          lastFetch = Date.now();
          isLoading.set(false);
          error.set(null);
        },
        error: (err) => {
          error.set(err);
          isLoading.set(false);
        }
      });
    };

    const revalidateIfStale = () => {
      if (Date.now() - lastFetch > staleMs) fetch();
    };

    fetch(); // initial load

    return { data, isLoading, error, revalidate: fetch, revalidateIfStale };
  }
}
```

---

## 7. TanStack Query vs Manual Caching in Angular

[TanStack Query for Angular](https://tanstack.com/query/latest/docs/framework/angular/overview) (via `@tanstack/angular-query-experimental`) brings the React Query mental model to Angular.

### What TanStack Query Provides Out of the Box

| Feature | TanStack Query | Manual (shareReplay / interceptor) |
|---|---|---|
| Stale-while-revalidate | Built-in | You implement it |
| Automatic background refetch | Built-in | You implement it |
| Deduplication of in-flight requests | Built-in | You implement it |
| Optimistic updates | Built-in | You implement it |
| Refetch on window focus | Built-in | You implement it |
| Retry with backoff | Built-in | You implement it |
| Garbage collection | Built-in | Memory leak if you forget |
| DevTools | Built-in | None |
| Query invalidation | `queryClient.invalidateQueries()` | Manual |
| Pagination / infinite scroll | Built-in helpers | You implement it |

### TanStack Query Basic Usage in Angular

```typescript
// Install: npm install @tanstack/angular-query-experimental

// app.config.ts
import { provideAngularQuery, QueryClient } from '@tanstack/angular-query-experimental';

export const appConfig: ApplicationConfig = {
  providers: [
    provideAngularQuery(new QueryClient({
      defaultOptions: {
        queries: {
          staleTime: 60_000,        // data is fresh for 60s
          gcTime: 5 * 60_000,       // garbage collect after 5 min
          retry: 2,
          refetchOnWindowFocus: true,
        }
      }
    }))
  ]
};
```

```typescript
// products.component.ts
import { injectQuery, injectMutation, injectQueryClient } from '@tanstack/angular-query-experimental';
import { HttpClient } from '@angular/common/http';
import { lastValueFrom } from 'rxjs';

@Component({
  template: `
    @if (query.isPending()) {
      <spinner />
    } @else if (query.isError()) {
      <error-view [error]="query.error()" />
    } @else {
      <product-list [products]="query.data()!" />
      <button (click)="createMutation.mutate(newProduct)">Add</button>
    }
  `
})
export class ProductsComponent {
  private http = inject(HttpClient);
  private queryClient = injectQueryClient();

  query = injectQuery(() => ({
    queryKey: ['products'],
    queryFn: () => lastValueFrom(this.http.get<Product[]>('/api/products')),
    staleTime: 30_000,
  }));

  createMutation = injectMutation(() => ({
    mutationFn: (product: Partial<Product>) =>
      lastValueFrom(this.http.post<Product>('/api/products', product)),
    onSuccess: () => {
      // Invalidate and refetch
      this.queryClient.invalidateQueries({ queryKey: ['products'] });
    }
  }));
}
```

### When to Use TanStack Query

Use TanStack Query when:
- The app is heavily CRUD-oriented with many server-state interactions
- You need optimistic updates, background sync, or window-focus refetch
- Multiple components need the same server data with automatic deduplication
- You want mature DevTools and a well-tested cache invalidation model
- You're building a new app and can afford the dependency

Use manual caching (shareReplay / interceptor) when:
- The app has limited data-fetching complexity (read-heavy, few mutations)
- You cannot add external dependencies (locked-down enterprise environment)
- You need sub-millisecond interception logic tied to custom HTTP context tokens
- You have an existing service architecture already handling state
- The data is truly static within a user session (configuration, feature flags)

### The Hybrid Approach

TanStack Query and Angular's HttpClient cache interceptor are not mutually exclusive. Use TanStack Query for user-facing server state, and keep a lightweight HTTP interceptor for caching shared infrastructure calls (feature flags, translations, reference data):

```typescript
// Feature flags — interceptor-cached, never need SWR or mutations
this.http.get('/api/config/flags', {
  context: new HttpContext().set(CACHE_TTL, 24 * 60 * 60_000) // 24h
});

// Products — TanStack Query, needs mutations and invalidation
query = injectQuery(() => ({ queryKey: ['products'], queryFn: ... }));
```

---

## 8. Common Pitfalls

### Pitfall 1: Stale Data Delivered After Mutation

**Problem:** A component reads from a cached observable, a mutation fires in another component, and the cache is never invalidated — the user sees outdated data.

```typescript
// BAD — cache never invalidated after create()
products$ = this.http.get<Product[]>('/api/products').pipe(shareReplay(1));

create(p: Partial<Product>) {
  return this.http.post('/api/products', p); // doesn't touch products$
}
```

```typescript
// GOOD — reset the cache reference on mutation
private _products$: Observable<Product[]> | null = null;

get products$(): Observable<Product[]> {
  this._products$ ??= this.http.get<Product[]>('/api/products').pipe(
    shareReplay({ bufferSize: 1, refCount: false })
  );
  return this._products$;
}

create(p: Partial<Product>): Observable<Product> {
  return this.http.post<Product>('/api/products', p).pipe(
    tap(() => { this._products$ = null; }) // drop cache on success
  );
}
```

### Pitfall 2: Cache Stampede

**Problem:** The cache expires and multiple concurrent requests for the same resource all miss simultaneously, flooding the server.

```typescript
// BAD — race condition on expiry
get<T>(url: string): Observable<T> {
  if (this.isExpired(url)) {
    this.cache.delete(url); // multiple callers see no entry
    const req$ = this.http.get<T>(url).pipe(shareReplay(1)); // multiple created!
    this.cache.set(url, req$);
  }
  return this.cache.get(url) as Observable<T>;
}
```

```typescript
// GOOD — store the in-flight observable immediately (deduplicate)
private inflight = new Map<string, Observable<unknown>>();

get<T>(url: string, ttlMs: number): Observable<T> {
  // Serve cached entry if valid
  const cached = this.getCacheEntry<T>(url);
  if (cached) return cached;

  // Serve in-flight request if one is already running
  if (this.inflight.has(url)) {
    return this.inflight.get(url) as Observable<T>;
  }

  // Start new request, register in inflight map immediately
  const req$ = this.http.get<T>(url).pipe(
    tap((data) => {
      this.setCacheEntry(url, data, ttlMs);
      this.inflight.delete(url);
    }),
    shareReplay({ bufferSize: 1, refCount: true }) // refCount: true cleans up after all subscribers done
  );

  this.inflight.set(url, req$);
  return req$;
}
```

### Pitfall 3: Memory Leaks from shareReplay

**Problem:** `shareReplay({ refCount: false })` holds a reference to the source subscription forever. In a lazy-loaded module that gets unloaded, the service may be destroyed but the underlying `HttpClient` subscription lives on, or worse — the service isn't GC'd because the observable still holds a reference.

```typescript
// BAD — service-level observable with refCount: false in a feature module service
@Injectable() // Not providedIn: 'root' — scoped to a lazy module
export class FeatureService implements OnDestroy {
  // This subject never completes → memory leak when module unloads
  private data$ = this.http.get('/api/data').pipe(shareReplay(1));
}
```

```typescript
// GOOD — take(1) completes the source after first emission, shareReplay can then GC
private data$ = this.http.get('/api/data').pipe(
  take(1),           // HTTP completes naturally, but take(1) makes it explicit
  shareReplay({ bufferSize: 1, refCount: true }) // refCount: true cleans up
);
```

```typescript
// GOOD alternative — destroy the observable manually
@Injectable()
export class FeatureService implements OnDestroy {
  private cache = new Map<string, BehaviorSubject<unknown>>();

  ngOnDestroy(): void {
    // Complete all subjects — subscribers know the stream is done
    this.cache.forEach((subject) => subject.complete());
    this.cache.clear();
  }
}
```

### Pitfall 4: Caching Non-Idempotent Responses

**Problem:** POST or PUT responses cached and replayed — mutation appears to succeed even when the server never received the request.

The cache interceptor shown in Section 4 guards against this with `if (req.method !== 'GET') return next(req)`. Never cache non-GET requests. Also beware of caching responses with side-effect query params:

```typescript
// This GET triggers a server-side action — cache key should exclude it
// or this endpoint should not be cached at all
GET /api/reports/generate?format=pdf
```

### Pitfall 5: Wrong refCount Default in Legacy Code

Before RxJS 6.4, `shareReplay(1)` used `refCount: false`. In RxJS 6.4+, the default **remained** `refCount: false` for `shareReplay(bufferSize)` (the single-argument overload) for backwards compatibility. This is a known footgun:

```typescript
// Ambiguous — refCount: false regardless of RxJS version
shareReplay(1)

// Explicit — always correct, intention clear
shareReplay({ bufferSize: 1, refCount: false }) // permanent cache
shareReplay({ bufferSize: 1, refCount: true })  // auto-cleanup cache
```

Always use the object-argument form in production code.

### Pitfall 6: Cache Key Collisions with Parameterized URLs

```typescript
// BAD — same cache key for different resources
const key = '/api/products'; // ignores query params

// GOOD — use req.urlWithParams
const key = req.urlWithParams;
// → '/api/products?category=shoes&page=2' is distinct from '/api/products?page=1'
```

---

## 9. Interview Q&A

---

### Q1: "Explain the difference between shareReplay(1) with refCount true vs false, and when you'd choose each."

**Model Answer:**

`shareReplay({ bufferSize: 1, refCount: false })` creates a multicasted subject that **never unsubscribes** from the source observable, even when all downstream subscribers unsubscribe. The cached value remains in memory for the lifetime of the service. This is the right choice when the service is a singleton (`providedIn: 'root'`) and the data should be fetched once per app session — e.g., user profile, reference data, feature flags.

`shareReplay({ bufferSize: 1, refCount: true })` unsubscribes from the source when the subscriber count drops to zero and resubscribes (re-fetches) when a new subscriber arrives. This is appropriate when components are short-lived and you want automatic cache cleanup to prevent memory leaks — e.g., a feature-module-scoped service where the cached data should not outlive the module.

The footgun: the single-argument form `shareReplay(1)` is equivalent to `refCount: false` in all current RxJS versions. I always use the object form to make intent explicit and avoid confusion during code review.

---

### Q2: "How would you design a cache interceptor that avoids the cache stampede problem?"

**Model Answer:**

A cache stampede occurs when multiple concurrent requests for the same URL all miss a recently-expired cache entry and all fire new HTTP requests simultaneously. The fix is to store the **in-flight observable** in the cache immediately — before the response arrives — so subsequent concurrent requests share the same observable via `shareReplay` instead of creating independent requests.

The flow is:
1. Check if a valid cached response exists → return it.
2. Check if an in-flight request for this URL exists → return its observable (already multicasted with `shareReplay`).
3. Otherwise, create a new request, immediately insert it into the in-flight map, and on response, move the result to the cache store and remove from in-flight.

The key insight is that `shareReplay` on the in-flight observable means every concurrent subscriber receives the same single response without triggering additional HTTP calls. The `refCount: true` option on the in-flight `shareReplay` ensures the in-flight entry cleans up after all subscribers have received the response.

---

### Q3: "A user creates a product, but the product list still shows the old data for several seconds. How do you debug and fix this?"

**Model Answer:**

This is a cache invalidation bug. The most common causes, in order of likelihood:

1. **`shareReplay` cache not invalidated on mutation.** The `products$` observable was created once and shared. The `create()` method fires a POST but doesn't reset the cached observable. Fix: `tap(() => this._products$ = null)` in the mutation pipeline, then re-assign on next `getAll()` call.

2. **HTTP Cache-Control max-age.** The server returns `Cache-Control: max-age=30`. The browser serves the cached response without hitting the network. Debug by checking the Network tab — `200 (disk cache)` or `200 (memory cache)` confirms this. Fix: send `Cache-Control: no-cache` on the GET after mutation, or append a cache-busting query param.

3. **Interceptor TTL not expired.** A cache interceptor with a TTL is serving a stale response. Fix: call `httpCacheStore.deleteByPrefix('/api/products')` in the mutation's `tap()`.

4. **Optimistic update not rolled back on error.** If using optimistic updates, a failed POST can leave stale data in the UI. Fix: handle `onError` in the mutation to rollback.

I would start by opening DevTools Network tab, triggering the create, then triggering the list refetch — checking whether the GET actually leaves the browser determines whether the issue is in the browser cache or the Angular service layer.

---

### Q4: "When would you choose TanStack Query for Angular over a custom caching service, and what are the tradeoffs?"

**Model Answer:**

I'd choose TanStack Query when the application has significant server-state complexity: many entities, frequent mutations, real-time consistency requirements, or the need for window-focus refetch and background sync. TanStack Query's `invalidateQueries` model is far more ergonomic than manually tracking which caches to evict after a given mutation — it uses query key hierarchies so `invalidateQueries(['products'])` busts all queries whose key starts with `'products'`, including paginated variants.

The tradeoffs against manual caching:

**TanStack Query advantages:** deduplication of concurrent requests, stale-while-revalidate out of the box, retry logic, DevTools, optimistic update helpers, garbage collection with configurable `gcTime`, pagination utilities.

**TanStack Query disadvantages:** external dependency (~25 kB gzipped) — for teams with strict bundle budgets this matters. The `@tanstack/angular-query-experimental` package is still experimental, so the API may change. The mental model (queries, mutations, query keys) adds onboarding cost for teams already fluent in RxJS patterns. It also wraps HttpClient responses in `lastValueFrom()` which feels awkward in a primarily Observable codebase.

My rule of thumb: use a custom service with `shareReplay` for apps with fewer than ~5 distinct cached entities and no complex mutation workflows. Once you're tracking invalidation across 10+ entity types, TanStack Query pays for itself in reduced bugs and boilerplate.

---

*Last updated: 2026-06-03 | Angular 17+ / RxJS 7+ / TanStack Query for Angular (experimental)*
