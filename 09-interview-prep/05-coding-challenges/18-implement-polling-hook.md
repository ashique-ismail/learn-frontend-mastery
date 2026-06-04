# Implement Polling Hook with Exponential Backoff

## The Core Pattern

Use `setTimeout` (not `setInterval`) so the next poll only starts after the current one completes — preventing overlapping requests.

```ts
function usePolling(
  fn: () => Promise<void>,
  interval: number,
  enabled = true
) {
  useEffect(() => {
    if (!enabled) return;
    let timeoutId: number;
    let stopped = false;

    async function poll() {
      try {
        await fn();
      } finally {
        if (!stopped) {
          timeoutId = window.setTimeout(poll, interval);
        }
      }
    }

    poll(); // start immediately
    return () => {
      stopped = true;
      clearTimeout(timeoutId);
    };
  }, [fn, interval, enabled]);
}
```

---

## Full Hook with Exponential Backoff

```ts
interface PollingOptions {
  interval: number;     // base interval (ms)
  maxInterval?: number; // cap for backoff (ms)
  maxRetries?: number;  // max consecutive errors before stopping
  enabled?: boolean;
  onError?: (error: Error, retryCount: number) => void;
}

function usePollingWithBackoff<T>(
  fetchFn: () => Promise<T>,
  onData: (data: T) => void,
  options: PollingOptions
) {
  const {
    interval,
    maxInterval = 60_000,
    maxRetries = 5,
    enabled = true,
    onError,
  } = options;

  const fnRef = useRef(fetchFn);
  const onDataRef = useRef(onData);
  fnRef.current = fetchFn;
  onDataRef.current = onData;

  useEffect(() => {
    if (!enabled) return;

    let timeoutId: number;
    let stopped = false;
    let consecutiveErrors = 0;

    async function poll() {
      try {
        const data = await fnRef.current();
        consecutiveErrors = 0; // reset on success
        onDataRef.current(data);
        scheduleNext(interval); // back to base interval
      } catch (error) {
        consecutiveErrors++;
        onError?.(error as Error, consecutiveErrors);

        if (consecutiveErrors >= maxRetries) {
          console.error('Polling stopped after max retries');
          return;
        }

        // Exponential backoff with jitter
        const backoffDelay = Math.min(
          interval * Math.pow(2, consecutiveErrors - 1),
          maxInterval
        );
        const jitter = Math.random() * 1000;
        scheduleNext(backoffDelay + jitter);
      }
    }

    function scheduleNext(delay: number) {
      if (!stopped) {
        timeoutId = window.setTimeout(poll, delay);
      }
    }

    poll(); // first call immediately
    return () => {
      stopped = true;
      clearTimeout(timeoutId);
    };
  }, [interval, maxInterval, maxRetries, enabled]);
}
```

---

## Visibility-Based Pausing

Don't poll when the tab is hidden — saves bandwidth and server resources:

```ts
function useVisibilityAwarePolling(fetchFn: () => Promise<void>, interval: number) {
  const [isVisible, setIsVisible] = useState(!document.hidden);

  useEffect(() => {
    function handleVisibilityChange() {
      setIsVisible(!document.hidden);
    }
    document.addEventListener('visibilitychange', handleVisibilityChange);
    return () => document.removeEventListener('visibilitychange', handleVisibilityChange);
  }, []);

  // Poll only when tab is visible
  usePolling(fetchFn, interval, isVisible);
}
```

---

## Usage: Live Order Status

```tsx
function OrderTracker({ orderId }: { orderId: string }) {
  const [order, setOrder] = useState<Order | null>(null);
  const [error, setError] = useState<string | null>(null);

  const fetchOrder = useCallback(() => api.getOrder(orderId), [orderId]);

  usePollingWithBackoff(
    fetchOrder,
    setOrder,
    {
      interval: 5000,       // poll every 5 seconds normally
      maxInterval: 30_000,  // max 30 seconds between retries
      maxRetries: 5,
      enabled: order?.status !== 'delivered', // stop when delivered
      onError: (err, count) => {
        if (count >= 3) setError(`Connection issues (retry ${count})`);
      },
    }
  );

  if (!order) return <Spinner />;
  return <OrderStatus order={order} error={error} />;
}
```

---

## Backoff Intervals

```
Error 1: 5s * 2^0 = 5s  + jitter
Error 2: 5s * 2^1 = 10s + jitter
Error 3: 5s * 2^2 = 20s + jitter
Error 4: 5s * 2^3 = 40s + jitter
Error 5: max 60s         → stop
```

Jitter prevents the **thundering herd problem** — multiple clients backing off and retrying at exactly the same time, creating a new spike of requests.

---

## Common Interview Questions

**Q: Why `setTimeout` instead of `setInterval` for polling?**
`setInterval` fires regardless of whether the previous request completed. If the request takes longer than the interval, requests overlap. `setTimeout` chained in the callback ensures sequential calls.

**Q: What's exponential backoff for?**
When a server is under load or temporarily unavailable, exponential backoff reduces client pressure. Instead of hammering every 5 seconds, wait 5s, 10s, 20s, 40s... This gives the server time to recover.

**Q: What does jitter do?**
Without jitter, all clients would retry at the same intervals (5s, 10s, 20s...) creating synchronized bursts. Random jitter spreads the retries across time, reducing peak load.
