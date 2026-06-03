# Suspense Internals

## Overview

Suspense is React's mechanism for handling asynchronous operations declaratively. Underneath its simple API lies a sophisticated system based on error boundaries, promise throwing, rendering phases, and coordination with concurrent rendering. Understanding Suspense internals reveals how React can "pause" rendering, show fallbacks, and resume when data is ready.

This guide explores the throw-promise pattern, Suspense boundary implementation, rendering phases, and how Suspense integrates with concurrent features.

## The Throw-Promise Pattern

Suspense works through a clever pattern: components throw promises to signal they're not ready:

```javascript
// Example 1: Basic Suspense pattern
const resource = createResource(fetchUser);

function UserProfile() {
  // read() throws a promise if data isn't ready
  const user = resource.read();
  
  return <div>{user.name}</div>;
}

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <UserProfile />
    </Suspense>
  );
}

// Example 2: Simplified resource implementation
function createResource(fetcher) {
  let status = 'pending';
  let result;
  let suspender = fetcher().then(
    data => {
      status = 'success';
      result = data;
    },
    error => {
      status = 'error';
      result = error;
    }
  );
  
  return {
    read() {
      if (status === 'pending') {
        // Throw the promise - React will catch it
        throw suspender;
      } else if (status === 'error') {
        throw result;
      } else {
        return result;
      }
    }
  };
}
```

How React catches thrown promises:

```javascript
// Example 3: Simplified Suspense boundary catching
function renderWithSuspense(Component) {
  try {
    // Try to render component
    return Component();
  } catch (thrown) {
    if (isPromise(thrown)) {
      // Component suspended - show fallback
      return handleSuspense(thrown);
    } else {
      // Regular error - use error boundary
      throw thrown;
    }
  }
}

function isPromise(value) {
  return (
    value !== null &&
    typeof value === 'object' &&
    typeof value.then === 'function'
  );
}
```

## Suspense Boundary Implementation

Suspense boundaries track suspended components and manage fallback state:

```javascript
// Example 4: Internal Suspense boundary structure
type SuspenseState = {
  dehydrated: null | SuspenseInstance;
  retryLane: Lane;
};

type SuspenseComponent = {
  pendingProps: {
    children: ReactNode;
    fallback: ReactNode;
  };
  memoizedState: SuspenseState | null;
  // ... other fiber properties
};

// Example 5: Suspense boundary behavior
function SuspenseBoundary({ children, fallback }) {
  // Internal state tracks suspended children
  const [showFallback, setShowFallback] = useState(false);
  const [suspendedPromises, setSuspendedPromises] = useState([]);
  
  const handleSuspend = (promise) => {
    setShowFallback(true);
    setSuspendedPromises(prev => [...prev, promise]);
    
    promise.then(() => {
      // When promise resolves, retry render
      setSuspendedPromises(prev => prev.filter(p => p !== promise));
      if (suspendedPromises.length === 1) {
        setShowFallback(false);
        // Trigger re-render to try children again
      }
    });
  };
  
  return showFallback ? fallback : children;
}
```

## Rendering Phases with Suspense

```javascript
// Example 6: Suspense rendering phases
function renderSuspenseComponent(workInProgress) {
  const nextProps = workInProgress.pendingProps;
  
  // Phase 1: Try to render children
  let nextChildren;
  try {
    nextChildren = renderChildren(nextProps.children);
  } catch (thrown) {
    if (isPromise(thrown)) {
      // Phase 2: Child suspended
      if (workInProgress.memoizedState === null) {
        // First suspension - mount fallback
        workInProgress.memoizedState = {
          dehydrated: null,
          retryLane: NoLane
        };
      }
      
      // Attach retry listener
      attachSuspenseRetryListeners(workInProgress, thrown);
      
      // Render fallback
      return mountSuspenseFallback(
        workInProgress,
        nextProps.fallback
      );
    } else {
      // Regular error - rethrow
      throw thrown;
    }
  }
  
  // Phase 3: Children rendered successfully
  return nextChildren;
}

// Example 7: Retry mechanism
function attachSuspenseRetryListeners(suspenseBoundary, promise) {
  const retry = () => {
    // Schedule render to retry children
    const lane = pickArbitraryLane(suspenseBoundary);
    suspenseBoundary.lanes |= lane;
    scheduleUpdateOnFiber(suspenseBoundary, lane);
  };
  
  promise.then(retry, retry);
}
```

## Multiple Suspense Boundaries

```javascript
// Example 8: Nested Suspense boundaries
function App() {
  return (
    <Suspense fallback={<PageLoader />}>
      <Header />
      <Suspense fallback={<SidebarLoader />}>
        <Sidebar />
      </Suspense>
      <Suspense fallback={<ContentLoader />}>
        <MainContent />
      </Suspense>
    </Suspense>
  );
}

// Suspension propagation:
// 1. MainContent suspends
// 2. Nearest Suspense (ContentLoader) catches it
// 3. Shows ContentLoader fallback
// 4. Header and Sidebar keep showing
//
// If Header suspends:
// 1. Header is above nested Suspense boundaries
// 2. Outer Suspense (PageLoader) catches it
// 3. Shows PageLoader fallback
// 4. Entire page replaced with loader
```

```javascript
// Example 9: Sibling suspense coordination
function Dashboard() {
  return (
    <div>
      <Suspense fallback={<Skeleton />}>
        <UserWidget />
      </Suspense>
      <Suspense fallback={<Skeleton />}>
        <StatsWidget />
      </Suspense>
      <Suspense fallback={<Skeleton />}>
        <ActivityWidget />
      </Suspense>
    </div>
  );
}

// Each widget suspends independently:
// - UserWidget suspends → shows its Skeleton
// - StatsWidget suspends → shows its Skeleton
// - ActivityWidget suspends → shows its Skeleton
// As data arrives, each replaces its skeleton independently
```

## SuspenseList (Experimental)

```javascript
// Example 10: SuspenseList coordination
function OrderedList() {
  return (
    <SuspenseList revealOrder="forwards">
      <Suspense fallback={<Skeleton />}>
        <Item1 />
      </Suspense>
      <Suspense fallback={<Skeleton />}>
        <Item2 />
      </Suspense>
      <Suspense fallback={<Skeleton />}>
        <Item3 />
      </Suspense>
    </SuspenseList>
  );
}

// With revealOrder="forwards":
// - All items start loading simultaneously
// - Item2 won't reveal until Item1 is ready
// - Item3 won't reveal until Item2 is ready
// Even if Item3 loads first, it waits for Item1 and Item2
//
// With revealOrder="backwards":
// - Item3 reveals first
// - Item2 reveals next
// - Item1 reveals last
//
// With revealOrder="together":
// - All items wait until all are ready
// - All reveal simultaneously
```

## Suspense with Transitions

```javascript
// Example 11: Suspense + transitions
function UserProfile({ userId }) {
  const user = use(fetchUser(userId));
  return <div>{user.name}</div>;
}

function App() {
  const [userId, setUserId] = useState(1);
  const [isPending, startTransition] = useTransition();
  
  const handleChange = (newId) => {
    startTransition(() => {
      setUserId(newId);
    });
  };
  
  return (
    <div>
      <button onClick={() => handleChange(2)}>
        Change User
      </button>
      {isPending && <InlineSpinner />}
      <Suspense fallback={<BigLoader />}>
        <UserProfile userId={userId} />
      </Suspense>
    </div>
  );
}

// Behavior without transition:
// 1. User clicks button
// 2. UserProfile suspends immediately
// 3. BigLoader replaces current content
// 4. Jarring experience
//
// Behavior with transition:
// 1. User clicks button
// 2. React keeps showing old UserProfile
// 3. isPending becomes true, shows InlineSpinner
// 4. Renders new UserProfile in background
// 5. When ready, swaps to new content
// 6. No BigLoader shown because update is non-urgent
```

## Cache Integration

```javascript
// Example 12: Suspense with cache
const userCache = new Map();

function fetchUserWithCache(userId) {
  if (userCache.has(userId)) {
    return userCache.get(userId);
  }
  
  const promise = fetch(`/api/users/${userId}`)
    .then(res => res.json())
    .then(data => {
      userCache.set(userId, data);
      return data;
    });
  
  // Throw promise on first access
  throw promise;
}

function UserProfile({ userId }) {
  const user = fetchUserWithCache(userId);
  return <div>{user.name}</div>;
}

// First render with userId=1:
// 1. fetchUserWithCache called
// 2. Cache miss, throws promise
// 3. Suspense catches, shows fallback
// 4. Promise resolves, React retries
// 5. Cache hit, returns data
// 6. Renders successfully
//
// Second render with userId=1:
// 1. fetchUserWithCache called
// 2. Cache hit, returns immediately
// 3. No suspension
```

## Error Handling with Suspense

```javascript
// Example 13: Suspense + error boundaries
function DataComponent() {
  const data = use(fetchData()); // May suspend or error
  return <div>{data.value}</div>;
}

function App() {
  return (
    <ErrorBoundary fallback={<ErrorUI />}>
      <Suspense fallback={<Loading />}>
        <DataComponent />
      </Suspense>
    </ErrorBoundary>
  );
}

// Scenario 1: Suspension
// 1. DataComponent throws promise
// 2. Suspense catches, shows Loading
// 3. Promise resolves successfully
// 4. Shows DataComponent
//
// Scenario 2: Error
// 1. DataComponent throws promise
// 2. Suspense catches, shows Loading
// 3. Promise rejects
// 4. React retries render
// 5. DataComponent throws error
// 6. ErrorBoundary catches, shows ErrorUI
```

## Suspense with Server-Side Rendering

```javascript
// Example 14: SSR with Suspense
function ServerApp() {
  return (
    <html>
      <body>
        <Suspense fallback={<HeaderSkeleton />}>
          <Header />
        </Suspense>
        <Suspense fallback={<MainSkeleton />}>
          <MainContent />
        </Suspense>
      </body>
    </html>
  );
}

// SSR streaming behavior:
// 1. Server starts streaming HTML
// 2. Sends <HeaderSkeleton /> and <MainSkeleton />
// 3. Header data ready → sends replacement script
// 4. Browser replaces HeaderSkeleton with Header
// 5. MainContent data ready → sends replacement script
// 6. Browser replaces MainSkeleton with MainContent
//
// Client receives HTML progressively without waiting for all data
```

## Dehydration and Hydration

```javascript
// Example 15: Suspense hydration
// Server renders:
<div id="root">
  <div>Loading...</div>  <!-- Suspense fallback -->
</div>

// Client hydrates:
function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <AsyncComponent />
    </Suspense>
  );
}

// Hydration behavior:
// 1. React sees Suspense boundary with fallback in HTML
// 2. Knows AsyncComponent suspended on server
// 3. Starts hydrating but keeps fallback visible
// 4. Fetches data for AsyncComponent
// 5. When ready, replaces fallback with real content
// 6. Continues hydration
```

## Suspense Retry Logic

```javascript
// Example 16: Internal retry mechanism
function retrySuspenseBoundary(suspenseBoundary) {
  const retryLane = suspenseBoundary.memoizedState.retryLane;
  
  if (retryLane !== NoLane) {
    // Retry at the lane that suspended
    scheduleUpdateOnFiber(suspenseBoundary, retryLane);
  } else {
    // Default retry
    const lane = pickArbitraryLane(suspenseBoundary);
    scheduleUpdateOnFiber(suspenseBoundary, lane);
  }
}

// Example 17: Custom retry logic
function CustomRetry() {
  const [attempt, setAttempt] = useState(0);
  
  return (
    <Suspense fallback={<Loading />}>
      <DataComponent key={attempt} />
      <button onClick={() => setAttempt(a => a + 1)}>
        Retry
      </button>
    </Suspense>
  );
}

// Changing key forces remount, triggering new fetch
```

## Advanced Patterns

```javascript
// Example 18: Preloading data
const resourceMap = new Map();

function preloadUser(userId) {
  if (!resourceMap.has(userId)) {
    const resource = createResource(() => fetchUser(userId));
    resourceMap.set(userId, resource);
    // Start fetch but don't wait
    try {
      resource.read();
    } catch (promise) {
      // Expected to throw, ignore
    }
  }
}

function UserLink({ userId, children }) {
  return (
    <a
      href={`/users/${userId}`}
      onMouseEnter={() => preloadUser(userId)}
      onClick={(e) => {
        e.preventDefault();
        navigateToUser(userId);
      }}
    >
      {children}
    </a>
  );
}

// Hover triggers preload, click navigates immediately
// Data likely ready by the time user clicks

// Example 19: Waterfall prevention
function UserProfile({ userId }) {
  // BAD: Waterfall
  const user = use(fetchUser(userId));
  const posts = use(fetchPosts(userId)); // Waits for user
  
  return <div>{user.name} - {posts.length} posts</div>;
}

// GOOD: Parallel loading
function preloadUserData(userId) {
  const userResource = createResource(() => fetchUser(userId));
  const postsResource = createResource(() => fetchPosts(userId));
  
  return { userResource, postsResource };
}

function BetterUserProfile({ userId }) {
  const { userResource, postsResource } = preloadUserData(userId);
  
  // Both fetch in parallel
  const user = userResource.read();
  const posts = postsResource.read();
  
  return <div>{user.name} - {posts.length} posts</div>;
}
```

## Common Pitfalls

```javascript
// Example 20: Suspending in useEffect
function BrokenComponent() {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    // WRONG: Can't suspend in effects
    const result = suspenseResource.read();
    setData(result);
  }, []);
  
  return <div>{data?.value}</div>;
}

// FIX: Suspend during render, not in effects
function FixedComponent() {
  const data = suspenseResource.read();
  return <div>{data.value}</div>;
}

// Example 21: Conditional suspension
function ConditionalSuspend({ shouldLoad }) {
  if (shouldLoad) {
    // WRONG: Conditional suspension can cause issues
    const data = resource.read();
    return <div>{data.value}</div>;
  }
  return <div>Not loading</div>;
}

// Better: Always suspend, conditionally render
function BetterConditional({ shouldLoad }) {
  const data = resource.read(); // Always suspend or return
  
  if (!shouldLoad) {
    return <div>Not loading</div>;
  }
  
  return <div>{data.value}</div>;
}
```

## Common Misconceptions

1. **"Suspense is just for data fetching"** - False. Suspense works with any async operation: code splitting, images, etc. Anything that can throw a promise works.

2. **"Suspense makes API calls"** - False. Suspense is just a boundary. Components throw promises; Suspense catches them and shows fallbacks.

3. **"You need a special data library"** - False. You can implement the throw-promise pattern yourself, though libraries like React Query make it easier.

4. **"Suspense always shows fallback"** - False. With transitions, React can keep showing old UI instead of fallback during background updates.

5. **"Suspense is production-ready for data fetching"** - Partially true. It works, but you need a good caching/coordination layer. Most apps still use hooks-based fetching.

## Performance Implications

1. **Fallback Thrashing** - Showing/hiding fallbacks rapidly creates poor UX. Use transitions or SuspenseList to coordinate.

2. **Cache Invalidation** - Without proper caching, components re-suspend on every render. Implement cache keys and invalidation logic.

3. **Waterfall Fetches** - Sequential read() calls create waterfalls. Start all fetches in parallel before rendering.

4. **Overly Granular Boundaries** - Too many small Suspense boundaries create visual noise. Group related content under shared boundaries.

## Interview Questions

1. **Q: How does Suspense work internally?**
   A: Suspense uses the throw-promise pattern. Components throw promises when data isn't ready. Suspense boundaries catch these promises, show fallback UI, attach then() handlers to retry rendering, and swap to actual content when promises resolve.

2. **Q: What's the difference between Suspense and error boundaries?**
   A: Both catch thrown values during render. Error boundaries catch errors (Error instances) and show error UI permanently. Suspense catches promises, shows temporary fallback UI, and retries rendering when promises resolve.

3. **Q: How do multiple Suspense boundaries interact?**
   A: Thrown promises bubble up to the nearest Suspense boundary, like error boundaries. Nested boundaries allow granular control—inner boundaries handle specific sections while outer boundaries catch suspensions from higher-level components.

4. **Q: Explain how Suspense integrates with transitions.**
   A: During transitions, Suspense shows old UI instead of fallback when new content suspends. React renders the new tree in the background, and if it suspends, keeps the old tree visible rather than immediately showing the fallback. This prevents jarring loading states.

5. **Q: How does Suspense enable streaming SSR?**
   A: During SSR, Suspense boundaries that suspend send their fallback HTML immediately while continuing to render other parts. When suspended content becomes ready, the server streams replacement HTML and JavaScript to swap out the fallback, enabling progressive page loading.

6. **Q: What is the retry mechanism in Suspense?**
   A: When a promise thrown by a suspended component resolves, Suspense schedules an update on the boundary fiber, triggering React to retry rendering the children. If successful, children replace the fallback. If they suspend again, the cycle repeats.

7. **Q: Why can't you suspend in useEffect?**
   A: Effects run after commit, outside the render phase. Suspense works by catching promises during render. Throwing from an effect would escape React's try-catch, resulting in an unhandled promise rejection. Suspension must happen during render.

8. **Q: How do you prevent Suspense waterfalls?**
   A: Start all data fetches before rendering rather than during render. Create resources/queries upfront, then pass them to components. This allows parallel fetching instead of sequential read() calls that wait for each other.

## Key Takeaways

1. Suspense catches promises thrown during render and shows fallback UI
2. Components signal they're not ready by throwing promises
3. Suspense boundaries retry rendering when caught promises resolve
4. Multiple boundaries provide granular control over loading states
5. Transitions keep old UI visible when new content suspends
6. Suspense enables streaming SSR by sending fallbacks immediately
7. Proper caching prevents unnecessary re-suspension
8. Start fetches early to avoid waterfalls

## Resources

- [Suspense Documentation](https://react.dev/reference/react/Suspense)
- [Data Fetching with Suspense](https://react.dev/blog/2022/03/29/react-v18#suspense-in-data-frameworks)
- [Streaming Server Rendering with Suspense](https://github.com/reactwg/react-18/discussions/37)
- [Building Your Own Suspense Cache](https://blog.logrocket.com/data-fetching-react-suspense/)
- [React Suspense Source Code](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberSuspenseComponent.js)
