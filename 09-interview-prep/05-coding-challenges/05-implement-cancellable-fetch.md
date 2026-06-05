# Implement Cancellable Fetch Hook

## The Idea

**In plain English:** A cancellable fetch is a way to request data from the internet and then stop that request mid-way if you no longer need it — for example, because the user navigated away. "Fetching" means asking a server for data; "cancelling" means telling the browser to drop that request before it finishes.

**Real-world analogy:** Imagine you call a pizza delivery place and order a pizza, but while it's being made you decide to leave the house. You call back and cancel the order so the driver doesn't show up at an empty house.

- The phone call to order = the `fetch()` request sent to the server
- Calling back to cancel = calling `controller.abort()` to stop the request
- The driver arriving at an empty house = `setState` being called after the component has already unmounted (a wasted, potentially harmful action)

---

## Why Cancellation Matters

Without cancellation:
1. Component mounts, starts fetching
2. Component unmounts before fetch completes
3. Fetch completes, `setState` called on unmounted component
4. React warning: "Can't perform a state update on an unmounted component"
5. Memory leak (stale closure holds component reference)

With cancellation: the request is aborted when the component unmounts.

---

## `AbortController` Basics

```ts
const controller = new AbortController();

fetch('/api/data', { signal: controller.signal })
  .then(res => res.json())
  .catch(err => {
    if (err.name === 'AbortError') {
      // Request was cancelled — don't treat as error
    } else {
      throw err; // real error
    }
  });

controller.abort(); // cancel the request
```

---

## `useFetch` Hook

```ts
type FetchState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

function useFetch<T>(url: string | null) {
  const [state, setState] = useState<FetchState<T>>({ status: 'idle' });

  useEffect(() => {
    if (!url) return;

    const controller = new AbortController();
    setState({ status: 'loading' });

    fetch(url, { signal: controller.signal })
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json() as Promise<T>;
      })
      .then(data => {
        setState({ status: 'success', data });
      })
      .catch(err => {
        if (err.name === 'AbortError') return; // cancelled — do nothing
        setState({ status: 'error', error: err });
      });

    // Cleanup: cancel request on unmount OR when url changes
    return () => controller.abort();
  }, [url]);

  return state;
}
```

---

## Usage

```tsx
function UserProfile({ userId }: { userId: string }) {
  const state = useFetch<User>(`/api/users/${userId}`);

  if (state.status === 'idle' || state.status === 'loading') return <Spinner />;
  if (state.status === 'error') return <Error message={state.error.message} />;

  const user = state.data;
  return <div>{user.name}</div>;
}
```

When `userId` changes, the previous request is automatically cancelled. When the component unmounts, the in-flight request is cancelled.

---

## Generic Version with TypeScript

```ts
interface UseFetchOptions<T> {
  transform?: (data: unknown) => T;
  onSuccess?: (data: T) => void;
  onError?: (error: Error) => void;
  enabled?: boolean;
}

function useFetch<T>(url: string | null, options: UseFetchOptions<T> = {}) {
  const { transform = (d) => d as T, onSuccess, onError, enabled = true } = options;
  const [state, setState] = useState<FetchState<T>>({ status: 'idle' });

  useEffect(() => {
    if (!url || !enabled) {
      setState({ status: 'idle' });
      return;
    }

    const controller = new AbortController();
    setState({ status: 'loading' });

    fetch(url, { signal: controller.signal })
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}: ${res.statusText}`);
        return res.json();
      })
      .then(raw => {
        const data = transform(raw);
        setState({ status: 'success', data });
        onSuccess?.(data);
      })
      .catch(err => {
        if (err.name === 'AbortError') return;
        setState({ status: 'error', error: err });
        onError?.(err);
      });

    return () => controller.abort();
  }, [url, enabled]); // Note: callbacks are excluded from deps to avoid re-fetch loops

  return state;
}
```

---

## Cancelling with a Cleanup Function (Async/Await Style)

```ts
useEffect(() => {
  let cancelled = false;
  const controller = new AbortController();

  async function load() {
    try {
      const res = await fetch(url, { signal: controller.signal });
      const data = await res.json();
      if (!cancelled) setState({ status: 'success', data }); // guard
    } catch (err) {
      if (!cancelled && err.name !== 'AbortError') {
        setState({ status: 'error', error: err });
      }
    }
  }

  load();

  return () => {
    cancelled = true;
    controller.abort();
  };
}, [url]);
```

The `cancelled` flag guards against the race condition where `abort()` doesn't cancel fast enough (the response was already received but the `.then()` hadn't run yet).

---

## Common Interview Questions

**Q: Does `AbortController.abort()` cancel a response that's already been received?**
If `fetch()` has already resolved and you're in `.then()` processing, `abort()` won't stop that synchronous code. The `cancelled` boolean flag handles this edge case.

**Q: Can you reuse an `AbortController` after aborting?**
No — once aborted, it stays aborted. Create a new `AbortController` for each request.

**Q: How does this relate to React 18's strict mode double invocation?**
In strict mode (dev only), effects run twice (mount → unmount → mount again). Without cancellation, you'd get two concurrent requests. The cleanup function ensures the first request is cancelled when the effect re-runs.
