# HTTP Methods, Status Codes, and Headers

## Overview

HTTP is the protocol underlying every web application. A senior developer must know more than "GET fetches, POST creates" — they need to understand the semantics that browsers, CDNs, proxies, and service workers rely on: idempotency, safety, cacheability, and the precise meaning of headers like `Cache-Control`, `ETag`, `Authorization`, and CORS headers. This guide covers the full picture from method semantics to status code categories to critical request/response headers.

## HTTP Methods and Their Semantics

```
Method    Safe?  Idempotent?  Body?   Cacheable?  Use Case
────────  ─────  ───────────  ──────  ──────────  ──────────────────────────
GET        Yes     Yes        No      Yes         Retrieve resource
HEAD       Yes     Yes        No      Yes         Retrieve headers only
OPTIONS    Yes     Yes        No      Yes         Pre-flight / capability
POST        No      No        Yes     Rarely      Create resource / submit
PUT         No     Yes        Yes     No          Replace resource entirely
PATCH       No      No*       Yes     No          Partial update
DELETE      No     Yes        No      No          Delete resource
CONNECT     No      No        No      No          Tunnel (WebSocket upgrade)
TRACE       Yes     Yes       No      No          Loop-back diagnostic
```

**Safe** = no side effects on the server (read-only)
**Idempotent** = calling N times has same effect as calling once

### GET

```typescript
// GET: retrieve, safe, idempotent, cacheable
// ✅ Correct: parameters in query string
GET /api/posts?page=2&limit=20&sort=createdAt HTTP/1.1

// ❌ Incorrect: body on GET (some clients/servers reject it)
// GET /api/posts with body { filter: 'active' } — non-standard

// Response:
// 200 OK with body
// 304 Not Modified (ETag/Last-Modified cache hit)
// 404 Not Found
```

### POST vs PUT vs PATCH

```typescript
// POST: create new resource at a collection endpoint
// Server assigns the ID
POST /api/posts HTTP/1.1
Content-Type: application/json
{ "title": "New Post", "body": "..." }
// Response: 201 Created, Location: /api/posts/123

// PUT: replace entire resource at a specific endpoint
// Idempotent: calling twice has same result
PUT /api/posts/123 HTTP/1.1
Content-Type: application/json
{ "id": "123", "title": "Updated", "body": "...", "authorId": "u1" }
// Must include ALL fields — missing fields may be cleared

// PATCH: partial update
// Send only fields that change
PATCH /api/posts/123 HTTP/1.1
Content-Type: application/merge-patch+json
{ "title": "Updated Title Only" }
// authorId, body etc. remain unchanged

// TypeScript client pattern
async function updatePost(id: string, partial: Partial<Post>): Promise<Post> {
  const response = await fetch(`/api/posts/${id}`, {
    method: 'PATCH',
    headers: { 'Content-Type': 'application/merge-patch+json' },
    body: JSON.stringify(partial),
  });
  if (!response.ok) throw new ApiError(response.status, await response.json());
  return response.json();
}
```

### DELETE and Idempotency

```typescript
// DELETE: remove resource — idempotent
// First call: 200 OK or 204 No Content
// Second call (already deleted): 404 Not Found OR 204 No Content
// ✅ Both responses are valid — idempotency means same server state, not same response

// OPTIONS: CORS preflight
OPTIONS /api/posts HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: DELETE
Access-Control-Request-Headers: Authorization

// Server response to preflight:
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 86400
```

## Status Code Categories

```
1xx — Informational (rarely seen in frontend)
2xx — Success
3xx — Redirection
4xx — Client Error
5xx — Server Error
```

### 2xx Success

```
200 OK               General success. GET/PUT/PATCH responses.
201 Created          POST created a resource. Include Location header.
202 Accepted         Request accepted for async processing.
204 No Content       Success with no response body. DELETE, PUT with no return.
206 Partial Content  Range request satisfied (video streaming, resume download).
```

```typescript
// 201 Created — return the created resource and its URL
app.post('/api/posts', async (req, res) => {
  const post = await db.posts.create(req.body);
  res.status(201)
     .header('Location', `/api/posts/${post.id}`)
     .json(post);
});

// 204 No Content — nothing to return
app.delete('/api/posts/:id', async (req, res) => {
  await db.posts.delete(req.params.id);
  res.status(204).end(); // No body
});

// 202 Accepted — async job started
app.post('/api/exports', async (req, res) => {
  const jobId = await queue.enqueue('generate-report', req.body);
  res.status(202).json({ jobId, statusUrl: `/api/jobs/${jobId}` });
});
```

### 3xx Redirection

```
301 Moved Permanently   Resource has new URL. Browser caches indefinitely.
302 Found               Temporary redirect. Browser does NOT cache.
303 See Other           POST → redirect to GET result (PRG pattern).
304 Not Modified        Cache valid, use cached body.
307 Temporary Redirect  Like 302 but preserves HTTP method.
308 Permanent Redirect  Like 301 but preserves HTTP method.
```

```typescript
// 303 See Other — Post/Redirect/Get pattern
app.post('/api/forms/contact', async (req, res) => {
  await processForm(req.body);
  res.status(303).header('Location', '/forms/success').end();
  // Browser GETs /forms/success — prevents double-submit on refresh
});

// 304 Not Modified — conditional GET
app.get('/api/posts/:id', async (req, res) => {
  const post = await db.posts.findById(req.params.id);
  const etag = `"${post.version}"`;
  
  if (req.headers['if-none-match'] === etag) {
    return res.status(304).end(); // Client has current version
  }
  
  res.header('ETag', etag).json(post);
});
```

### 4xx Client Errors

```
400 Bad Request         Malformed request (invalid JSON, missing required field).
401 Unauthorized        Not authenticated. Authenticate and retry.
403 Forbidden           Authenticated but not authorized.
404 Not Found           Resource doesn't exist (or don't reveal it exists).
405 Method Not Allowed  HTTP method not supported for this endpoint.
409 Conflict            State conflict (duplicate, version mismatch, concurrent edit).
410 Gone                Resource existed but was permanently deleted.
422 Unprocessable       Valid syntax but semantic errors (validation failures).
429 Too Many Requests   Rate limit exceeded. Include Retry-After header.
```

```typescript
// Proper 4xx usage
class ApiError extends Error {
  constructor(public status: number, public code: string, message: string) {
    super(message);
  }
}

// 400 vs 422
app.post('/api/posts', async (req, res) => {
  // 400: can't even parse the request
  if (!req.body || typeof req.body !== 'object') {
    return res.status(400).json({ error: 'Invalid JSON body' });
  }
  
  // 422: parsed fine but validation fails
  const errors = validatePost(req.body);
  if (errors.length) {
    return res.status(422).json({ errors });
  }
  
  const post = await db.posts.create(req.body);
  res.status(201).json(post);
});

// 401 vs 403
app.get('/api/admin/users', authenticate, (req, res) => {
  if (!req.user) {
    // 401: no credentials provided
    return res.status(401)
      .header('WWW-Authenticate', 'Bearer realm="API"')
      .json({ error: 'Authentication required' });
  }
  
  if (!req.user.isAdmin) {
    // 403: credentials valid but insufficient permissions
    return res.status(403).json({ error: 'Admin access required' });
  }
  
  res.json(await db.users.findAll());
});

// 409 Conflict — optimistic locking
app.put('/api/posts/:id', async (req, res) => {
  const post = await db.posts.findById(req.params.id);
  
  if (post.version !== req.body.version) {
    return res.status(409).json({
      error: 'Version conflict — resource was modified since you fetched it',
      currentVersion: post.version,
    });
  }
  
  const updated = await db.posts.update(req.params.id, req.body);
  res.json(updated);
});
```

### 5xx Server Errors

```
500 Internal Server Error  Unexpected server failure.
502 Bad Gateway            Upstream server (proxy/LB) got invalid response.
503 Service Unavailable    Server temporarily down (deploying, overloaded).
504 Gateway Timeout        Upstream server didn't respond in time.
```

```typescript
// 503 with Retry-After
app.use((req, res, next) => {
  if (isMaintenanceMode()) {
    return res.status(503)
      .header('Retry-After', '3600') // retry in 1 hour
      .json({ error: 'Maintenance in progress', retryAfter: 3600 });
  }
  next();
});
```

## Important HTTP Headers

### Cache-Control

```
Direction  Value                    Meaning
─────────  ───────────────────────  ─────────────────────────────────────
Request    no-cache                 Revalidate before using cached response
Request    no-store                 Don't cache this request at all
Response   public                   Cacheable by CDN and browser
Response   private                  Browser only, not CDN
Response   no-cache                 Cache but revalidate before use
Response   no-store                 Do not cache at all
Response   max-age=3600             Fresh for 3600 seconds
Response   s-maxage=86400           CDN max age (overrides max-age for CDNs)
Response   stale-while-revalidate   Serve stale while fetching fresh
Response   immutable                Never revalidate (for content-hashed assets)
```

```typescript
// Headers for different resource types
const headers = {
  // Static assets with content hash — cache forever
  'main.a3f4b2c1.js': 'Cache-Control: public, max-age=31536000, immutable',
  
  // HTML pages — always revalidate
  'index.html': 'Cache-Control: no-cache',
  
  // API responses — short cache, CDN can cache
  'GET /api/products': 'Cache-Control: public, max-age=60, stale-while-revalidate=300',
  
  // User-specific API — browser only, short TTL
  'GET /api/profile': 'Cache-Control: private, max-age=300',
  
  // Sensitive data — no cache at all
  'GET /api/payment': 'Cache-Control: no-store',
};
```

### ETag and Conditional Requests

```typescript
// ETag: fingerprint of the response body
// If-None-Match: conditional GET using ETag
// If-Match: conditional PUT/PATCH (optimistic locking)

// Server: send ETag with response
app.get('/api/posts/:id', async (req, res) => {
  const post = await db.posts.findById(req.params.id);
  const etag = `"${createHash('md5').update(JSON.stringify(post)).digest('hex')}"`;
  
  res.setHeader('ETag', etag);
  res.setHeader('Last-Modified', post.updatedAt.toUTCString());
  res.json(post);
});

// Client: conditional GET — saves bandwidth if unchanged
async function fetchPostWithCache(id: string, cachedEtag?: string) {
  const headers: HeadersInit = {};
  if (cachedEtag) headers['If-None-Match'] = cachedEtag;
  
  const response = await fetch(`/api/posts/${id}`, { headers });
  
  if (response.status === 304) {
    return null; // use cached version
  }
  
  const post = await response.json();
  const etag = response.headers.get('ETag');
  return { post, etag };
}
```

### Authorization Headers

```typescript
// Common Authorization schemes

// Bearer token (JWT, OAuth 2.0)
headers: { Authorization: 'Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...' }

// Basic auth (username:password base64 encoded — only over HTTPS!)
const credentials = btoa(`${username}:${password}`);
headers: { Authorization: `Basic ${credentials}` }

// API Key (common patterns)
headers: { 'X-API-Key': 'sk_live_abc123...' }       // custom header
headers: { Authorization: 'ApiKey sk_live_abc123' }   // Authorization header

// TypeScript: Auth interceptor pattern
class AuthenticatedFetch {
  constructor(private getToken: () => Promise<string | null>) {}

  async fetch(url: string, options: RequestInit = {}): Promise<Response> {
    const token = await this.getToken();
    
    const headers = new Headers(options.headers);
    if (token) headers.set('Authorization', `Bearer ${token}`);
    
    const response = await fetch(url, { ...options, headers });
    
    if (response.status === 401) {
      // Token expired — try refresh
      const newToken = await this.refreshToken();
      if (newToken) {
        headers.set('Authorization', `Bearer ${newToken}`);
        return fetch(url, { ...options, headers });
      }
    }
    
    return response;
  }
}
```

### Content-Type and Accept

```typescript
// Content-Type: describes the body being sent
const contentTypes = {
  json:         'application/json',
  form:         'application/x-www-form-urlencoded',
  multipart:    'multipart/form-data',   // file uploads — browser sets this with boundary
  text:         'text/plain; charset=UTF-8',
  html:         'text/html; charset=UTF-8',
  jsonPatch:    'application/json-patch+json',
  mergePatch:   'application/merge-patch+json',
  octetStream:  'application/octet-stream', // binary
};

// Accept: tells server what the client can handle
headers: { Accept: 'application/json' }
headers: { Accept: 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8' }

// Content negotiation
app.get('/api/posts/:id', (req, res) => {
  const post = getPost(req.params.id);
  
  res.format({
    'application/json': () => res.json(post),
    'text/html':        () => res.render('post', { post }),
    default:            () => res.status(406).send('Not Acceptable'),
  });
});
```

### CORS Headers

```
CORS Flow:

Browser                    Server
   │                          │
   │── Simple Request ────────►│
   │◄── Response + Access-Control-Allow-Origin: * ──│
   │
   │── Pre-flight OPTIONS ─────►│  (for POST/PUT/DELETE with custom headers)
   │◄── 204 + Access-Control-* ─│
   │
   │── Actual Request ──────────►│
   │◄── Response ───────────────│
```

```typescript
// Express CORS middleware
app.use((req, res, next) => {
  const origin = req.headers.origin;
  const allowedOrigins = ['https://app.example.com', 'https://admin.example.com'];

  if (origin && allowedOrigins.includes(origin)) {
    res.setHeader('Access-Control-Allow-Origin', origin);
    res.setHeader('Vary', 'Origin'); // tell CDN to cache per-origin
  }

  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, PATCH, DELETE, OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization, X-Request-ID');
  res.setHeader('Access-Control-Allow-Credentials', 'true');
  res.setHeader('Access-Control-Max-Age', '86400'); // cache preflight 24h

  if (req.method === 'OPTIONS') {
    return res.status(204).end(); // preflight complete
  }

  next();
});
```

### Security Headers

```typescript
// Essential security headers
const securityHeaders = {
  // Prevent content type sniffing
  'X-Content-Type-Options': 'nosniff',
  
  // Clickjacking protection
  'X-Frame-Options': 'DENY',
  
  // Force HTTPS
  'Strict-Transport-Security': 'max-age=31536000; includeSubDomains; preload',
  
  // Referrer policy
  'Referrer-Policy': 'strict-origin-when-cross-origin',
  
  // Permissions policy (formerly Feature-Policy)
  'Permissions-Policy': 'camera=(), microphone=(), geolocation=(self)',
  
  // Content Security Policy
  'Content-Security-Policy': "default-src 'self'; script-src 'self' 'nonce-{nonce}'",
};
```

### Request Tracing Headers

```typescript
// Distributed tracing — correlate logs across services
const tracingHeaders = {
  'X-Request-ID':       crypto.randomUUID(),        // unique per request
  'X-Correlation-ID':   parentCorrelationId,         // traces a user journey
  'X-B3-TraceId':       '80f198ee56343ba864fe8b2a57d3eff7',  // Zipkin
  'X-B3-SpanId':        'e457b5a2e4d86bd1',
  'traceparent':        '00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01', // W3C
};
```

## Common Mistakes

### 1. Using POST for Idempotent Operations

```typescript
// ❌ Bad: non-idempotent endpoint for an idempotent action
POST /api/posts/123/publish   // creates duplicate publishes on retry

// ✅ Good: idempotent operation via PUT or PATCH
PUT /api/posts/123/status
{ "status": "published" }
// Calling twice has the same result
```

### 2. Returning 200 Instead of 201 for Created Resources

```typescript
// ❌ Bad
app.post('/api/users', (req, res) => {
  const user = createUser(req.body);
  res.status(200).json(user); // clients can't tell if created or fetched

// ✅ Good
  res.status(201).header('Location', `/api/users/${user.id}`).json(user);
});
```

### 3. Confusing 401 and 403

```typescript
// ❌ Bad: returning 403 when user isn't logged in
if (!req.user) return res.status(403).json({ error: 'Forbidden' });
// Browser/client doesn't know to prompt for login

// ✅ Good:
// 401 = "Who are you? Please authenticate."
// 403 = "I know who you are, you just can't do this."
if (!req.user) return res.status(401).header('WWW-Authenticate', 'Bearer').json(...);
if (!req.user.canDelete) return res.status(403).json(...);
```

### 4. Not Setting Vary Header with CORS

```typescript
// ❌ Bad: CDN caches response for origin-A, serves to origin-B
res.setHeader('Access-Control-Allow-Origin', req.headers.origin);
// Missing Vary header — CDN doesn't know to key cache by Origin

// ✅ Good
res.setHeader('Access-Control-Allow-Origin', req.headers.origin);
res.setHeader('Vary', 'Origin');
```

## Interview Questions

### 1. What is the difference between PUT and PATCH, and which is idempotent?

**Answer**: PUT replaces the entire resource at a given URL — you must send all fields, missing fields may be cleared or default to null. It is idempotent: calling `PUT /posts/1` with the same body twice leaves the server in the same state. PATCH makes a partial update — you send only the fields that change. PATCH is generally *not* guaranteed to be idempotent (e.g., a PATCH that increments a counter is not idempotent), though specific implementations can be. In practice, most REST APIs implement PATCH as idempotent JSON Merge Patch (`application/merge-patch+json`), but the spec doesn't require it. Use PUT when clients always have the full resource; use PATCH when clients make targeted edits.

### 2. What is the difference between 401 and 403?

**Answer**: 401 Unauthorized (misleadingly named) means the request lacks valid authentication credentials — the client hasn't identified itself. The server should include a `WWW-Authenticate` header indicating how to authenticate. The client should prompt for login. 403 Forbidden means the client is authenticated (the server knows who they are) but they don't have permission for the requested action. No further authentication will help — the user simply doesn't have access. A classic mistake is returning 403 for unauthenticated requests — this prevents clients from knowing they should prompt for login.

### 3. How does the ETag header enable efficient caching and optimistic locking?

**Answer**: An ETag is a version fingerprint of a resource (hash, version number, or timestamp). For caching: a client stores the ETag alongside a cached response and sends `If-None-Match: "etag-value"` on subsequent requests. If the resource hasn't changed, the server returns 304 Not Modified with no body — saving bandwidth. For optimistic locking: a client sends `If-Match: "etag-value"` with a PUT/PATCH. If the resource was modified by someone else since the client last read it, the server returns 412 Precondition Failed, preventing the "lost update" problem. The client must re-fetch and merge before retrying.

### 4. What CORS headers does a server need to include and why?

**Answer**: The minimum for a simple CORS request is `Access-Control-Allow-Origin: <origin>` (or `*` for public APIs). For pre-flight (non-simple requests like POST with JSON, or requests with Authorization header): `Access-Control-Allow-Methods` (comma-separated allowed methods), `Access-Control-Allow-Headers` (comma-separated allowed custom headers), `Access-Control-Max-Age` (how long browsers can cache the preflight result). For credentialed requests (cookies, Authorization): `Access-Control-Allow-Credentials: true` AND a specific origin (not `*`). Critical gotcha: when reflecting the request's Origin back, include `Vary: Origin` so CDNs don't cache a response for one origin and serve it to another.

### 5. When should you use 202 Accepted instead of 201 Created?

**Answer**: Use 202 Accepted when the request has been received and will be processed asynchronously — the work isn't done yet. Examples: triggering a report generation job, sending a bulk email campaign, processing a video upload. The response should include a way to check status: a `Location` header pointing to a job-status endpoint, or a body with `{ jobId, statusUrl, estimatedCompletion }`. Use 201 Created when the resource has actually been created by the time the response is sent. The distinction matters for clients implementing retry logic — a 201 means "your resource is ready," while a 202 means "we'll get to it."
