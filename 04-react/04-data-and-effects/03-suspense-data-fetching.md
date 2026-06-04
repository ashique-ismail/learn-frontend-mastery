# Suspense for Data Fetching

## Overview

Suspense is a React feature that lets components "wait" for something before rendering, showing a fallback UI in the meantime. While initially designed for code-splitting with React.lazy(), React 18 expanded Suspense to support data fetching, enabling a more declarative approach to handling loading states.

Suspense fundamentally changes how we think about async operations in React, moving from imperative (checking loading states) to declarative (components suspend while data loads).

## Core Concepts

### What is Suspense?

Suspense allows you to:

1. **Declaratively specify loading states** with fallback UI
2. **Coordinate multiple loading boundaries** in your component tree
3. **Avoid loading waterfalls** by starting requests early
4. **Stream server-rendered content** in React Server Components

### Traditional vs Suspense Approach

```javascript
// Traditional: Imperative loading states
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetchUser(userId)
      .then(setUser)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [userId]);

  if (loading) return <Spinner />;
  if (error) return <Error error={error} />;
  return <div>{user.name}</div>;
}

// Suspense: Declarative loading states
function UserProfile({ userId }) {
  const user = use(fetchUser(userId)); // Suspends while loading
  return <div>{user.name}</div>;
}

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <UserProfile userId={1} />
    </Suspense>
  );
}
```

## Basic Usage

### Suspense with React.lazy()

```javascript
import { lazy, Suspense } from 'react';

const HeavyComponent = lazy(() => import('./HeavyComponent'));

function App() {
  return (
    <Suspense fallback={<div>Loading component...</div>}>
      <HeavyComponent />
    </Suspense>
  );
}
```

### Multiple Boundaries

```javascript
function App() {
  return (
    <div>
      <Suspense fallback={<NavSkeleton />}>
        <Navigation />
      </Suspense>

      <Suspense fallback={<ContentSkeleton />}>
        <Content />
      </Suspense>

      <Suspense fallback={<SidebarSkeleton />}>
        <Sidebar />
      </Suspense>
    </div>
  );
}
```

### Nested Suspense Boundaries

```javascript
function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Layout>
        <Suspense fallback={<HeaderSkeleton />}>
          <Header />
        </Suspense>

        <Suspense fallback={<MainSkeleton />}>
          <MainContent>
            <Suspense fallback={<ArticleSkeleton />}>
              <Article />
            </Suspense>
            <Suspense fallback={<CommentsSkeleton />}>
              <Comments />
            </Suspense>
          </MainContent>
        </Suspense>
      </Layout>
    </Suspense>
  );
}
```

## Data Fetching with Suspense

### The `use` Hook (React 19)

```javascript
import { use, Suspense } from 'react';

// Create a promise-based data source
function fetchUser(userId) {
  return fetch(`/api/users/${userId}`).then(res => res.json());
}

function UserProfile({ userId }) {
  // use() suspends until promise resolves
  const user = use(fetchUser(userId));

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}

function App() {
  return (
    <Suspense fallback={<div>Loading user...</div>}>
      <UserProfile userId={1} />
    </Suspense>
  );
}
```

### Creating Suspense-Compatible Resources

```javascript
// Resource wrapper for Suspense
function wrapPromise(promise) {
  let status = 'pending';
  let result;

  const suspender = promise.then(
    (data) => {
      status = 'success';
      result = data;
    },
    (error) => {
      status = 'error';
      result = error;
    }
  );

  return {
    read() {
      if (status === 'pending') {
        throw suspender; // Suspend!
      }
      if (status === 'error') {
        throw result; // Error boundary catches this
      }
      return result;
    },
  };
}

// Usage
function fetchUser(userId) {
  const promise = fetch(`/api/users/${userId}`).then(r => r.json());
  return wrapPromise(promise);
}

function UserProfile({ userResource }) {
  const user = userResource.read(); // Suspends if pending
  return <div>{user.name}</div>;
}

// Create resource outside component
const userResource = fetchUser(1);

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <UserProfile userResource={userResource} />
    </Suspense>
  );
}
```

### Caching with Suspense

```javascript
// Simple cache for Suspense resources
const cache = new Map();

function createCachedResource(cacheKey, fetcher) {
  if (cache.has(cacheKey)) {
    return cache.get(cacheKey);
  }

  const resource = wrapPromise(fetcher());
  cache.set(cacheKey, resource);
  return resource;
}

function fetchUser(userId) {
  return createCachedResource(`user-${userId}`, () =>
    fetch(`/api/users/${userId}`).then(r => r.json())
  );
}

function UserProfile({ userId }) {
  const userResource = fetchUser(userId);
  const user = userResource.read();
  return <div>{user.name}</div>;
}
```

## Advanced Patterns

### Coordinated Loading States

```javascript
function Dashboard() {
  return (
    <Suspense fallback={<DashboardSkeleton />}>
      {/* All children wait together */}
      <UserStats />
      <RecentActivity />
      <Notifications />
    </Suspense>
  );
}

// vs Independent Loading

function Dashboard() {
  return (
    <div>
      <Suspense fallback={<StatsSkeleton />}>
        <UserStats />
      </Suspense>
      <Suspense fallback={<ActivitySkeleton />}>
        <RecentActivity />
      </Suspense>
      <Suspense fallback={<NotificationsSkeleton />}>
        <Notifications />
      </Suspense>
    </div>
  );
}
```

### Start Fetching Early (Render-as-You-Fetch)

```javascript
// Bad: Fetch-on-Render (waterfall)
function BadProfile({ userId }) {
  return (
    <Suspense fallback={<Spinner />}>
      <User userId={userId} />
      <Posts userId={userId} /> {/* Waits for User to load first */}
    </Suspense>
  );
}

// Good: Render-as-You-Fetch (parallel)
function createProfileData(userId) {
  return {
    user: fetchUser(userId),      // Start immediately
    posts: fetchPosts(userId),    // Start immediately
  };
}

function GoodProfile({ userId }) {
  // Start fetching before render
  const data = useMemo(() => createProfileData(userId), [userId]);

  return (
    <Suspense fallback={<Spinner />}>
      <User userResource={data.user} />
      <Posts postsResource={data.posts} />
    </Suspense>
  );
}
```

### Progressive Enhancement with SuspenseList

```javascript
import { SuspenseList, Suspense } from 'react';

function Feed() {
  return (
    <SuspenseList revealOrder="forwards" tail="collapsed">
      <Suspense fallback={<PostSkeleton />}>
        <Post id={1} />
      </Suspense>
      <Suspense fallback={<PostSkeleton />}>
        <Post id={2} />
      </Suspense>
      <Suspense fallback={<PostSkeleton />}>
        <Post id={3} />
      </Suspense>
    </SuspenseList>
  );
}

// revealOrder options:
// - "forwards": Show in order (1, 2, 3)
// - "backwards": Show in reverse order (3, 2, 1)
// - "together": Wait for all, show together

// tail options:
// - "collapsed": Show one skeleton at a time
// - "hidden": Don't show skeletons for waiting items
```

### Suspense with Transitions

```javascript
import { startTransition, useState, Suspense } from 'react';

function App() {
  const [tab, setTab] = useState('home');

  const handleTabChange = (newTab) => {
    startTransition(() => {
      setTab(newTab); // Mark as low priority
    });
  };

  return (
    <div>
      <button onClick={() => handleTabChange('home')}>Home</button>
      <button onClick={() => handleTabChange('posts')}>Posts</button>
      <button onClick={() => handleTabChange('profile')}>Profile</button>

      <Suspense fallback={<TabSkeleton />}>
        {tab === 'home' && <HomeTab />}
        {tab === 'posts' && <PostsTab />}
        {tab === 'profile' && <ProfileTab />}
      </Suspense>
    </div>
  );
}

// With transitions, UI stays responsive during tab switch
// Old content remains visible until new content is ready
```

## Error Boundaries with Suspense

```javascript
import { Component } from 'react';

class ErrorBoundary extends Component {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    console.error('Error caught:', error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return (
        <div>
          <h2>Something went wrong</h2>
          <p>{this.state.error?.message}</p>
          <button onClick={() => this.setState({ hasError: false })}>
            Try again
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}

// Usage with Suspense
function App() {
  return (
    <ErrorBoundary>
      <Suspense fallback={<Spinner />}>
        <UserProfile userId={1} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

## Real-World Patterns

### Suspense-Enabled Data Fetching Library

```javascript
// Simplified React Query-like implementation
class QueryClient {
  constructor() {
    this.cache = new Map();
    this.promises = new Map();
  }

  fetchQuery(key, fetcher) {
    // Return cached data if available
    if (this.cache.has(key)) {
      return this.cache.get(key);
    }

    // Return in-flight promise if exists
    if (this.promises.has(key)) {
      throw this.promises.get(key);
    }

    // Start new fetch
    const promise = fetcher().then(
      (data) => {
        this.cache.set(key, data);
        this.promises.delete(key);
        return data;
      },
      (error) => {
        this.promises.delete(key);
        throw error;
      }
    );

    this.promises.set(key, promise);
    throw promise; // Suspend
  }

  invalidate(key) {
    this.cache.delete(key);
    this.promises.delete(key);
  }
}

const queryClient = new QueryClient();

function useQuery(key, fetcher) {
  return queryClient.fetchQuery(key, fetcher);
}

// Usage
function UserProfile({ userId }) {
  const user = useQuery(`user-${userId}`, () =>
    fetch(`/api/users/${userId}`).then(r => r.json())
  );

  return <div>{user.name}</div>;
}

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <UserProfile userId={1} />
    </Suspense>
  );
}
```

### Image Loading with Suspense

```javascript
function preloadImage(src) {
  return new Promise((resolve, reject) => {
    const img = new Image();
    img.onload = () => resolve(src);
    img.onerror = reject;
    img.src = src;
  });
}

const imageCache = new Map();

function SuspenseImage({ src, alt, ...props }) {
  if (!imageCache.has(src)) {
    throw preloadImage(src).then(() => {
      imageCache.set(src, true);
    });
  }

  return <img src={src} alt={alt} {...props} />;
}

// Usage
function Gallery() {
  return (
    <div>
      {images.map(src => (
        <Suspense key={src} fallback={<ImageSkeleton />}>
          <SuspenseImage src={src} alt="Gallery image" />
        </Suspense>
      ))}
    </div>
  );
}
```

### Lazy Route Loading

```javascript
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';

const Home = lazy(() => import('./routes/Home'));
const About = lazy(() => import('./routes/About'));
const Contact = lazy(() => import('./routes/Contact'));

function App() {
  return (
    <Suspense fallback={<PageSpinner />}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
        <Route path="/contact" element={<Contact />} />
      </Routes>
    </Suspense>
  );
}
```

## Best Practices

### 1. Place Suspense Boundaries Strategically

```javascript
// Good: Boundary at appropriate level
function App() {
  return (
    <div>
      <Header /> {/* Non-suspending */}
      <Suspense fallback={<ContentSkeleton />}>
        <MainContent /> {/* May suspend */}
      </Suspense>
      <Footer /> {/* Non-suspending */}
    </div>
  );
}

// Bad: Too granular
function App() {
  return (
    <div>
      <Suspense fallback={<div>Loading...</div>}>
        <Header />
      </Suspense>
      <Suspense fallback={<div>Loading...</div>}>
        <MainContent />
      </Suspense>
      <Suspense fallback={<div>Loading...</div>}>
        <Footer />
      </Suspense>
    </div>
  );
}
```

### 2. Provide Meaningful Fallbacks

```javascript
// Good: Skeleton matches layout
<Suspense fallback={<ProfileSkeleton />}>
  <UserProfile />
</Suspense>

// Bad: Generic spinner
<Suspense fallback={<Spinner />}>
  <UserProfile />
</Suspense>
```

### 3. Start Fetching Early

```javascript
// Good: Start fetch before render
const userResource = fetchUser(userId);

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <UserProfile userResource={userResource} />
    </Suspense>
  );
}

// Bad: Fetch in component (slower)
function UserProfile({ userId }) {
  const user = use(fetchUser(userId)); // Late start
  return <div>{user.name}</div>;
}
```

### 4. Combine with Error Boundaries

```javascript
// Always wrap Suspense with ErrorBoundary
<ErrorBoundary fallback={<ErrorMessage />}>
  <Suspense fallback={<Spinner />}>
    <Component />
  </Suspense>
</ErrorBoundary>
```

## Common Mistakes

### 1. Creating Resources Inside Components

```javascript
// Bad: Creates new resource on every render
function UserProfile({ userId }) {
  const userResource = fetchUser(userId); // New promise every render!
  const user = userResource.read();
  return <div>{user.name}</div>;
}

// Good: Memoize or create outside
function UserProfile({ userId }) {
  const userResource = useMemo(() => fetchUser(userId), [userId]);
  const user = userResource.read();
  return <div>{user.name}</div>;
}
```

### 2. Not Handling Errors

```javascript
// Bad: No error boundary
<Suspense fallback={<Spinner />}>
  <Component /> {/* Errors crash the app */}
</Suspense>

// Good: Error boundary wraps Suspense
<ErrorBoundary>
  <Suspense fallback={<Spinner />}>
    <Component />
  </Suspense>
</ErrorBoundary>
```

### 3. Suspense Waterfalls

```javascript
// Bad: Sequential loading
function Profile() {
  return (
    <Suspense fallback={<Spinner />}>
      <User /> {/* Loads first */}
      <Posts /> {/* Loads after User */}
    </Suspense>
  );
}

// Good: Parallel loading
function Profile() {
  const data = useMemo(() => ({
    user: fetchUser(),  // Start both immediately
    posts: fetchPosts(),
  }), []);

  return (
    <Suspense fallback={<Spinner />}>
      <User resource={data.user} />
      <Posts resource={data.posts} />
    </Suspense>
  );
}
```

## Testing

```javascript
import { render, screen } from '@testing-library/react';
import { Suspense } from 'react';

test('shows fallback while loading', () => {
  render(
    <Suspense fallback={<div>Loading...</div>}>
      <AsyncComponent />
    </Suspense>
  );

  expect(screen.getByText('Loading...')).toBeInTheDocument();
});

test('shows content after loading', async () => {
  render(
    <Suspense fallback={<div>Loading...</div>}>
      <AsyncComponent />
    </Suspense>
  );

  expect(await screen.findByText('Content')).toBeInTheDocument();
  expect(screen.queryByText('Loading...')).not.toBeInTheDocument();
});

test('handles errors with error boundary', async () => {
  const ThrowError = () => {
    throw new Error('Test error');
  };

  render(
    <ErrorBoundary>
      <Suspense fallback={<div>Loading...</div>}>
        <ThrowError />
      </Suspense>
    </ErrorBoundary>
  );

  expect(await screen.findByText(/Test error/)).toBeInTheDocument();
});
```

## Library Support

### TanStack Query (React Query)

```javascript
import { useSuspenseQuery } from '@tanstack/react-query';

function UserProfile({ userId }) {
  const { data: user } = useSuspenseQuery({
    queryKey: ['user', userId],
    queryFn: () => fetch(`/api/users/${userId}`).then(r => r.json()),
  });

  return <div>{user.name}</div>;
}

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <UserProfile userId={1} />
    </Suspense>
  );
}
```

### SWR

```javascript
import useSWR from 'swr';

function UserProfile({ userId }) {
  const { data: user } = useSWR(`/api/users/${userId}`, {
    suspense: true,
  });

  return <div>{user.name}</div>;
}

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <UserProfile userId={1} />
    </Suspense>
  );
}
```

## Key Takeaways

1. **Declarative Loading**: Suspense makes loading states declarative
2. **Coordinated Loading**: Group related components in one boundary
3. **Start Early**: Fetch data before rendering for best performance
4. **Error Boundaries**: Always combine Suspense with error boundaries
5. **Strategic Boundaries**: Place Suspense at appropriate levels
6. **Meaningful Fallbacks**: Use skeleton screens over generic spinners
7. **Avoid Waterfalls**: Start parallel requests simultaneously
8. **Library Support**: Use libraries like React Query for production apps

## Additional Resources

- [React Suspense Documentation](https://react.dev/reference/react/Suspense)
- [Building Great User Experiences with Concurrent Mode](https://legacy.reactjs.org/blog/2019/11/06/building-great-user-experiences-with-concurrent-mode-and-suspense.html)
- [TanStack Query Suspense](https://tanstack.com/query/latest/docs/react/guides/suspense)
- [React Server Components](https://react.dev/reference/react/use-server)
