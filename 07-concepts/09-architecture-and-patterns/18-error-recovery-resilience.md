# Error Recovery and Resilience Patterns

## The Idea

**In plain English:** Error recovery and resilience is about building apps that gracefully handle failures — like a network request failing or a third-party service going down — so the rest of the app keeps working instead of everything breaking at once. "Resilience" means your app can absorb problems and bounce back without the user noticing (or at least without them losing their work).

**Real-world analogy:** Think of a hospital with separate wings for different departments. If there's a flood in the radiology wing, the emergency room, surgery, and pharmacy all keep running normally — the problem is contained to one section.

- The hospital wing = a UI component or feature section
- The flood in one wing = a bug or crashed component
- The separate wings that keep running = other components wrapped in their own error boundaries
- The emergency backup procedures = fallback UIs and graceful degradation strategies

---

## Overview

Resilience engineering applies to frontend just as it does to backend. A resilient frontend limits blast radius when things go wrong: a failed API call doesn't crash the page, a third-party script error doesn't block checkout, and a temporary outage recovers automatically without user intervention.

---

## Circuit Breaker Pattern

A circuit breaker tracks failure rate on a dependency and "opens" (stops calling it) when failures exceed a threshold, giving the downstream service time to recover.

**States:**
- **Closed** — normal operation, failures tracked
- **Open** — all calls fail immediately (fast fail), no network traffic
- **Half-open** — one probe request allowed; success closes, failure re-opens

```typescript
type CircuitState = 'closed' | 'open' | 'half-open';

interface CircuitBreakerOptions {
  failureThreshold: number;  // failures before opening
  successThreshold: number;  // successes in half-open before closing
  timeout: number;           // ms before transitioning open → half-open
}

class CircuitBreaker {
  private state: CircuitState = 'closed';
  private failures = 0;
  private successes = 0;
  private nextAttempt = 0;

  constructor(private options: CircuitBreakerOptions) {}

  async call<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() < this.nextAttempt) {
        throw new CircuitOpenError('Circuit is open');
      }
      this.state = 'half-open';
      this.successes = 0;
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
    this.failures = 0;
    if (this.state === 'half-open') {
      this.successes++;
      if (this.successes >= this.options.successThreshold) {
        this.state = 'closed';
      }
    }
  }

  private onFailure(): void {
    this.failures++;
    if (
      this.state === 'half-open' ||
      this.failures >= this.options.failureThreshold
    ) {
      this.state = 'open';
      this.nextAttempt = Date.now() + this.options.timeout;
    }
  }

  get isOpen(): boolean { return this.state === 'open'; }
}

class CircuitOpenError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'CircuitOpenError';
  }
}

// Usage
const paymentBreaker = new CircuitBreaker({
  failureThreshold: 5,
  successThreshold: 2,
  timeout: 30_000,
});

async function processPayment(data: PaymentData) {
  try {
    return await paymentBreaker.call(() => paymentApi.charge(data));
  } catch (err) {
    if (err instanceof CircuitOpenError) {
      // Show "service temporarily unavailable" — don't retry
      showCircuitOpenMessage();
    } else {
      throw err;
    }
  }
}
```

---

## Retry with Exponential Backoff and Jitter

Blind retries at fixed intervals cause thundering herd problems. Exponential backoff + jitter spreads retries out.

```typescript
interface RetryOptions {
  maxAttempts: number;
  baseDelay: number;   // ms
  maxDelay: number;    // ms cap
  jitter: boolean;
  shouldRetry?: (error: unknown, attempt: number) => boolean;
}

async function withRetry<T>(
  fn: () => Promise<T>,
  opts: RetryOptions
): Promise<T> {
  const { maxAttempts, baseDelay, maxDelay, jitter, shouldRetry } = opts;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxAttempts) throw error;
      if (shouldRetry && !shouldRetry(error, attempt)) throw error;

      const exponential = Math.min(baseDelay * 2 ** (attempt - 1), maxDelay);
      const delay = jitter
        ? exponential * (0.5 + Math.random() * 0.5) // 50–100% of exponential
        : exponential;

      await sleep(delay);
    }
  }
  throw new Error('unreachable');
}

const sleep = (ms: number) => new Promise(r => setTimeout(r, ms));

// Usage: retry only on transient errors
const data = await withRetry(
  () => fetch('/api/data').then(r => {
    if (!r.ok) throw Object.assign(new Error('HTTP error'), { status: r.status });
    return r.json();
  }),
  {
    maxAttempts: 3,
    baseDelay: 500,
    maxDelay: 5000,
    jitter: true,
    shouldRetry: (err: any) => err.status >= 500 || err.status === 429,
  }
);
```

---

## Bulkhead Pattern

Isolate failures in one part of the UI so they can't cascade to others. In browsers, the key mechanisms are:

**Error Boundaries (React):**

```tsx
class Bulkhead extends React.Component<
  { name: string; fallback: React.ReactNode; children: React.ReactNode },
  { error: Error | null }
> {
  state = { error: null };

  static getDerivedStateFromError(error: Error) {
    return { error };
  }

  componentDidCatch(error: Error, info: React.ErrorInfo) {
    // Report to Sentry without crashing the parent
    reportError(error, { boundary: this.props.name, ...info });
  }

  render() {
    if (this.state.error) return this.props.fallback;
    return this.props.children;
  }
}

// Isolate each dashboard widget
function Dashboard() {
  return (
    <div className="grid">
      <Bulkhead name="revenue-chart" fallback={<ChartError />}>
        <RevenueChart />
      </Bulkhead>
      <Bulkhead name="user-table" fallback={<TableError />}>
        <UserTable />
      </Bulkhead>
      <Bulkhead name="notifications" fallback={null}>
        <NotificationPanel />
      </Bulkhead>
    </div>
  );
}
```

**Angular: Component-level error handling:**

```typescript
@Component({
  selector: 'app-bulkhead',
  template: `
    @if (error) {
      <app-error-fallback [error]="error" />
    } @else {
      <ng-content />
    }
  `,
})
export class BulkheadComponent implements OnInit {
  @Input() name = '';
  error: Error | null = null;

  // Catch errors from child components via ErrorHandler
}

@Injectable()
export class BoundaryErrorHandler implements ErrorHandler {
  handleError(error: any): void {
    // Log and contain — don't rethrow
    console.error('[BulkheadErrorHandler]', error);
    reportToSentry(error);
  }
}
```

---

## Graceful Degradation

Provide a reduced but functional experience when a feature fails:

```typescript
// Payment: primary → fallback → manual
async function checkout(cart: Cart) {
  try {
    return await stripePayment(cart);
  } catch {
    try {
      // Fallback to PayPal
      return await paypalPayment(cart);
    } catch {
      // Manual fallback: redirect to phone order
      showManualCheckoutInstructions();
    }
  }
}

// Search: instant → debounced → full-page
class SearchController {
  async search(query: string) {
    try {
      return await instantSearch(query);      // WebSocket live results
    } catch {
      try {
        return await fetchSearch(query);      // REST API fallback
      } catch {
        window.location.href = `/search?q=${encodeURIComponent(query)}`; // hard nav
      }
    }
  }
}

// Real-time → polling fallback
class DataFeed {
  private ws: WebSocket | null = null;
  private pollInterval: number | null = null;

  connect(url: string, onData: (d: unknown) => void): void {
    try {
      this.ws = new WebSocket(url);
      this.ws.onmessage = (e) => onData(JSON.parse(e.data));
      this.ws.onerror = () => {
        this.ws = null;
        this.startPolling(url, onData); // degrade to polling
      };
    } catch {
      this.startPolling(url, onData);
    }
  }

  private startPolling(url: string, onData: (d: unknown) => void): void {
    this.pollInterval = window.setInterval(async () => {
      const data = await fetch(url).then(r => r.json()).catch(() => null);
      if (data) onData(data);
    }, 5000);
  }
}
```

---

## Timeout Pattern

Never let a network call hang indefinitely:

```typescript
function withTimeout<T>(promise: Promise<T>, ms: number): Promise<T> {
  const timeout = new Promise<never>((_, reject) =>
    setTimeout(() => reject(new TimeoutError(`Timed out after ${ms}ms`)), ms)
  );
  return Promise.race([promise, timeout]);
}

class TimeoutError extends Error {
  constructor(msg: string) { super(msg); this.name = 'TimeoutError'; }
}

// With AbortController (cancels the fetch too, not just the promise)
async function fetchWithTimeout(url: string, ms: number): Promise<Response> {
  const ctrl = new AbortController();
  const id    = setTimeout(() => ctrl.abort(), ms);
  try {
    return await fetch(url, { signal: ctrl.signal });
  } finally {
    clearTimeout(id);
  }
}

// Usage
try {
  const data = await fetchWithTimeout('/api/heavy', 3000);
} catch (err) {
  if (err instanceof DOMException && err.name === 'AbortError') {
    showTimeoutMessage();
  }
}
```

---

## Stale-While-Revalidate (Cache-First Resilience)

Show cached data immediately; update in background; never show a blank state:

```typescript
class ResilientCache<T> {
  private cache = new Map<string, { data: T; ts: number }>();

  async fetch(
    key: string,
    fetcher: () => Promise<T>,
    staleMs = 60_000
  ): Promise<T> {
    const cached = this.cache.get(key);

    if (cached) {
      const age = Date.now() - cached.ts;
      if (age < staleMs) return cached.data; // fresh

      // Stale: return immediately, revalidate in background
      this.revalidate(key, fetcher);
      return cached.data;
    }

    // No cache: must wait
    return this.revalidate(key, fetcher);
  }

  private async revalidate(key: string, fetcher: () => Promise<T>): Promise<T> {
    try {
      const data = await fetcher();
      this.cache.set(key, { data, ts: Date.now() });
      return data;
    } catch (err) {
      const cached = this.cache.get(key);
      if (cached) return cached.data; // serve stale on error
      throw err;
    }
  }
}
```

---

## Frontend Chaos Engineering

Deliberately inject failures to verify your resilience patterns hold:

```typescript
// Development-only chaos layer
class ChaosProxy {
  constructor(
    private api: ApiClient,
    private options = {
      failureRate:   0.2,  // 20% of calls fail
      latencyMs:     1000, // add 1s delay
      enabled:       process.env.NODE_ENV === 'development',
    }
  ) {}

  async get<T>(path: string): Promise<T> {
    if (!this.options.enabled) return this.api.get(path);

    await sleep(this.options.latencyMs * Math.random());

    if (Math.random() < this.options.failureRate) {
      throw new Error(`[Chaos] Simulated failure for ${path}`);
    }

    return this.api.get(path);
  }
}
```

---

## Monitoring Resilience in Production

Resilience patterns are only useful if you know when they activate:

```typescript
// Log when circuit opens — signals sustained downstream failure
class InstrumentedCircuitBreaker extends CircuitBreaker {
  protected override onStateChange(from: CircuitState, to: CircuitState): void {
    analytics.track('circuit_breaker_state_change', {
      name: this.name,
      from,
      to,
      timestamp: Date.now(),
    });
    if (to === 'open') {
      alertOncall(`Circuit ${this.name} opened`);
    }
  }
}

// Track retry attempts
async function trackedRetry<T>(fn: () => Promise<T>, opts: RetryOptions) {
  let attempt = 0;
  return withRetry(async () => {
    attempt++;
    if (attempt > 1) {
      analytics.track('retry_attempt', { attempt, name: opts.name });
    }
    return fn();
  }, opts);
}
```

---

## Interview Questions

**Q: What is a circuit breaker in a frontend context?**  
A: It wraps a dependency (an API, a third-party service) and tracks the failure rate. When failures exceed a threshold it "opens" and all calls fail immediately without hitting the network — giving the downstream service time to recover and preventing the user from waiting on guaranteed failures. After a timeout it enters "half-open" to probe if the service recovered.

**Q: What's the difference between a retry and a circuit breaker?**  
A: Retries handle transient failures (a single request that failed). Circuit breakers handle sustained failures (the downstream service is down). Using retries alone during a sustained outage amplifies load on the failing service; a circuit breaker stops sending traffic entirely.

**Q: How do you isolate a failing widget so it doesn't crash the whole page?**  
A: Use error boundaries (React) or equivalent component-level error handling (Angular). Each independently-failing section is wrapped in its own boundary with a degraded fallback UI. The parent and sibling sections are unaffected.

**Q: What is stale-while-revalidate and when is it a resilience technique?**  
A: It's a caching strategy where you immediately return cached data while fetching an update in the background. As a resilience technique: if the background fetch fails, you continue serving the stale data rather than showing an error — the user sees slightly old data rather than a broken state.
