# TanStack Query - Powerful Async State Management

## The Idea

**In plain English:** TanStack Query is a tool that manages the process of asking a server for data — it remembers the answers so your app does not have to keep asking the same questions, and it automatically checks for updates in the background. "Server state" just means information that lives on a remote computer (like a database), not inside your app itself.

**Real-world analogy:** Imagine a school librarian who keeps a notebook of recent book searches. When a student asks "do we have Harry Potter?", the librarian checks her notebook first — if she looked it up recently, she gives the answer instantly without walking to the shelves. If the answer is old, she quietly goes to check the shelves while still giving you the last known answer right away, and updates her notebook when she returns.

- The librarian's notebook = the query cache (stored answers)
- Asking "do we have Harry Potter?" = a `useQuery` call (fetching data)
- The librarian quietly re-checking the shelves = background refetching
- The notebook entry going "stale" after a while = `staleTime` (how long data is considered fresh)

---

## Overview

TanStack Query (formerly React Query) is a powerful data-fetching and state management library for web applications. It handles caching, background updates, and stale data with zero configuration while remaining highly customizable.

### Key Features

- **Automatic caching**: Smart caching with background refetching
- **Optimistic updates**: Update UI before server response
- **Pagination and infinite scroll**: Built-in support
- **Request deduplication**: Automatic deduplication
- **DevTools**: Debugging interface
- **SSR support**: Server-side rendering compatible
- **TypeScript**: Full type safety

## Installation

```bash
npm install @tanstack/react-query
```

## Basic Setup

```typescript
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient();

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <YourApp />
    </QueryClientProvider>
  );
}
```

## useQuery - Fetching Data

### Basic Query

```typescript
import { useQuery } from '@tanstack/react-query';

interface User {
  id: string;
  name: string;
  email: string;
}

function UserProfile({ userId }: { userId: string }) {
  const { data, isLoading, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: async () => {
      const response = await fetch(`/api/users/${userId}`);
      if (!response.ok) throw new Error('Failed to fetch user');
      return response.json() as Promise<User>;
    },
  });

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <div>
      <h2>{data.name}</h2>
      <p>{data.email}</p>
    </div>
  );
}
```

### Query with Options

```typescript
const { data, isLoading, isFetching, refetch } = useQuery({
  queryKey: ['products'],
  queryFn: fetchProducts,
  staleTime: 5 * 60 * 1000, // 5 minutes
  cacheTime: 10 * 60 * 1000, // 10 minutes
  retry: 3,
  retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
  refetchOnWindowFocus: true,
  refetchOnMount: true,
  refetchOnReconnect: true,
});
```

## useMutation - Modifying Data

### Basic Mutation

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';

function CreatePost() {
  const queryClient = useQueryClient();
  
  const mutation = useMutation({
    mutationFn: async (newPost: { title: string; body: string }) => {
      const response = await fetch('/api/posts', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(newPost),
      });
      return response.json();
    },
    onSuccess: () => {
      // Invalidate and refetch
      queryClient.invalidateQueries({ queryKey: ['posts'] });
    },
  });

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    mutation.mutate({
      title: 'New Post',
      body: 'Content here',
    });
  };

  return (
    <form onSubmit={handleSubmit}>
      <button type="submit" disabled={mutation.isPending}>
        {mutation.isPending ? 'Creating...' : 'Create Post'}
      </button>
      {mutation.isError && <p>Error: {mutation.error.message}</p>}
      {mutation.isSuccess && <p>Post created!</p>}
    </form>
  );
}
```

### Optimistic Updates

```typescript
interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

function TodoList() {
  const queryClient = useQueryClient();

  const { data: todos } = useQuery<Todo[]>({
    queryKey: ['todos'],
    queryFn: fetchTodos,
  });

  const mutation = useMutation({
    mutationFn: async (id: string) => {
      await fetch(`/api/todos/${id}/toggle`, { method: 'PATCH' });
    },
    onMutate: async (id) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: ['todos'] });

      // Snapshot previous value
      const previousTodos = queryClient.getQueryData<Todo[]>(['todos']);

      // Optimistically update
      queryClient.setQueryData<Todo[]>(['todos'], (old) =>
        old?.map((todo) =>
          todo.id === id ? { ...todo, completed: !todo.completed } : todo
        )
      );

      return { previousTodos };
    },
    onError: (err, id, context) => {
      // Rollback on error
      queryClient.setQueryData(['todos'], context?.previousTodos);
    },
    onSettled: () => {
      // Refetch after error or success
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });

  return (
    <ul>
      {todos?.map((todo) => (
        <li key={todo.id}>
          <input
            type="checkbox"
            checked={todo.completed}
            onChange={() => mutation.mutate(todo.id)}
          />
          {todo.text}
        </li>
      ))}
    </ul>
  );
}
```

## Dependent Queries

```typescript
function UserPosts({ userId }: { userId: string }) {
  const { data: user } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });

  const { data: posts } = useQuery({
    queryKey: ['posts', userId],
    queryFn: () => fetchUserPosts(userId),
    enabled: !!user, // Only run when user is available
  });

  return (
    <div>
      <h2>{user?.name}'s Posts</h2>
      <ul>
        {posts?.map((post) => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

## Pagination

```typescript
function Posts() {
  const [page, setPage] = React.useState(1);

  const { data, isLoading, isPlaceholderData } = useQuery({
    queryKey: ['posts', page],
    queryFn: () => fetchPosts(page),
    placeholderData: (previousData) => previousData,
  });

  return (
    <div>
      {isLoading ? (
        <div>Loading...</div>
      ) : (
        <ul>
          {data?.posts.map((post) => (
            <li key={post.id}>{post.title}</li>
          ))}
        </ul>
      )}

      <button
        onClick={() => setPage((old) => Math.max(old - 1, 1))}
        disabled={page === 1}
      >
        Previous
      </button>
      <span>Page {page}</span>
      <button
        onClick={() => {
          if (!isPlaceholderData && data?.hasMore) {
            setPage((old) => old + 1);
          }
        }}
        disabled={isPlaceholderData || !data?.hasMore}
      >
        Next
      </button>
    </div>
  );
}
```

## Infinite Queries

```typescript
import { useInfiniteQuery } from '@tanstack/react-query';

function InfinitePosts() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfiniteQuery({
    queryKey: ['posts'],
    queryFn: ({ pageParam = 0 }) => fetchPosts(pageParam),
    getNextPageParam: (lastPage, pages) => lastPage.nextCursor,
    initialPageParam: 0,
  });

  return (
    <div>
      {data?.pages.map((page, i) => (
        <React.Fragment key={i}>
          {page.posts.map((post) => (
            <div key={post.id}>{post.title}</div>
          ))}
        </React.Fragment>
      ))}

      <button
        onClick={() => fetchNextPage()}
        disabled={!hasNextPage || isFetchingNextPage}
      >
        {isFetchingNextPage
          ? 'Loading more...'
          : hasNextPage
          ? 'Load More'
          : 'Nothing more to load'}
      </button>
    </div>
  );
}
```

## Prefetching

```typescript
function PostList() {
  const queryClient = useQueryClient();

  const { data: posts } = useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
  });

  const prefetchPost = (postId: string) => {
    queryClient.prefetchQuery({
      queryKey: ['post', postId],
      queryFn: () => fetchPost(postId),
    });
  };

  return (
    <ul>
      {posts?.map((post) => (
        <li
          key={post.id}
          onMouseEnter={() => prefetchPost(post.id)}
        >
          <Link to={`/posts/${post.id}`}>{post.title}</Link>
        </li>
      ))}
    </ul>
  );
}
```

## Query Invalidation

```typescript
const queryClient = useQueryClient();

// Invalidate specific query
queryClient.invalidateQueries({ queryKey: ['posts'] });

// Invalidate all queries starting with 'posts'
queryClient.invalidateQueries({ queryKey: ['posts'], exact: false });

// Invalidate and refetch immediately
queryClient.invalidateQueries({ 
  queryKey: ['posts'],
  refetchType: 'active'
});

// Remove query from cache
queryClient.removeQueries({ queryKey: ['posts', postId] });

// Reset query to initial state
queryClient.resetQueries({ queryKey: ['posts'] });
```

## Query Cancellation

```typescript
const { data, refetch } = useQuery({
  queryKey: ['posts'],
  queryFn: async ({ signal }) => {
    const response = await fetch('/api/posts', { signal });
    return response.json();
  },
});

// Query will be cancelled if component unmounts
```

## Parallel Queries

```typescript
function Dashboard() {
  const userQuery = useQuery({ queryKey: ['user'], queryFn: fetchUser });
  const postsQuery = useQuery({ queryKey: ['posts'], queryFn: fetchPosts });
  const commentsQuery = useQuery({ queryKey: ['comments'], queryFn: fetchComments });

  if (userQuery.isLoading || postsQuery.isLoading || commentsQuery.isLoading) {
    return <div>Loading...</div>;
  }

  return (
    <div>
      <User data={userQuery.data} />
      <Posts data={postsQuery.data} />
      <Comments data={commentsQuery.data} />
    </div>
  );
}

// Or use useQueries for dynamic parallel queries
function DynamicDashboard({ ids }: { ids: string[] }) {
  const queries = useQueries({
    queries: ids.map((id) => ({
      queryKey: ['item', id],
      queryFn: () => fetchItem(id),
    })),
  });

  return (
    <div>
      {queries.map((query, i) => (
        <div key={ids[i]}>
          {query.isLoading ? 'Loading...' : query.data?.name}
        </div>
      ))}
    </div>
  );
}
```

## Query Keys as Dependencies

```typescript
function Post({ postId, userId }: { postId: string; userId: string }) {
  const { data } = useQuery({
    queryKey: ['post', postId, userId],
    queryFn: () => fetchPost(postId, userId),
  });

  // Query will refetch when postId or userId changes
  return <div>{data?.title}</div>;
}
```

## DevTools

```typescript
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <YourApp />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

## Testing

```typescript
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { renderHook, waitFor } from '@testing-library/react';

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
    },
  });

  return ({ children }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
};

test('useQuery hook', async () => {
  const { result } = renderHook(() => useQuery({
    queryKey: ['test'],
    queryFn: () => Promise.resolve('data'),
  }), {
    wrapper: createWrapper(),
  });

  await waitFor(() => expect(result.current.isSuccess).toBe(true));
  expect(result.current.data).toBe('data');
});
```

## Common Mistakes

1. **Not using query keys correctly**: Keys should be arrays
2. **Missing error handling**: Always handle errors
3. **Not leveraging caching**: Understand staleTime vs cacheTime
4. **Mutating cached data**: Use setQueryData correctly
5. **Not invalidating queries**: Update cache after mutations

## Best Practices

1. **Use TypeScript**: Full type safety for queries
2. **Organize query keys**: Consistent naming convention
3. **Handle loading states**: Provide good UX
4. **Use optimistic updates**: Improve perceived performance
5. **Leverage prefetching**: Anticipate user actions
6. **Configure defaults**: Set global defaults on QueryClient
7. **Use DevTools**: Debug and monitor queries

## When to Use TanStack Query

### Good Use Cases

- Server state management
- Data fetching and caching
- Real-time data synchronization
- Pagination and infinite scroll
- Optimistic UI updates

### Not Ideal For

- Local-only state
- Simple one-time fetches
- When you need full control over every request

## Interview Questions

1. **What is the difference between staleTime and cacheTime?**
   - staleTime: how long data is fresh; cacheTime: how long unused data stays in cache

2. **How do optimistic updates work?**
   - Update UI immediately, rollback on error, refetch on success

3. **What are query keys?**
   - Unique identifiers for queries used for caching and refetching

4. **How does automatic refetching work?**
   - Based on window focus, mount, reconnect, and intervals

5. **What is query invalidation?**
   - Marking queries as stale to trigger refetches

## Key Takeaways

- TanStack Query handles server state management
- Automatic caching and background refetching
- Optimistic updates for better UX
- Built-in pagination and infinite scroll
- Query invalidation for data synchronization
- DevTools for debugging
- Excellent TypeScript support
- Framework agnostic (React, Vue, Solid, Svelte)

## Resources

- [Official Documentation](https://tanstack.com/query/latest)
- [Examples](https://tanstack.com/query/latest/docs/react/examples/react/simple)
- [GitHub](https://github.com/TanStack/query)
- [DevTools](https://tanstack.com/query/latest/docs/react/devtools)
