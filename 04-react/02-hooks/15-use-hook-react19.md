# use Hook (React 19)

## The Idea

**In plain English:** The `use` hook is a special tool in React 19 that lets your component pause and wait for data (like information fetched from the internet) before it finishes drawing itself on the screen. A "hook" is just a built-in React function that gives your component extra abilities.

**Real-world analogy:** Imagine you're at a restaurant and you order food. The waiter takes your order to the kitchen, then comes back only when the food is ready to serve it to you. While the kitchen is cooking, other tables are still being served.

- The waiter taking your order = the `use` hook asking for data (a Promise)
- The kitchen cooking your food = the server fetching the data in the background
- You waiting at your table without blocking others = React showing a loading spinner (Suspense) while other parts of the page still work
- The waiter returning with your plate = the `use` hook receiving the finished data so the component can render

---

## Overview

The `use` hook is a **React 19** feature that allows you to read the value of resources like Promises and Context within render. Unlike other hooks, `use` can be called conditionally and in loops.

```jsx
const value = use(resource);
```

**Resources supported:**

- Promises
- Context

## Why use Exists

### Reading Promises in Render

Before React 19, you couldn't read Promise values directly in render:

```jsx
// Old way: useEffect + useState
function Component() {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    fetchData().then(setData);
  }, []);
  
  if (!data) return <Loading />;
  return <div>{data.title}</div>;
}

// React 19: use() with Suspense
function Component({ dataPromise }) {
  const data = use(dataPromise);
  return <div>{data.title}</div>;
}
```

### Conditional Context Reading

```jsx
// Old way: Can't conditionally read context
function Component({ useTheme }) {
  // Error: Can't conditionally call useContext
  const theme = useTheme ? useContext(ThemeContext) : null;
}

// React 19: use() can be conditional
function Component({ useTheme }) {
  const theme = useTheme ? use(ThemeContext) : null;
  return <div style={{ color: theme?.color }}>Content</div>;
}
```

## use with Promises

### Basic Promise Reading

```jsx
import { use, Suspense } from 'react';

function UserProfile({ userPromise }) {
  // use() suspends until promise resolves
  const user = use(userPromise);
  
  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}

// Parent must wrap in Suspense
function App() {
  const userPromise = fetchUser(123);
  
  return (
    <Suspense fallback={<div>Loading user...</div>}>
      <UserProfile userPromise={userPromise} />
    </Suspense>
  );
}
```

### How It Works

```jsx
function use(promise) {
  // 1. If promise is pending: throw promise (triggers Suspense)
  // 2. If promise resolved: return value
  // 3. If promise rejected: throw error (caught by Error Boundary)
}
```

### With Error Boundaries

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false };
  
  static getDerivedStateFromError(error) {
    return { hasError: true };
  }
  
  render() {
    if (this.state.hasError) {
      return <div>Error loading data</div>;
    }
    return this.props.children;
  }
}

function App() {
  return (
    <ErrorBoundary>
      <Suspense fallback={<Loading />}>
        <UserProfile userPromise={fetchUser(123)} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

## use with Context

### Basic Context Reading

```jsx
import { createContext, use } from 'react';

const ThemeContext = createContext(null);

function Button() {
  const theme = use(ThemeContext);
  
  return (
    <button style={{ background: theme.background }}>
      Click me
    </button>
  );
}

function App() {
  return (
    <ThemeContext.Provider value={{ background: 'blue' }}>
      <Button />
    </ThemeContext.Provider>
  );
}
```

### Conditional Context Reading with use

```jsx
function Component({ needsTheme }) {
  // This is allowed with use()! (not allowed with useContext)
  const theme = needsTheme ? use(ThemeContext) : null;
  
  return (
    <div style={needsTheme ? { color: theme.color } : {}}>
      Content
    </div>
  );
}
```

### In Loops

```jsx
function ComponentList({ items, contexts }) {
  return items.map((item, index) => {
    // Can call use() in a loop!
    const context = use(contexts[index]);
    
    return (
      <div key={item.id} style={{ color: context.color }}>
        {item.name}
      </div>
    );
  });
}
```

## Practical Examples

### Example 1: Data Fetching with use

```jsx
// API wrapper that returns promises
function fetchUser(id) {
  return fetch(`/api/users/${id}`).then(r => r.json());
}

function UserCard({ userId }) {
  const user = use(fetchUser(userId));
  
  return (
    <div className="user-card">
      <img src={user.avatar} alt={user.name} />
      <h2>{user.name}</h2>
      <p>{user.bio}</p>
    </div>
  );
}

function UserList({ userIds }) {
  return (
    <div>
      {userIds.map(id => (
        <Suspense key={id} fallback={<CardSkeleton />}>
          <UserCard userId={id} />
        </Suspense>
      ))}
    </div>
  );
}
```

### Example 2: Parallel Data Fetching

```jsx
function Dashboard() {
  // Start all fetches in parallel
  const userPromise = fetchUser();
  const statsPromise = fetchStats();
  const notificationsPromise = fetchNotifications();
  
  return (
    <div>
      <Suspense fallback={<UserSkeleton />}>
        <UserProfile promise={userPromise} />
      </Suspense>
      
      <Suspense fallback={<StatsSkeleton />}>
        <Stats promise={statsPromise} />
      </Suspense>
      
      <Suspense fallback={<NotificationsSkeleton />}>
        <Notifications promise={notificationsPromise} />
      </Suspense>
    </div>
  );
}

function UserProfile({ promise }) {
  const user = use(promise);
  return <div>{user.name}</div>;
}

function Stats({ promise }) {
  const stats = use(promise);
  return <div>Posts: {stats.posts}</div>;
}

function Notifications({ promise }) {
  const notifications = use(promise);
  return <div>{notifications.length} new</div>;
}
```

### Example 3: Waterfall vs Parallel

```jsx
// Bad: Waterfall (sequential loading)
function BadDashboard() {
  return (
    <Suspense fallback={<Loading />}>
      <UserProfile />  {/* Waits */}
      <Posts />        {/* Then waits */}
      <Comments />     {/* Then waits */}
    </Suspense>
  );
}

function UserProfile() {
  const user = use(fetchUser());  // Fetch 1
  return <div>{user.name}</div>;
}

function Posts() {
  const posts = use(fetchPosts());  // Fetch 2 (after Fetch 1)
  return <div>{posts.length} posts</div>;
}

// Good: Parallel loading
function GoodDashboard() {
  // Start all fetches immediately
  const userPromise = fetchUser();
  const postsPromise = fetchPosts();
  const commentsPromise = fetchComments();
  
  return (
    <Suspense fallback={<Loading />}>
      <UserProfile promise={userPromise} />
      <Posts promise={postsPromise} />
      <Comments promise={commentsPromise} />
    </Suspense>
  );
}

function UserProfile({ promise }) {
  const user = use(promise);
  return <div>{user.name}</div>;
}

function Posts({ promise }) {
  const posts = use(promise);
  return <div>{posts.length} posts</div>;
}
```

### Example 4: Conditional Data Fetching

```jsx
function UserProfile({ userId, showDetails }) {
  const user = use(fetchUser(userId));
  
  // Conditional use of promise!
  const details = showDetails 
    ? use(fetchUserDetails(userId))
    : null;
  
  return (
    <div>
      <h1>{user.name}</h1>
      {showDetails && (
        <div>
          <p>Phone: {details.phone}</p>
          <p>Address: {details.address}</p>
        </div>
      )}
    </div>
  );
}
```

### Example 5: Resource Pattern

```jsx
// Create resource wrapper
function createResource(promise) {
  let status = 'pending';
  let result;
  
  const suspender = promise.then(
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
      if (status === 'pending') throw suspender;
      if (status === 'error') throw result;
      return result;
    }
  };
}

// Can be used with use()
function Component({ resource }) {
  const data = use(resource.read());
  return <div>{data.title}</div>;
}

// Create resource outside component
const userResource = createResource(fetchUser(123));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <Component resource={userResource} />
    </Suspense>
  );
}
```

### Example 6: Conditional Context

```jsx
const AuthContext = createContext(null);
const ThemeContext = createContext(null);

function Button({ useAuth, useTheme }) {
  // Can conditionally read contexts!
  const auth = useAuth ? use(AuthContext) : null;
  const theme = useTheme ? use(ThemeContext) : null;
  
  return (
    <button
      onClick={auth?.logout}
      style={{ background: theme?.primary }}
    >
      {auth ? 'Logout' : 'Click me'}
    </button>
  );
}

function App() {
  return (
    <AuthContext.Provider value={{ user: 'John', logout: () => {} }}>
      <ThemeContext.Provider value={{ primary: 'blue' }}>
        <Button useAuth useTheme />
        <Button useAuth={false} useTheme />
      </ThemeContext.Provider>
    </AuthContext.Provider>
  );
}
```

### Example 7: Progressive Enhancement

```jsx
function ArticleList({ articlesPromise, priority = 'high' }) {
  return (
    <>
      {priority === 'high' ? (
        <Suspense fallback={<Skeleton />}>
          <Articles promise={articlesPromise} />
        </Suspense>
      ) : (
        <LazyArticles promise={articlesPromise} />
      )}
    </>
  );
}

function Articles({ promise }) {
  const articles = use(promise);
  return articles.map(a => <Article key={a.id} {...a} />);
}

function LazyArticles({ promise }) {
  const [show, setShow] = useState(false);
  
  return (
    <>
      <button onClick={() => setShow(true)}>Load Articles</button>
      {show && (
        <Suspense fallback={<Skeleton />}>
          <Articles promise={promise} />
        </Suspense>
      )}
    </>
  );
}
```

## Advanced Patterns

### Pattern 1: Deduplication

```jsx
const cache = new Map();

function fetchWithCache(url) {
  if (cache.has(url)) {
    return cache.get(url);
  }
  
  const promise = fetch(url).then(r => r.json());
  cache.set(url, promise);
  return promise;
}

function Component({ url }) {
  // Multiple components fetching same URL will use cached promise
  const data = use(fetchWithCache(url));
  return <div>{data.title}</div>;
}
```

### Pattern 2: Prefetching

```jsx
function ArticleList() {
  const articles = use(fetchArticles());
  
  return articles.map(article => (
    <Link
      key={article.id}
      to={`/article/${article.id}`}
      onMouseEnter={() => {
        // Prefetch on hover
        fetchArticle(article.id);
      }}
    >
      {article.title}
    </Link>
  ));
}

function ArticlePage({ id }) {
  // Might already be in cache from prefetch
  const article = use(fetchArticle(id));
  return <div>{article.content}</div>;
}
```

### Pattern 3: Retry Logic

```jsx
function fetchWithRetry(url, retries = 3) {
  return fetch(url)
    .then(r => {
      if (!r.ok) throw new Error('Failed');
      return r.json();
    })
    .catch(error => {
      if (retries > 0) {
        return fetchWithRetry(url, retries - 1);
      }
      throw error;
    });
}

function Component({ url }) {
  const data = use(fetchWithRetry(url));
  return <div>{data.title}</div>;
}
```

### Pattern 4: Timeout Handling

```jsx
function fetchWithTimeout(url, timeout = 5000) {
  return Promise.race([
    fetch(url).then(r => r.json()),
    new Promise((_, reject) => 
      setTimeout(() => reject(new Error('Timeout')), timeout)
    )
  ]);
}

function Component({ url }) {
  const data = use(fetchWithTimeout(url));
  return <div>{data.title}</div>;
}
```

## TypeScript

```tsx
import { use, Suspense } from 'react';

interface User {
  id: number;
  name: string;
  email: string;
}

function fetchUser(id: number): Promise<User> {
  return fetch(`/api/users/${id}`).then(r => r.json());
}

interface UserProfileProps {
  userPromise: Promise<User>;
}

function UserProfile({ userPromise }: UserProfileProps) {
  const user = use(userPromise);
  
  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}

// With Context
interface Theme {
  primary: string;
  secondary: string;
}

const ThemeContext = createContext<Theme | null>(null);

function Button() {
  const theme = use(ThemeContext);
  
  if (!theme) {
    return <button>No theme</button>;
  }
  
  return (
    <button style={{ background: theme.primary }}>
      Themed
    </button>
  );
}

// Conditional use
function ConditionalComponent({ needsTheme }: { needsTheme: boolean }) {
  const theme = needsTheme ? use(ThemeContext) : null;
  
  return (
    <div style={theme ? { color: theme.primary } : {}}>
      Content
    </div>
  );
}
```

## Rules and Constraints

### What's Different from Other Hooks

```jsx
// ✓ use() can be conditional
if (condition) {
  const value = use(promise);
}

// ✓ use() can be in loops
for (let i = 0; i < 10; i++) {
  const value = use(promises[i]);
}

// ✓ use() can be in callbacks (if called during render)
function Component() {
  const data = someFunction(() => use(promise));
  return <div>{data}</div>;
}

// ✗ use() cannot be in event handlers
function Component() {
  const handleClick = () => {
    const data = use(promise);  // Error!
  };
}

// ✗ use() cannot be in useEffect
function Component() {
  useEffect(() => {
    const data = use(promise);  // Error!
  }, []);
}
```

### Must Be During Render

```jsx
// ✓ Good: During render
function Component({ promise }) {
  const data = use(promise);
  return <div>{data}</div>;
}

// ✗ Bad: After render
function Component({ promise }) {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    const value = use(promise);  // Error!
    setData(value);
  }, []);
}
```

## Common Mistakes

### Mistake 1: Creating Promise in Render

```jsx
// Bad: New promise every render
function BadComponent({ userId }) {
  const user = use(fetchUser(userId));  // New promise each render!
  return <div>{user.name}</div>;
}

// Good: Create promise outside or in parent
function GoodParent({ userId }) {
  const userPromise = useMemo(
    () => fetchUser(userId),
    [userId]
  );
  
  return (
    <Suspense fallback={<Loading />}>
      <UserProfile promise={userPromise} />
    </Suspense>
  );
}

function UserProfile({ promise }) {
  const user = use(promise);
  return <div>{user.name}</div>;
}
```

### Mistake 2: Not Wrapping in Suspense

```jsx
// Bad: No Suspense boundary
function BadApp() {
  return <UserProfile promise={fetchUser()} />;  // Error!
}

// Good: Wrap in Suspense
function GoodApp() {
  return (
    <Suspense fallback={<Loading />}>
      <UserProfile promise={fetchUser()} />
    </Suspense>
  );
}
```

### Mistake 3: Using in Effects

```jsx
// Bad: Can't use in effects
function BadComponent({ promise }) {
  useEffect(() => {
    const data = use(promise);  // Error!
  }, [promise]);
}

// Good: use() in render, effect uses result
function GoodComponent({ promise }) {
  const data = use(promise);
  
  useEffect(() => {
    console.log('Data loaded:', data);
  }, [data]);
}
```

### Mistake 4: Missing Error Boundary

```jsx
// Bad: No error handling
function BadApp() {
  return (
    <Suspense fallback={<Loading />}>
      <UserProfile promise={fetchUser()} />
    </Suspense>
  );
  // If promise rejects, error bubbles up with no handler
}

// Good: Add Error Boundary
function GoodApp() {
  return (
    <ErrorBoundary fallback={<div>Error loading</div>}>
      <Suspense fallback={<Loading />}>
        <UserProfile promise={fetchUser()} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

## Comparison with Other Approaches

### use() vs useEffect

```jsx
// useEffect approach
function WithEffect({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    setLoading(true);
    fetchUser(userId)
      .then(setUser)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [userId]);
  
  if (loading) return <Loading />;
  if (error) return <Error />;
  return <div>{user.name}</div>;
}

// use() approach
function WithUse({ userPromise }) {
  const user = use(userPromise);
  return <div>{user.name}</div>;
}

// Parent handles loading/error
function Parent({ userId }) {
  const promise = useMemo(() => fetchUser(userId), [userId]);
  
  return (
    <ErrorBoundary fallback={<Error />}>
      <Suspense fallback={<Loading />}>
        <WithUse userPromise={promise} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

### use() vs React Query

```jsx
// React Query
function WithQuery({ userId }) {
  const { data, isLoading, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId)
  });
  
  if (isLoading) return <Loading />;
  if (error) return <Error />;
  return <div>{data.name}</div>;
}

// use() (simpler but less features)
function WithUse({ userPromise }) {
  const user = use(userPromise);
  return <div>{user.name}</div>;
}

// Note: React Query provides caching, refetching, etc.
// use() is lower-level, you implement those features yourself
```

## Browser Support

- React 19+
- All modern browsers
- Server Components (Next.js 15+)

## Resources

- [React 19 Beta Announcement](https://react.dev/blog/2024/04/25/react-19)
- [RFC: First class support for promises](https://github.com/reactjs/rfcs/pull/229)
- [Suspense for Data Fetching](https://react.dev/reference/react/Suspense)
