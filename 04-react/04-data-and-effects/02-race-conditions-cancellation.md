# Race Conditions and Cancellation

## Overview

Race conditions occur when multiple asynchronous operations compete, and the order of completion can lead to inconsistent or incorrect application state. In React applications, race conditions commonly happen with data fetching, where a newer request completes before an older one, potentially displaying stale data.

Understanding and preventing race conditions is critical for building reliable React applications, especially those with complex data fetching patterns.

## Core Concepts

### What is a Race Condition?

A race condition in React typically occurs when:

1. **Multiple async operations** are triggered in sequence
2. **Completion order is unpredictable** due to network timing
3. **Later operations complete first**, but earlier ones overwrite results
4. **Stale data** is displayed to users

### Common Scenarios

```javascript
// Race condition example
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    // Problem: If userId changes quickly, requests may complete out of order
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => setUser(data)); // May set stale data!
  }, [userId]);

  return <div>{user?.name}</div>;
}

// Scenario:
// 1. User clicks profile for userId=1 (slow request - 2 seconds)
// 2. User clicks profile for userId=2 (fast request - 0.5 seconds)
// 3. Request 2 completes, shows user 2
// 4. Request 1 completes, shows user 1 (WRONG!)
```

## Prevention Strategies

### 1. Cleanup with Boolean Flag

```javascript
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    let isActive = true; // Cleanup flag

    setLoading(true);
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => {
        if (isActive) { // Only update if still active
          setUser(data);
          setLoading(false);
        }
      });

    return () => {
      isActive = false; // Mark as inactive on cleanup
    };
  }, [userId]);

  return loading ? <div>Loading...</div> : <div>{user?.name}</div>;
}
```

### 2. AbortController (Modern Approach)

```javascript
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    const controller = new AbortController();

    setLoading(true);
    setError(null);

    fetch(`/api/users/${userId}`, {
      signal: controller.signal, // Pass abort signal
    })
      .then(res => res.json())
      .then(data => {
        setUser(data);
        setLoading(false);
      })
      .catch(err => {
        if (err.name === 'AbortError') {
          // Request was cancelled, don't update state
          console.log('Request cancelled');
        } else {
          setError(err.message);
          setLoading(false);
        }
      });

    return () => {
      controller.abort(); // Cancel request on cleanup
    };
  }, [userId]);

  if (error) return <div>Error: {error}</div>;
  if (loading) return <div>Loading...</div>;
  return <div>{user?.name}</div>;
}
```

### 3. Request Versioning

```javascript
function SearchResults({ query }) {
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    let requestVersion = Date.now(); // Unique identifier for this request
    const currentVersion = requestVersion;

    setLoading(true);

    fetch(`/api/search?q=${query}`)
      .then(res => res.json())
      .then(data => {
        // Only update if this is still the latest request
        if (requestVersion === currentVersion) {
          setResults(data);
          setLoading(false);
        }
      });

    return () => {
      requestVersion = null; // Invalidate this version
    };
  }, [query]);

  return (
    <div>
      {loading && <div>Loading...</div>}
      <ul>
        {results.map(result => (
          <li key={result.id}>{result.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

### 4. Debouncing to Reduce Requests

```javascript
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

function SearchResults({ query }) {
  const [results, setResults] = useState([]);
  const debouncedQuery = useDebounce(query, 500); // Wait 500ms

  useEffect(() => {
    if (!debouncedQuery) return;

    const controller = new AbortController();

    fetch(`/api/search?q=${debouncedQuery}`, {
      signal: controller.signal,
    })
      .then(res => res.json())
      .then(setResults)
      .catch(err => {
        if (err.name !== 'AbortError') {
          console.error(err);
        }
      });

    return () => controller.abort();
  }, [debouncedQuery]);

  return (
    <ul>
      {results.map(result => (
        <li key={result.id}>{result.title}</li>
      ))}
    </ul>
  );
}
```

## Advanced Patterns

### Custom useFetch Hook with Cancellation

```javascript
function useFetch(url, options = {}) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    if (!url) return;

    const controller = new AbortController();
    let isActive = true;

    const fetchData = async () => {
      setLoading(true);
      setError(null);

      try {
        const response = await fetch(url, {
          ...options,
          signal: controller.signal,
        });

        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }

        const result = await response.json();

        if (isActive) {
          setData(result);
          setLoading(false);
        }
      } catch (err) {
        if (isActive && err.name !== 'AbortError') {
          setError(err.message);
          setLoading(false);
        }
      }
    };

    fetchData();

    return () => {
      isActive = false;
      controller.abort();
    };
  }, [url, JSON.stringify(options)]);

  return { data, loading, error };
}

// Usage
function UserProfile({ userId }) {
  const { data: user, loading, error } = useFetch(
    `/api/users/${userId}`
  );

  if (error) return <div>Error: {error}</div>;
  if (loading) return <div>Loading...</div>;
  return <div>{user?.name}</div>;
}
```

### Race-Safe Mutation Hook

```javascript
function useRaceSafeMutation(mutationFn) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const latestCallId = useRef(0);

  const mutate = useCallback(
    async (...args) => {
      const callId = ++latestCallId.current;

      setLoading(true);
      setError(null);

      try {
        const result = await mutationFn(...args);

        // Only update if this is still the latest call
        if (callId === latestCallId.current) {
          setData(result);
          setLoading(false);
        }

        return result;
      } catch (err) {
        if (callId === latestCallId.current) {
          setError(err.message);
          setLoading(false);
        }
        throw err;
      }
    },
    [mutationFn]
  );

  return { mutate, data, loading, error };
}

// Usage
function UpdateProfileForm() {
  const { mutate: updateProfile, loading, error } = useRaceSafeMutation(
    async (formData) => {
      const res = await fetch('/api/profile', {
        method: 'PUT',
        body: JSON.stringify(formData),
      });
      return res.json();
    }
  );

  const handleSubmit = async (e) => {
    e.preventDefault();
    const formData = new FormData(e.target);
    await updateProfile(Object.fromEntries(formData));
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="name" />
      <button type="submit" disabled={loading}>
        {loading ? 'Saving...' : 'Save'}
      </button>
      {error && <div>Error: {error}</div>}
    </form>
  );
}
```

### Automatic Retry with Cancellation

```javascript
function useFetchWithRetry(url, maxRetries = 3) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [retryCount, setRetryCount] = useState(0);

  useEffect(() => {
    if (!url) return;

    const controller = new AbortController();
    let isActive = true;
    let retries = 0;

    const fetchData = async () => {
      setLoading(true);

      while (retries <= maxRetries && isActive) {
        try {
          const response = await fetch(url, {
            signal: controller.signal,
          });

          if (!response.ok) {
            throw new Error(`HTTP ${response.status}`);
          }

          const result = await response.json();

          if (isActive) {
            setData(result);
            setError(null);
            setLoading(false);
            setRetryCount(retries);
          }
          return;
        } catch (err) {
          if (err.name === 'AbortError') {
            return; // Request cancelled
          }

          retries++;

          if (retries > maxRetries) {
            if (isActive) {
              setError(err.message);
              setLoading(false);
              setRetryCount(retries - 1);
            }
            return;
          }

          // Exponential backoff
          await new Promise(resolve =>
            setTimeout(resolve, Math.min(1000 * 2 ** retries, 10000))
          );
        }
      }
    };

    fetchData();

    return () => {
      isActive = false;
      controller.abort();
    };
  }, [url, maxRetries]);

  return { data, loading, error, retryCount };
}

// Usage
function UserProfile({ userId }) {
  const { data: user, loading, error, retryCount } = useFetchWithRetry(
    `/api/users/${userId}`,
    3
  );

  if (error) return <div>Error after {retryCount} retries: {error}</div>;
  if (loading) return <div>Loading... (attempt {retryCount + 1})</div>;
  return <div>{user?.name}</div>;
}
```

## Real-World Patterns

### Typeahead Search with Race Protection

```javascript
function TypeaheadSearch() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);
  const debouncedQuery = useDebounce(query, 300);
  const abortControllerRef = useRef(null);

  useEffect(() => {
    if (!debouncedQuery.trim()) {
      setResults([]);
      return;
    }

    // Cancel previous request if it exists
    if (abortControllerRef.current) {
      abortControllerRef.current.abort();
    }

    abortControllerRef.current = new AbortController();

    const search = async () => {
      setLoading(true);

      try {
        const response = await fetch(
          `/api/search?q=${debouncedQuery}`,
          {
            signal: abortControllerRef.current.signal,
          }
        );

        const data = await response.json();
        setResults(data);
        setLoading(false);
      } catch (err) {
        if (err.name !== 'AbortError') {
          console.error('Search error:', err);
          setLoading(false);
        }
      }
    };

    search();

    return () => {
      if (abortControllerRef.current) {
        abortControllerRef.current.abort();
      }
    };
  }, [debouncedQuery]);

  return (
    <div>
      <input
        type="search"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search..."
      />
      {loading && <div>Searching...</div>}
      <ul>
        {results.map(result => (
          <li key={result.id}>{result.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

### Infinite Scroll with Race Protection

```javascript
function InfiniteList() {
  const [items, setItems] = useState([]);
  const [page, setPage] = useState(1);
  const [loading, setLoading] = useState(false);
  const [hasMore, setHasMore] = useState(true);
  const observerTarget = useRef(null);

  useEffect(() => {
    const controller = new AbortController();
    let isActive = true;

    const loadMore = async () => {
      if (loading || !hasMore) return;

      setLoading(true);

      try {
        const response = await fetch(`/api/items?page=${page}`, {
          signal: controller.signal,
        });

        const data = await response.json();

        if (isActive) {
          setItems(prev => [...prev, ...data.items]);
          setHasMore(data.hasMore);
          setLoading(false);
        }
      } catch (err) {
        if (isActive && err.name !== 'AbortError') {
          console.error('Load error:', err);
          setLoading(false);
        }
      }
    };

    loadMore();

    return () => {
      isActive = false;
      controller.abort();
    };
  }, [page]);

  // Intersection Observer for infinite scroll
  useEffect(() => {
    const observer = new IntersectionObserver(
      entries => {
        if (entries[0].isIntersecting && hasMore && !loading) {
          setPage(prev => prev + 1);
        }
      },
      { threshold: 1 }
    );

    if (observerTarget.current) {
      observer.observe(observerTarget.current);
    }

    return () => observer.disconnect();
  }, [hasMore, loading]);

  return (
    <div>
      <ul>
        {items.map(item => (
          <li key={item.id}>{item.title}</li>
        ))}
      </ul>
      <div ref={observerTarget}>
        {loading && <div>Loading more...</div>}
        {!hasMore && <div>No more items</div>}
      </div>
    </div>
  );
}
```

### Concurrent Requests with Latest-Wins

```javascript
function DashboardData({ filters }) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);
  const requestIdRef = useRef(0);

  useEffect(() => {
    const requestId = ++requestIdRef.current;

    const fetchDashboardData = async () => {
      setLoading(true);

      try {
        // Fetch multiple endpoints concurrently
        const [users, orders, analytics] = await Promise.all([
          fetch(`/api/users?filter=${filters.users}`).then(r => r.json()),
          fetch(`/api/orders?filter=${filters.orders}`).then(r => r.json()),
          fetch(`/api/analytics?filter=${filters.analytics}`).then(r => r.json()),
        ]);

        // Only update if this is still the latest request
        if (requestId === requestIdRef.current) {
          setData({ users, orders, analytics });
          setLoading(false);
        }
      } catch (err) {
        if (requestId === requestIdRef.current) {
          console.error('Dashboard error:', err);
          setLoading(false);
        }
      }
    };

    fetchDashboardData();
  }, [filters]);

  if (loading) return <div>Loading dashboard...</div>;
  if (!data) return null;

  return (
    <div>
      <div>Users: {data.users.length}</div>
      <div>Orders: {data.orders.length}</div>
      <div>Analytics: {JSON.stringify(data.analytics)}</div>
    </div>
  );
}
```

## Best Practices

### 1. Always Clean Up Effects

```javascript
// Good: Always return cleanup function
useEffect(() => {
  const controller = new AbortController();

  fetch(url, { signal: controller.signal })
    .then(/* ... */);

  return () => controller.abort(); // Cleanup
}, [url]);
```

### 2. Use AbortController for Fetch

```javascript
// Good: Modern cancellation with AbortController
const controller = new AbortController();
fetch(url, { signal: controller.signal });

// Avoid: Boolean flags (but acceptable if AbortController unavailable)
let isActive = true;
// ...
if (isActive) { /* ... */ }
```

### 3. Combine with Debouncing

```javascript
// Good: Debounce + cancellation for search
function SearchComponent() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);

  useEffect(() => {
    const controller = new AbortController();
    // Fetch with debouncedQuery
    return () => controller.abort();
  }, [debouncedQuery]);
}
```

### 4. Handle AbortError Gracefully

```javascript
// Good: Check for AbortError
.catch(err => {
  if (err.name === 'AbortError') {
    // Expected, don't show error
    return;
  }
  setError(err.message);
});

// Bad: Show error for all failures
.catch(err => {
  setError(err.message); // Shows error even when cancelled
});
```

## Common Mistakes

### 1. Not Cancelling Requests

```javascript
// Bad: No cleanup
useEffect(() => {
  fetch(url).then(res => res.json()).then(setData);
}, [url]);

// Good: Cancel on cleanup
useEffect(() => {
  const controller = new AbortController();
  fetch(url, { signal: controller.signal })
    .then(res => res.json())
    .then(setData);
  return () => controller.abort();
}, [url]);
```

### 2. Not Checking isActive Flag

```javascript
// Bad: Updates state even after unmount
useEffect(() => {
  fetch(url).then(data => setState(data)); // May run after unmount!
}, [url]);

// Good: Check if still mounted
useEffect(() => {
  let isActive = true;
  fetch(url).then(data => {
    if (isActive) setState(data);
  });
  return () => { isActive = false; };
}, [url]);
```

### 3. Forgetting Debounce for Search

```javascript
// Bad: Request on every keystroke
function Search() {
  const [query, setQuery] = useState('');

  useEffect(() => {
    fetch(`/api/search?q=${query}`); // Too many requests!
  }, [query]);
}

// Good: Debounced search
function Search() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);

  useEffect(() => {
    if (!debouncedQuery) return;
    fetch(`/api/search?q=${debouncedQuery}`);
  }, [debouncedQuery]);
}
```

## Testing

```javascript
import { render, screen, waitFor } from '@testing-library/react';
import { rest } from 'msw';
import { setupServer } from 'msw/node';

const server = setupServer(
  rest.get('/api/users/:id', (req, res, ctx) => {
    return res(ctx.json({ id: req.params.id, name: 'User' }));
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test('prevents race conditions', async () => {
  const { rerender } = render(<UserProfile userId={1} />);

  // Change userId quickly
  rerender(<UserProfile userId={2} />);
  rerender(<UserProfile userId={3} />);

  // Should show user 3, not user 1 or 2
  await waitFor(() => {
    expect(screen.getByText(/User/)).toBeInTheDocument();
  });
});

test('cancels previous request', async () => {
  let abortedCount = 0;

  server.use(
    rest.get('/api/users/:id', (req, res, ctx) => {
      req.signal.addEventListener('abort', () => {
        abortedCount++;
      });
      return res(ctx.delay(100), ctx.json({ id: req.params.id }));
    })
  );

  const { rerender } = render(<UserProfile userId={1} />);
  rerender(<UserProfile userId={2} />);

  await waitFor(() => {
    expect(abortedCount).toBe(1); // First request cancelled
  });
});
```

## Key Takeaways

1. **Always Clean Up**: Return cleanup functions from useEffect
2. **Use AbortController**: Modern way to cancel fetch requests
3. **Boolean Flags**: Acceptable fallback when AbortController isn't available
4. **Debounce Searches**: Reduce requests for typeahead/search
5. **Request Versioning**: Track latest request with IDs or refs
6. **Handle AbortError**: Don't show errors for cancelled requests
7. **Test Race Conditions**: Write tests that verify proper cancellation

## Additional Resources

- [AbortController MDN](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)
- [React Beta Docs - You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect)
- [Handling Race Conditions in React](https://maxrozen.com/race-conditions-fetching-data-react-with-useeffect)
- [TanStack Query](https://tanstack.com/query) - Handles race conditions automatically
