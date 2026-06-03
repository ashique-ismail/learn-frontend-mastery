# Suspense — How It Works Internally

## What Suspense Does

Suspense lets you declaratively specify a loading state while a component tree is "not ready" — whether waiting for a lazy-loaded component or async data.

```jsx
<Suspense fallback={<Spinner />}>
  <UserProfile />  {/* may suspend */}
</Suspense>
```

---

## The Mechanism: Throwing a Promise

React Suspense works by **catching thrown promises**. When a component isn't ready, it throws a Promise (a thenable). React catches it, shows the nearest `<Suspense>` fallback, and re-renders the component when the promise resolves.

```js
// Simplified data-fetching with suspense
function createResource(promise) {
  let status = 'pending';
  let result;
  const suspender = promise.then(
    data => { status = 'success'; result = data; },
    err  => { status = 'error';   result = err; }
  );

  return {
    read() {
      if (status === 'pending') throw suspender;   // ← React catches this
      if (status === 'error')   throw result;
      return result;  // success — return data synchronously
    }
  };
}

const userResource = createResource(fetchUser(1));

function UserProfile() {
  const user = userResource.read(); // throws if pending, returns if ready
  return <div>{user.name}</div>;
}
```

---

## React's Internal Flow

1. React renders `<UserProfile />`
2. `userResource.read()` throws a Promise (the "suspender")
3. React unwinds the render, finds the nearest `<Suspense>` boundary
4. React displays the `fallback`
5. React attaches a `.then()` to the thrown promise
6. When the promise resolves, React re-renders the suspended tree from the boundary

This is called the **"throw a promise"** or **"suspense protocol"**. Libraries must implement this protocol to integrate with Suspense.

---

## `React.lazy` — Code Splitting Integration

```jsx
const LazyChart = React.lazy(() => import('./Chart'));

function Dashboard() {
  return (
    <Suspense fallback={<div>Loading chart...</div>}>
      <LazyChart data={data} />
    </Suspense>
  );
}
```

Internally: `React.lazy()` creates a component that throws a promise on first render (the dynamic import). When the import resolves, React re-renders and the component shows.

---

## Data Fetching Libraries Integration

Libraries like TanStack Query, Relay, and SWR implement the suspense protocol:

```jsx
// TanStack Query with suspense
const { data } = useSuspenseQuery({
  queryKey: ['user', id],
  queryFn: () => fetchUser(id),
});
// No loading check needed — component only renders when data is ready
```

---

## Concurrent Suspense vs Legacy

| | Legacy (React 17) | Concurrent (React 18+) |
|---|---|---|
| Render approach | Synchronous | Interruptible |
| Suspense during transition | Tears / shows stale | Shows stale until ready (startTransition) |
| `useTransition` | Not available | Prevents fallback flash for fast updates |
| Streaming SSR | Not supported | Fully supported |

```jsx
// Concurrent: avoid fallback flash on transitions
const [isPending, startTransition] = useTransition();

startTransition(() => {
  setUserId(newId); // React shows old content while new data loads
});
```

---

## Error Boundaries with Suspense

Thrown errors (not promises) are caught by Error Boundaries, which must be class components or use `react-error-boundary`:

```jsx
<ErrorBoundary fallback={<ErrorMessage />}>
  <Suspense fallback={<Spinner />}>
    <DataDependentComponent />
  </Suspense>
</ErrorBoundary>
```

---

## Common Interview Questions

**Q: What does Suspense actually catch?**
Any value thrown during render that has a `.then` method (a thenable/Promise). React checks `typeof thrown.then === 'function'` to distinguish Suspense-able throws from errors.

**Q: Can you use Suspense with useEffect-based data fetching?**
No. `useEffect` runs after render, not during. Suspense requires throwing during render. Libraries that use `useEffect` for fetching (classic pattern) cannot integrate with Suspense without rearchitecting to the throw-promise protocol.

**Q: What's the difference between Suspense for code-splitting and Suspense for data?**
Functionally identical mechanism — both throw a promise. The difference is what the promise resolves to: a module (code splitting) vs data.

**Q: Can Suspense boundaries be nested?**
Yes. React finds the nearest enclosing Suspense boundary. Inner boundaries can handle component-level loading while outer boundaries handle page-level loading.
