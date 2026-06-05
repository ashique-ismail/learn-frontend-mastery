# SWR - React Hooks for Data Fetching

## The Idea

**In plain English:** SWR is a tool that helps your webpage automatically fetch fresh data from a server — it shows you saved (cached) data instantly while quietly checking in the background for any updates. Think of it like a smart shortcut that keeps your screen from being blank while new info loads.

**Real-world analogy:** Imagine you walk into a coffee shop and ask the barista for today's specials. They immediately hand you yesterday's printed menu (so you're not standing there doing nothing), then radio the kitchen to get the actual current list and swap your menu out once it arrives.

- The printed menu from yesterday = the cached (stale) data SWR shows immediately
- Radioing the kitchen = the background network request SWR sends to the server
- The updated menu you get a moment later = the fresh data that replaces the stale version

---

## Overview

SWR is a React Hooks library for data fetching developed by Vercel. The name "SWR" is derived from stale-while-revalidate, an HTTP cache invalidation strategy. SWR first returns cached (stale) data, then sends the fetch request, and finally comes with the up-to-date data.

### Key Features

- **Fast page navigation**: Instant loading with cache
- **Revalidation**: Auto revalidate on focus/reconnect
- **Mutation**: Optimistic UI updates
- **Pagination**: Built-in support
- **TypeScript**: Full type safety
- **Lightweight**: <5KB bundle size
- **SSR/SSG**: Server-side rendering support

## Installation

```bash
npm install swr
```

## Basic Usage

```typescript
import useSWR from 'swr';

const fetcher = (url: string) => fetch(url).then(res => res.json());

function Profile() {
  const { data, error, isLoading } = useSWR('/api/user', fetcher);

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Failed to load</div>;
  
  return <div>Hello {data.name}!</div>;
}
```

## Global Configuration

```typescript
import { SWRConfig } from 'swr';

function App() {
  return (
    <SWRConfig 
      value={{
        fetcher: (url: string) => fetch(url).then(res => res.json()),
        refreshInterval: 3000,
        revalidateOnFocus: true,
        revalidateOnReconnect: true,
      }}
    >
      <YourApp />
    </SWRConfig>
  );
}
```

## Error Handling

```typescript
interface User {
  name: string;
  email: string;
}

function UserProfile() {
  const { data, error } = useSWR<User>('/api/user', fetcher, {
    onError: (error) => {
      console.error('Error loading user:', error);
    },
    shouldRetryOnError: true,
    errorRetryCount: 3,
    errorRetryInterval: 5000,
  });

  if (error) {
    return (
      <div>
        <p>Error: {error.message}</p>
        <button onClick={() => window.location.reload()}>Retry</button>
      </div>
    );
  }

  if (!data) return <div>Loading...</div>;

  return (
    <div>
      <h2>{data.name}</h2>
      <p>{data.email}</p>
    </div>
  );
}
```

## Mutation

### Bound Mutate

```typescript
import useSWR, { mutate } from 'swr';

function TodoList() {
  const { data: todos } = useSWR('/api/todos', fetcher);

  const addTodo = async (text: string) => {
    // Optimistic update
    await mutate(
      '/api/todos',
      async (currentTodos) => {
        const newTodo = { id: Date.now(), text, completed: false };
        
        // Update local data immediately
        const updatedTodos = [...(currentTodos || []), newTodo];
        
        // Send request to API
        await fetch('/api/todos', {
          method: 'POST',
          body: JSON.stringify(newTodo),
        });
        
        return updatedTodos;
      },
      {
        optimisticData: (currentTodos) => [
          ...(currentTodos || []),
          { id: Date.now(), text, completed: false }
        ],
        rollbackOnError: true,
        populateCache: true,
        revalidate: false,
      }
    );
  };

  return (
    <div>
      <ul>
        {todos?.map((todo) => (
          <li key={todo.id}>{todo.text}</li>
        ))}
      </ul>
      <button onClick={() => addTodo('New Todo')}>Add Todo</button>
    </div>
  );
}
```

### useSWRMutation

```typescript
import useSWRMutation from 'swr/mutation';

async function sendRequest(url: string, { arg }: { arg: { text: string } }) {
  return fetch(url, {
    method: 'POST',
    body: JSON.stringify(arg),
  }).then(res => res.json());
}

function CreateTodo() {
  const { trigger, isMutating, error } = useSWRMutation('/api/todos', sendRequest);

  return (
    <button
      disabled={isMutating}
      onClick={async () => {
        try {
          const result = await trigger({ text: 'New Todo' });
          console.log('Created:', result);
        } catch (e) {
          console.error('Error:', e);
        }
      }}
    >
      Create Todo
    </button>
  );
}
```

## Pagination

```typescript
function PagedPosts() {
  const [page, setPage] = React.useState(1);
  
  const { data, error } = useSWR(`/api/posts?page=${page}`, fetcher, {
    keepPreviousData: true,
  });

  return (
    <div>
      {data?.posts.map((post) => (
        <div key={post.id}>{post.title}</div>
      ))}
      
      <button
        onClick={() => setPage(page - 1)}
        disabled={page === 1}
      >
        Previous
      </button>
      <span>Page {page}</span>
      <button
        onClick={() => setPage(page + 1)}
        disabled={!data?.hasMore}
      >
        Next
      </button>
    </div>
  );
}
```

## Infinite Loading

```typescript
import useSWRInfinite from 'swr/infinite';

interface Post {
  id: string;
  title: string;
}

function InfinitePosts() {
  const getKey = (pageIndex: number, previousPageData: any) => {
    if (previousPageData && !previousPageData.posts.length) return null;
    return `/api/posts?page=${pageIndex + 1}`;
  };

  const { data, size, setSize, isLoading } = useSWRInfinite<{ posts: Post[]; hasMore: boolean }>(
    getKey,
    fetcher
  );

  const posts = data ? data.flatMap(page => page.posts) : [];
  const isLoadingMore = isLoading || (size > 0 && data && typeof data[size - 1] === 'undefined');
  const isEmpty = data?.[0]?.posts.length === 0;
  const isReachingEnd = isEmpty || (data && !data[data.length - 1]?.hasMore);

  return (
    <div>
      {posts.map((post) => (
        <div key={post.id}>{post.title}</div>
      ))}
      
      {!isReachingEnd && (
        <button
          disabled={isLoadingMore}
          onClick={() => setSize(size + 1)}
        >
          {isLoadingMore ? 'Loading...' : 'Load More'}
        </button>
      )}
    </div>
  );
}
```

## Conditional Fetching

```typescript
function UserPosts({ userId }: { userId: string | null }) {
  const { data } = useSWR(
    userId ? `/api/users/${userId}/posts` : null,
    fetcher
  );

  if (!userId) return <div>Select a user</div>;
  if (!data) return <div>Loading...</div>;

  return (
    <ul>
      {data.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

## Dependent Queries

```typescript
function UserPosts({ userId }: { userId: string }) {
  const { data: user } = useSWR(`/api/users/${userId}`, fetcher);
  const { data: posts } = useSWR(
    user ? `/api/users/${user.id}/posts` : null,
    fetcher
  );

  if (!user) return <div>Loading user...</div>;
  if (!posts) return <div>Loading posts...</div>;

  return (
    <div>
      <h2>{user.name}'s Posts</h2>
      <ul>
        {posts.map((post) => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

## Prefetching

```typescript
import { mutate } from 'swr';

function PostList() {
  const { data: posts } = useSWR('/api/posts', fetcher);

  const prefetch = (postId: string) => {
    mutate(
      `/api/posts/${postId}`,
      fetch(`/api/posts/${postId}`).then(res => res.json()),
      { revalidate: false }
    );
  };

  return (
    <ul>
      {posts?.map((post) => (
        <li key={post.id} onMouseEnter={() => prefetch(post.id)}>
          <Link to={`/posts/${post.id}`}>{post.title}</Link>
        </li>
      ))}
    </ul>
  );
}
```

## Polling

```typescript
function LiveData() {
  const { data } = useSWR('/api/data', fetcher, {
    refreshInterval: 1000, // Poll every second
    refreshWhenHidden: false,
    refreshWhenOffline: false,
  });

  return <div>Data: {data?.value}</div>;
}
```

## Suspense Mode

```typescript
import { Suspense } from 'react';

function Profile() {
  const { data } = useSWR('/api/user', fetcher, {
    suspense: true,
  });

  return <div>Hello {data.name}!</div>;
}

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Profile />
    </Suspense>
  );
}
```

## Focus Revalidation

```typescript
function RealtimeData() {
  const { data } = useSWR('/api/data', fetcher, {
    revalidateOnFocus: true,
    revalidateOnReconnect: true,
    focusThrottleInterval: 5000,
  });

  return <div>Latest: {data?.value}</div>;
}
```

## Middleware

```typescript
import { SWRConfig, Middleware } from 'swr';

const logger: Middleware = (useSWRNext) => (key, fetcher, config) => {
  const swr = useSWRNext(key, fetcher, config);

  React.useEffect(() => {
    console.log('SWR Request:', key);
  }, [key]);

  return swr;
};

function App() {
  return (
    <SWRConfig value={{ use: [logger] }}>
      <YourApp />
    </SWRConfig>
  );
}
```

## Testing

```typescript
import { renderHook, waitFor } from '@testing-library/react';
import { SWRConfig } from 'swr';

test('useSWR hook', async () => {
  const wrapper = ({ children }) => (
    <SWRConfig value={{ provider: () => new Map() }}>
      {children}
    </SWRConfig>
  );

  const { result } = renderHook(
    () => useSWR('/api/user', () => Promise.resolve({ name: 'John' })),
    { wrapper }
  );

  await waitFor(() => expect(result.current.data).toEqual({ name: 'John' }));
});
```

## Common Mistakes

1. **Not handling errors**: Always provide error UI
2. **Unnecessary revalidation**: Configure appropriately
3. **Missing key dependencies**: Keys should include all dependencies
4. **Not using optimistic updates**: Missing better UX
5. **Incorrect mutation**: Not properly updating cache

## Best Practices

1. **Use TypeScript**: Full type safety
2. **Global configuration**: Set defaults in SWRConfig
3. **Optimistic updates**: Improve perceived performance
4. **Prefetch on hover**: Better navigation experience
5. **Handle errors gracefully**: Provide retry mechanisms
6. **Use suspense mode**: Simpler loading states
7. **Configure revalidation**: Based on data freshness needs

## When to Use SWR

### Good Use Cases
- Real-time data applications
- Simple data fetching needs
- When bundle size matters
- Projects using Next.js/Vercel
- Auto-revalidation requirements

### Not Ideal For
- Complex query dependencies
- Need for advanced caching strategies
- GraphQL APIs (use Apollo Client instead)

## Interview Questions

1. **What does SWR stand for?**
   - Stale-While-Revalidate

2. **How does SWR handle caching?**
   - Returns cached data immediately, revalidates in background

3. **What is optimistic update?**
   - Updating UI before server confirms the change

4. **How does focus revalidation work?**
   - Automatically refetches when user focuses window

5. **What is the difference between useSWR and useSWRMutation?**
   - useSWR for GET requests; useSWRMutation for POST/PUT/DELETE

## Key Takeaways

- SWR provides simple data fetching hooks
- Stale-while-revalidate strategy
- Automatic revalidation on focus/reconnect
- Built-in optimistic updates
- Lightweight bundle size
- Excellent TypeScript support
- Perfect for Next.js applications
- Simple API compared to alternatives

## Resources

- [Official Documentation](https://swr.vercel.app/)
- [Examples](https://swr.vercel.app/examples/basic)
- [GitHub](https://github.com/vercel/swr)
- [API Reference](https://swr.vercel.app/docs/api)
