# Rate Limiting, Retries, and Exponential Backoff

## The Idea

**In plain English:** When your app asks a server for data and the server is too busy or says "slow down," rate limiting, retries, and backoff are the strategies your app uses to wait politely and try again without making the problem worse.

**Real-world analogy:** Imagine calling a popular pizza place on a Friday night. The line is busy, so you hang up and call back — but you wait a little longer each time instead of redialing instantly, so you're not clogging the line. And if dozens of people all call back at the exact same second, the phones jam again, so everyone waits a slightly different random amount of time before redialing.

- The busy phone line = a server that has hit its rate limit (429 Too Many Requests)
- Waiting longer before each redial = exponential backoff (doubling the wait time each attempt)
- Each person waiting a slightly different random amount = jitter (randomness added to spread out retries)

---

## Overview

Frontend applications make assumptions about API availability that fail in production: rate limits get hit, servers return 503 during deployments, flaky networks cause timeouts. This guide covers how rate limiting works from the client's perspective, how to implement resilient retry logic with exponential backoff and jitter, and how the circuit breaker pattern prevents cascading failures.

## Rate Limiting Basics

A rate limit caps how many requests a client can make within a time window. When exceeded, the server returns `429 Too Many Requests`.

```
Rate limiting strategies:

Fixed window:
  │▓▓▓▓▓▓▓▓▓▓│          │          │
  0s        60s         120s       180s
  10 req used  0 → 10 new  (cliff at boundary)

Sliding window:
  At any point, count requests in the last 60s
  Smoother — no burst at window boundary

Token bucket:
  Tokens refill at rate R/s, max capacity C
  Burst allowed up to C; sustained limited to R
  (most common in practice — allows short bursts)

Leaky bucket:
  Requests processed at fixed rate regardless of burst
  Excess queued or dropped
  (smooth output, good for protecting downstream)
```

### 429 Response Headers

```typescript
// Standard headers on a 429 response
HTTP/1.1 429 Too Many Requests
Retry-After: 30                    // seconds until limit resets
X-RateLimit-Limit: 100             // total allowed requests
X-RateLimit-Remaining: 0           // remaining in current window
X-RateLimit-Reset: 1718000000      // Unix timestamp of window reset
X-RateLimit-Policy: 100;w=60       // IETF draft: 100 per 60s window

// Some APIs use different header names
RateLimit-Limit: 100
RateLimit-Remaining: 0
RateLimit-Reset: 30
```

## Basic Retry Logic

The simplest retry — just try again N times:

```typescript
// ❌ Naive retry — hammers the server harder when it's already struggling
async function naiveRetry<T>(
  fn: () => Promise<T>,
  retries = 3
): Promise<T> {
  try {
    return await fn();
  } catch (err) {
    if (retries > 0) return naiveRetry(fn, retries - 1);
    throw err;
  }
}
```

Problems with naive retry:
1. Retries immediately — server still overloaded
2. Multiple clients all retry at the same time — synchronized thundering herd
3. Doesn't distinguish retryable errors from permanent failures

## Exponential Backoff

Wait progressively longer between retries. Each attempt doubles the wait time:

```typescript
// Retry attempt:  0     1     2      3       4
// Wait (ms):      0   1000  2000   4000    8000
// Formula: waitMs = baseMs * 2^attempt

interface RetryOptions {
  maxAttempts: number;
  baseDelayMs: number;
  maxDelayMs: number;
  retryOn?: (error: unknown) => boolean;
}

async function withExponentialBackoff<T>(
  fn: () => Promise<T>,
  options: RetryOptions
): Promise<T> {
  const { maxAttempts, baseDelayMs, maxDelayMs, retryOn } = options;

  for (let attempt = 0; attempt < maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      const isLastAttempt = attempt === maxAttempts - 1;
      const shouldRetry = retryOn ? retryOn(error) : isRetryableError(error);

      if (isLastAttempt || !shouldRetry) {
        throw error;
      }

      const delay = Math.min(baseDelayMs * 2 ** attempt, maxDelayMs);
      await sleep(delay);
    }
  }
  throw new Error('Unreachable');
}

function isRetryableError(error: unknown): boolean {
  if (error instanceof Response) {
    // Only retry transient server errors and rate limits
    return error.status === 429 || error.status >= 500;
  }
  if (error instanceof TypeError) {
    // Network error (no internet, DNS failure, CORS abort)
    return true;
  }
  return false;
}

const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));
```

## Jitter: Preventing Thundering Herd

Without jitter, all clients retry at the same moment after a failure, creating a synchronized spike that overwhelms the server again:

```
Without jitter:
  t=0   [FAIL]
  t=1s  [1000 clients retry simultaneously] → server overloaded again

With jitter:
  t=0   [FAIL]
  t=0.1s  [client A retries]
  t=0.7s  [client B retries]
  t=0.9s  [client C retries]
  → spread load; server recovers
```

### Full Jitter (Recommended)

```typescript
// Full jitter: random value in [0, exponential_backoff]
function getBackoffWithFullJitter(attempt: number, baseMs: number, maxMs: number): number {
  const exponential = Math.min(baseMs * 2 ** attempt, maxMs);
  return Math.random() * exponential;
}
```

### Equal Jitter

```typescript
// Equal jitter: half exponential + half random
// Guarantees minimum wait while still spreading load
function getBackoffWithEqualJitter(attempt: number, baseMs: number, maxMs: number): number {
  const cap = Math.min(baseMs * 2 ** attempt, maxMs);
  return cap / 2 + Math.random() * (cap / 2);
}
```

### Decorrelated Jitter (AWS recommendation)

```typescript
// AWS recommended: base randomized on previous sleep
function getDecorrelatedJitter(
  attempt: number,
  baseMs: number,
  maxMs: number,
  previousSleep: number
): number {
  return Math.min(maxMs, Math.random() * (previousSleep * 3 - baseMs) + baseMs);
}
```

## Respecting Retry-After

Always honor the server's `Retry-After` header — it's the server saying "try again no sooner than this":

```typescript
interface HttpError {
  status: number;
  headers: Headers;
  message: string;
}

async function fetchWithRetry(
  url: string,
  options?: RequestInit,
  maxAttempts = 4
): Promise<Response> {
  let lastError: unknown;
  let previousSleep = 1000;

  for (let attempt = 0; attempt < maxAttempts; attempt++) {
    try {
      const response = await fetch(url, options);

      if (response.ok) return response;

      // Don't retry client errors (except 429)
      if (response.status < 500 && response.status !== 429) {
        throw response;
      }

      if (attempt === maxAttempts - 1) throw response;

      // Calculate delay — Retry-After takes precedence
      const retryAfter = response.headers.get('Retry-After');
      let delayMs: number;

      if (retryAfter) {
        // Retry-After can be seconds or an HTTP-date
        const seconds = parseInt(retryAfter, 10);
        delayMs = isNaN(seconds)
          ? new Date(retryAfter).getTime() - Date.now()
          : seconds * 1000;
        // Cap at a reasonable maximum (10 minutes)
        delayMs = Math.min(delayMs, 10 * 60 * 1000);
      } else {
        // Fall back to jittered exponential backoff
        const cap = Math.min(1000 * 2 ** attempt, 30_000);
        delayMs = cap / 2 + Math.random() * (cap / 2);
      }

      lastError = response;
      previousSleep = delayMs;
      await sleep(delayMs);
    } catch (err) {
      if (err instanceof Response) throw err; // Don't retry non-retryable HTTP errors
      // Network error — retry with backoff
      const cap = Math.min(1000 * 2 ** attempt, 30_000);
      const delayMs = cap / 2 + Math.random() * (cap / 2);
      lastError = err;
      await sleep(delayMs);
    }
  }

  throw lastError;
}
```

## Circuit Breaker Pattern

The circuit breaker prevents an application from repeatedly calling a failing service. After a threshold of failures, it "opens" the circuit — subsequent calls fail immediately without hitting the network:

```
Circuit Breaker states:

  CLOSED ──────────────▶ calls pass through
    │  (below threshold)
    │  failure threshold exceeded
    ▼
  OPEN ─────────────────▶ calls fail immediately (no network hit)
    │  (half-open timer)
    │  timeout elapsed → try one probe request
    ▼
  HALF-OPEN ────────────▶ one request allowed through
    │  success → CLOSED   │  failure → OPEN
    ▼                     ▼
  CLOSED               OPEN (reset timer)
```

```typescript
type CircuitState = 'CLOSED' | 'OPEN' | 'HALF_OPEN';

interface CircuitBreakerOptions {
  failureThreshold: number;   // consecutive failures before opening
  successThreshold: number;   // consecutive successes before closing from half-open
  timeout: number;            // ms before attempting half-open from open
  onStateChange?: (from: CircuitState, to: CircuitState) => void;
}

class CircuitBreaker {
  private state: CircuitState = 'CLOSED';
  private failureCount = 0;
  private successCount = 0;
  private nextAttemptTime = 0;

  constructor(private options: CircuitBreakerOptions) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttemptTime) {
        throw new Error(`Circuit breaker OPEN — service unavailable`);
      }
      this.transition('HALF_OPEN');
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (err) {
      this.onFailure();
      throw err;
    }
  }

  private onSuccess(): void {
    this.failureCount = 0;
    if (this.state === 'HALF_OPEN') {
      this.successCount++;
      if (this.successCount >= this.options.successThreshold) {
        this.transition('CLOSED');
      }
    }
  }

  private onFailure(): void {
    this.failureCount++;
    this.successCount = 0;

    if (
      this.state === 'HALF_OPEN' ||
      this.failureCount >= this.options.failureThreshold
    ) {
      this.transition('OPEN');
      this.nextAttemptTime = Date.now() + this.options.timeout;
    }
  }

  private transition(to: CircuitState): void {
    const from = this.state;
    this.state = to;
    this.options.onStateChange?.(from, to);
  }

  getState(): CircuitState {
    return this.state;
  }
}

// Usage
const userServiceBreaker = new CircuitBreaker({
  failureThreshold: 5,
  successThreshold: 2,
  timeout: 30_000, // 30s before half-open probe
  onStateChange(from, to) {
    console.warn(`Circuit breaker: ${from} → ${to}`);
    metrics.increment(`circuit_breaker.${to.toLowerCase()}`);
  },
});

async function getUser(id: string): Promise<User> {
  return userServiceBreaker.execute(() =>
    fetchWithRetry(`/api/users/${id}`)
      .then((r) => r.json())
  );
}
```

## Frontend Retry Strategies

### React Query Built-in Retry

```typescript
import { useQuery, useMutation } from '@tanstack/react-query';

// Queries retry 3 times by default with exponential backoff
const { data } = useQuery({
  queryKey: ['posts'],
  queryFn: fetchPosts,
  retry: 3,
  retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30_000),
  // retryDelay: attemptIndex => attemptIndex * 1000  // linear
});

// Disable retry for mutations (user-triggered — don't silently retry)
const mutation = useMutation({
  mutationFn: createPost,
  retry: false,
  onError(error) {
    // Let the user decide to retry via UI
    toast.error('Failed to create post. Try again?');
  },
});

// Custom retry condition
const { data: criticalData } = useQuery({
  queryKey: ['config'],
  queryFn: fetchConfig,
  retry(failureCount, error) {
    // Only retry network errors, not 4xx responses
    if (error instanceof Response && error.status < 500) return false;
    return failureCount < 3;
  },
});
```

### Axios Interceptor with Retry

```typescript
import axios, { AxiosError, AxiosInstance } from 'axios';

function addRetryInterceptor(
  client: AxiosInstance,
  maxRetries = 3
): AxiosInstance {
  client.interceptors.response.use(
    (response) => response,
    async (error: AxiosError) => {
      const config = error.config as any;

      if (!config || !error.response) throw error;

      config._retryCount = config._retryCount || 0;

      const isRetryable =
        error.response.status === 429 || error.response.status >= 500;

      if (config._retryCount >= maxRetries || !isRetryable) throw error;

      config._retryCount++;

      const retryAfter = error.response.headers['retry-after'];
      const delay = retryAfter
        ? parseInt(retryAfter) * 1000
        : Math.min(1000 * 2 ** config._retryCount, 30_000) * (0.5 + Math.random() * 0.5);

      await sleep(delay);
      return client(config);
    }
  );

  return client;
}

const apiClient = addRetryInterceptor(
  axios.create({ baseURL: '/api', timeout: 10_000 })
);
```

## Idempotency Keys

Retries are only safe for idempotent operations. For mutations, use an idempotency key to prevent duplicate side effects:

```typescript
// Generate a stable key per logical request (not per HTTP attempt)
function generateIdempotencyKey(): string {
  return crypto.randomUUID();
}

async function createOrder(order: CreateOrderRequest): Promise<Order> {
  const idempotencyKey = generateIdempotencyKey();
  // Store key so retries use the same key
  sessionStorage.setItem('pendingOrderKey', idempotencyKey);

  return fetchWithRetry('/api/orders', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Idempotency-Key': idempotencyKey,
    },
    body: JSON.stringify(order),
  }).then((r) => {
    sessionStorage.removeItem('pendingOrderKey');
    return r.json();
  });
}
// Server uses the key to deduplicate: if it sees the same key again,
// it returns the same response without creating a second order.
```

## Common Mistakes

### 1. Retrying Non-Idempotent Operations Without Idempotency Keys

```typescript
// ❌ Retrying a POST without idempotency key
// If the server processed the request but network failed before response,
// retry creates a duplicate order/payment/email
async function submitPayment(data: PaymentData) {
  return fetchWithRetry('/api/payments', {
    method: 'POST',
    body: JSON.stringify(data),
    // Missing: 'Idempotency-Key' header
  });
}

// ✅ Generate key once before the retry loop
async function submitPayment(data: PaymentData) {
  const key = crypto.randomUUID();
  return fetchWithRetry('/api/payments', {
    method: 'POST',
    headers: { 'Idempotency-Key': key },
    body: JSON.stringify(data),
  });
}
```

### 2. Retrying 4xx Errors

```typescript
// ❌ Retrying 400/401/403/404 — they won't become 200 on retry
function isRetryable(status: number): boolean {
  return status >= 400; // WRONG — retries auth failures, validation errors
}

// ✅ Only retry transient errors
function isRetryable(status: number): boolean {
  return status === 429 || status >= 500;
}
```

### 3. No Maximum Backoff Cap

```typescript
// ❌ Uncapped backoff — attempt 20 would wait 2^20s ≈ 12 days
const delay = 1000 * 2 ** attempt;

// ✅ Cap at a reasonable max
const delay = Math.min(1000 * 2 ** attempt, 60_000); // max 60s
```

### 4. Ignoring Request Timeouts

```typescript
// ❌ No timeout — hanging requests block retry logic indefinitely
const response = await fetch('/api/data');

// ✅ AbortController with timeout, then retry
async function fetchWithTimeout(url: string, timeoutMs = 5000) {
  const controller = new AbortController();
  const id = setTimeout(() => controller.abort(), timeoutMs);
  try {
    return await fetch(url, { signal: controller.signal });
  } finally {
    clearTimeout(id);
  }
}
```

### 5. Retrying in Parallel

```typescript
// ❌ Firing multiple concurrent retries
const results = await Promise.all(
  items.map((item) => fetchWithRetry(`/api/items/${item.id}`))
);
// If all fail and all retry simultaneously, thundering herd

// ✅ Either limit concurrency or stagger start times
import pLimit from 'p-limit';
const limit = pLimit(5); // max 5 concurrent
const results = await Promise.all(
  items.map((item) => limit(() => fetchWithRetry(`/api/items/${item.id}`)))
);
```

## Interview Questions

### 1. What is exponential backoff and why is jitter necessary?

**Answer:** Exponential backoff waits progressively longer between retry attempts — each wait doubles (or grows by some factor). This prevents hammering an already-struggling server on every retry. Jitter adds randomness to the backoff delay. Without jitter, all clients that received a failure at time T will retry at T+1s, T+2s, T+4s simultaneously — creating synchronized spikes that re-overwhelm the server (thundering herd). With jitter, each client's retry times are spread randomly across the backoff window, distributing the load. AWS recommends "full jitter" — a random value between 0 and the exponential cap.

### 2. When should you not retry a failed request?

**Answer:** Never retry 4xx client errors (except 429) — a 400 Bad Request, 401 Unauthorized, 403 Forbidden, or 404 Not Found won't become a 200 on retry because the problem is with the request, not the server's availability. Don't retry user-triggered mutations (POST/PUT/DELETE) without idempotency keys — you risk duplicate operations (double charges, double orders). Don't retry if the user has navigated away or the component unmounted (use AbortController). Don't retry if the circuit breaker is open. Only retry transient failures: 429 (rate limit), 5xx (server errors), and network errors (timeout, DNS failure).

### 3. What is the circuit breaker pattern and when does it apply on the frontend?

**Answer:** The circuit breaker tracks consecutive failures when calling a service. After a threshold, it "opens" — subsequent calls fail immediately without hitting the network (fail fast). After a timeout, it enters "half-open" and probes with one request. If successful, it closes again. On the frontend, circuit breakers make sense when calling third-party APIs, microservices, or any endpoint where repeated failures have cost (battery/CPU on mobile, user-visible spinners). Without a circuit breaker, a dead service causes every component that calls it to hang, show errors, and waste resources. The circuit breaker degrades gracefully with cached/default data instead.

### 4. How do idempotency keys prevent duplicate operations during retries?

**Answer:** An idempotency key is a unique identifier generated once per logical request (not per HTTP attempt) and sent in a header (e.g., `Idempotency-Key: uuid`). The server stores the key and its response. If the same key arrives again (retry after network failure), the server returns the stored response without re-executing the operation. This makes non-idempotent operations (POST) safe to retry. The key must be generated before the retry loop, stored on the client, and remain stable across all retry attempts. Stripe, Adyen, and most payment APIs implement this pattern.

### 5. How would you implement a rate-limit-aware API client?

**Answer:** Intercept all responses at a central point (Axios interceptor, fetch wrapper). On 429, read the `Retry-After` header to determine the mandatory wait time (in seconds or as an HTTP date). Honor that time rather than using backoff — the server has specified exactly when to retry. Optionally implement a client-side token bucket that tracks the rate limit headers (`X-RateLimit-Remaining`, `X-RateLimit-Reset`) and proactively queues requests before they hit the limit. For high-throughput scenarios, a request queue with a rate limiter (e.g., `p-throttle`) prevents bursts from ever reaching 429.
