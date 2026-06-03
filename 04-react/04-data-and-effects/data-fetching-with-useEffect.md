# Data Fetching with useEffect (and Its Problems)

## Overview

Using `useEffect` for data fetching is a common pattern in React, but it comes with several challenges and pitfalls. While it works for simple cases, modern alternatives (React Query, SWR, Server Components) often provide better solutions.

## Basic Pattern

### Simple Fetch

```jsx
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    setLoading(true);
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => {
        setUser(data);
        setLoading(false);
      })
      .catch(error => {
        setError(error);
        setLoading(false);
      });
  }, [userId]);
  
  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  if (!user) return null;
  
  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}
```

### With Async/Await

```jsx
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    // Can't make useEffect callback async directly!
    const fetchUser = async () => {
      try {
        setLoading(true);
        const response = await fetch(`/api/users/${userId}`);
        const data = await response.json();
        setUser(data);
      } catch (err) {
        setError(err);
      } finally {
        setLoading(false);
      }
    };
    
    fetchUser();
  }, [userId]);
  
  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  return <div>{user.name}</div>;
}
```

## The Problems

### Problem 1: Race Conditions

```jsx
// ❌ Bad: Race condition
function SearchResults({ query }) {
  const [results, setResults] = useState([]);
  
  useEffect(() => {
    fetch(`/api/search?q=${query}`)
      .then(res => res.json())
      .then(setResults);
  }, [query]);
  
  // Problem:
  // User types "react" → request starts
  // User types "react native" → another request starts
  // "react native" returns first → setResults called
  // "react" returns second → setResults called with old data!
  // UI shows "react" results for "react native" query
  
  return <ResultsList results={results} />;
}

// ✅ Good: Handle race condition with cleanup
function SearchResults({ query }) {
  const [results, setResults] = useState([]);
  
  useEffect(() => {
    let cancelled = false;
    
    fetch(`/api/search?q=${query}`)
      .then(res => res.json())
      .then(data => {
        if (!cancelled) {
          setResults(data);
        }
      });
    
    return () => {
      cancelled = true;
    };
  }, [query]);
  
  return <ResultsList results={results} />;
}
```

### Problem 2: Memory Leaks

```jsx
// ❌ Bad: Memory leak if component unmounts
function BadComponent({ userId }) {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(setUser);  // ← Called even if component unmounted!
  }, [userId]);
  
  return <div>{user?.name}</div>;
}

// ✅ Good: Cleanup prevents memory leak
function GoodComponent({ userId }) {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    let cancelled = false;
    
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => {
        if (!cancelled) {
          setUser(data);
        }
      });
    
    return () => {
      cancelled = true;
    };
  }, [userId]);
  
  return <div>{user?.name}</div>;
}
```

### Problem 3: No Request Cancellation

```jsx
// ❌ Bad: Can't cancel ongoing requests
function Component({ userId }) {
  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(setUser);
    
    // Cleanup doesn't cancel the network request!
    return () => {
      // Request still in flight...
    };
  }, [userId]);
}

// ✅ Good: Use AbortController
function Component({ userId }) {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    const controller = new AbortController();
    
    fetch(`/api/users/${userId}`, {
      signal: controller.signal
    })
      .then(res => res.json())
      .then(setUser)
      .catch(err => {
        if (err.name === 'AbortError') {
          console.log('Request cancelled');
        }
      });
    
    return () => {
      controller.abort();
    };
  }, [userId]);
  
  return <div>{user?.name}</div>;
}
```

### Problem 4: Loading States

```jsx
// ❌ Bad: Shows stale data during loading
function BadComponent({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(false);
  
  useEffect(() => {
    setLoading(true);
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => {
        setUser(data);
        setLoading(false);
      });
  }, [userId]);
  
  // Problem: Shows old user while loading new one
  if (loading) return <Spinner />;
  return <div>{user?.name}</div>;
}

// ✅ Better: Show loading state for initial load only
function BetterComponent({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [refreshing, setRefreshing] = useState(false);
  
  useEffect(() => {
    const isInitialLoad = !user;
    
    if (isInitialLoad) {
      setLoading(true);
    } else {
      setRefreshing(true);
    }
    
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => {
        setUser(data);
        setLoading(false);
        setRefreshing(false);
      });
  }, [userId]);
  
  if (loading) return <Spinner />;
  
  return (
    <div style={{ opacity: refreshing ? 0.6 : 1 }}>
      {user?.name}
    </div>
  );
}
```

### Problem 5: No Caching

```jsx
// ❌ Bad: Fetches every time component mounts
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(setUser);
  }, [userId]);
  
  // If you navigate away and back, fetches again!
  // Same userId → same request → wasted bandwidth
  
  return <div>{user?.name}</div>;
}

// ✅ Better: Implement simple cache
const userCache = new Map();

function UserProfile({ userId }) {
  const [user, setUser] = useState(() => userCache.get(userId) || null);
  
  useEffect(() => {
    if (userCache.has(userId)) {
      setUser(userCache.get(userId));
      return;
    }
    
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => {
        userCache.set(userId, data);
        setUser(data);
      });
  }, [userId]);
  
  return <div>{user?.name}</div>;
}
```

### Problem 6: Error Recovery

```jsx
// ❌ Bad: No retry logic
function BadComponent({ userId }) {
  const [user, setUser] = useState(null);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(setUser)
      .catch(setError);
  }, [userId]);
  
  if (error) return <div>Error! 😞</div>;  // User stuck
  return <div>{user?.name}</div>;
}

// ✅ Better: Add retry
function BetterComponent({ userId }) {
  const [user, setUser] = useState(null);
  const [error, setError] = useState(null);
  const [retryCount, setRetryCount] = useState(0);
  
  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(setUser)
      .catch(err => {
        setError(err);
      });
  }, [userId, retryCount]);
  
  if (error) {
    return (
      <div>
        Error: {error.message}
        <button onClick={() => setRetryCount(c => c + 1)}>
          Retry
        </button>
      </div>
    );
  }
  
  return <div>{user?.name}</div>;
}
```

## Better Patterns

### Custom Hook for Data Fetching

```jsx
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    const controller = new AbortController();
    
    setLoading(true);
    
    fetch(url, { signal: controller.signal })
      .then(res => {
        if (!res.ok) throw new Error('Network response was not ok');
        return res.json();
      })
      .then(data => {
        setData(data);
        setLoading(false);
      })
      .catch(err => {
        if (err.name !== 'AbortError') {
          setError(err);
          setLoading(false);
        }
      });
    
    return () => {
      controller.abort();
    };
  }, [url]);
  
  return { data, loading, error };
}

// Usage
function UserProfile({ userId }) {
  const { data: user, loading, error } = useFetch(`/api/users/${userId}`);
  
  if (loading) return <Spinner />;
  if (error) return <Error message={error.message} />;
  return <div>{user.name}</div>;
}
```

### With Retry Logic

```jsx
function useFetchWithRetry(url, retries = 3) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    let cancelled = false;
    let currentRetries = 0;
    
    const fetchData = async () => {
      try {
        setLoading(true);
        const response = await fetch(url);
        
        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }
        
        const data = await response.json();
        
        if (!cancelled) {
          setData(data);
          setError(null);
        }
      } catch (err) {
        if (!cancelled) {
          if (currentRetries < retries) {
            currentRetries++;
            console.log(`Retry ${currentRetries}/${retries}`);
            setTimeout(fetchData, 1000 * currentRetries);  // Exponential backoff
          } else {
            setError(err);
          }
        }
      } finally {
        if (!cancelled) {
          setLoading(false);
        }
      }
    };
    
    fetchData();
    
    return () => {
      cancelled = true;
    };
  }, [url, retries]);
  
  return { data, loading, error };
}
```

### With Polling

```jsx
function usePoll(url, interval = 5000) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    let cancelled = false;
    
    const fetchData = async () => {
      try {
        const response = await fetch(url);
        const data = await response.json();
        
        if (!cancelled) {
          setData(data);
          setLoading(false);
        }
      } catch (err) {
        if (!cancelled) {
          setError(err);
          setLoading(false);
        }
      }
    };
    
    fetchData();  // Initial fetch
    
    const pollInterval = setInterval(fetchData, interval);
    
    return () => {
      cancelled = true;
      clearInterval(pollInterval);
    };
  }, [url, interval]);
  
  return { data, loading, error };
}

// Usage: Real-time dashboard
function Dashboard() {
  const { data: stats } = usePoll('/api/stats', 10000);  // Poll every 10s
  
  return <StatsDisplay stats={stats} />;
}
```

### With Pagination

```jsx
function usePaginatedFetch(url, pageSize = 10) {
  const [data, setData] = useState([]);
  const [page, setPage] = useState(1);
  const [hasMore, setHasMore] = useState(true);
  const [loading, setLoading] = useState(false);
  
  useEffect(() => {
    const fetchPage = async () => {
      setLoading(true);
      
      const response = await fetch(
        `${url}?page=${page}&pageSize=${pageSize}`
      );
      const newData = await response.json();
      
      setData(prev => [...prev, ...newData]);
      setHasMore(newData.length === pageSize);
      setLoading(false);
    };
    
    fetchPage();
  }, [url, page, pageSize]);
  
  const loadMore = () => {
    if (!loading && hasMore) {
      setPage(p => p + 1);
    }
  };
  
  return { data, loading, hasMore, loadMore };
}

// Usage: Infinite scroll
function ProductList() {
  const { data, loading, hasMore, loadMore } = usePaginatedFetch('/api/products');
  
  return (
    <>
      {data.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
      
      {hasMore && (
        <button onClick={loadMore} disabled={loading}>
          {loading ? 'Loading...' : 'Load More'}
        </button>
      )}
    </>
  );
}
```

## Modern Alternatives

### React Query (Recommended)

```jsx
import { useQuery } from '@tanstack/react-query';

function UserProfile({ userId }) {
  const { data: user, isLoading, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetch(`/api/users/${userId}`).then(res => res.json())
  });
  
  if (isLoading) return <Spinner />;
  if (error) return <Error message={error.message} />;
  return <div>{user.name}</div>;
}

// Benefits:
// ✅ Automatic caching
// ✅ Request deduplication
// ✅ Background refetching
// ✅ Retry logic
// ✅ Pagination support
// ✅ Optimistic updates
// ✅ And much more!
```

### SWR

```jsx
import useSWR from 'swr';

const fetcher = url => fetch(url).then(res => res.json());

function UserProfile({ userId }) {
  const { data: user, error, isLoading } = useSWR(
    `/api/users/${userId}`,
    fetcher
  );
  
  if (isLoading) return <Spinner />;
  if (error) return <Error />;
  return <div>{user.name}</div>;
}

// Benefits:
// ✅ Stale-while-revalidate strategy
// ✅ Focus revalidation
// ✅ Interval polling
// ✅ Request deduplication
// ✅ Lightweight
```

### Server Components (React 19)

```jsx
// Server Component - async by default!
async function UserProfile({ userId }) {
  const user = await fetch(`/api/users/${userId}`).then(res => res.json());
  
  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}

// Benefits:
// ✅ No useEffect needed
// ✅ Fetches on server
// ✅ No client JS bundle
// ✅ Better SEO
// ✅ Faster initial load
```

## When to Use useEffect for Fetching

### Good Use Cases:
- Quick prototypes
- Simple one-off fetches
- Learning purposes
- No library dependencies allowed
- Very simple requirements

### Bad Use Cases:
- Production applications
- Complex data requirements
- Need caching/revalidation
- Multiple related requests
- Real-time data

## Best Practices

1. **Always cleanup**: Use AbortController
2. **Handle race conditions**: Check if cancelled
3. **Loading states**: Show appropriate feedback
4. **Error handling**: Don't leave users stuck
5. **Avoid waterfall requests**: Fetch in parallel
6. **Consider alternatives**: React Query, SWR, Server Components

## Resources

- [React Query](https://tanstack.com/query)
- [SWR](https://swr.vercel.app/)
- [Server Components](https://react.dev/blog/2023/03/22/react-labs-what-we-have-been-working-on-march-2023#react-server-components)
- [AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)
