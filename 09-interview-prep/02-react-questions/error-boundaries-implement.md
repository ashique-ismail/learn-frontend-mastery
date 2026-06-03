# Error Boundaries — Implement One

## What Error Boundaries Catch

Error boundaries catch JavaScript errors during **rendering**, in **lifecycle methods**, and in **constructors** of child components.

**They do NOT catch:**
- Errors in event handlers (use try/catch there)
- Asynchronous errors (setTimeout, fetch, async functions)
- Server-side rendering errors
- Errors in the error boundary itself

---

## Class Component Requirement

Error boundaries must be class components (as of React 18 — no hook equivalent yet):

```jsx
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  // Called during render phase — update state to show fallback
  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  // Called after render — use for logging
  componentDidCatch(error, info) {
    console.error('Error:', error);
    console.error('Component stack:', info.componentStack);
    // logToSentry(error, info);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? <h2>Something went wrong.</h2>;
    }
    return this.props.children;
  }
}
```

Usage:
```jsx
<ErrorBoundary fallback={<ErrorPage />}>
  <UserDashboard />
</ErrorBoundary>
```

---

## Reusable ErrorBoundary with Reset

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, info) {
    this.props.onError?.(error, info);
  }

  // Allow parent to reset (e.g., on navigation)
  componentDidUpdate(prevProps) {
    if (this.state.hasError && prevProps.resetKey !== this.props.resetKey) {
      this.setState({ hasError: false, error: null });
    }
  }

  reset = () => this.setState({ hasError: false, error: null });

  render() {
    if (this.state.hasError) {
      const FallbackComponent = this.props.FallbackComponent;
      return FallbackComponent
        ? <FallbackComponent error={this.state.error} reset={this.reset} />
        : <div>Error: {this.state.error?.message}</div>;
    }
    return this.props.children;
  }
}
```

```jsx
function ErrorFallback({ error, reset }) {
  return (
    <div role="alert">
      <p>Something went wrong: {error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}

<ErrorBoundary
  FallbackComponent={ErrorFallback}
  onError={(err, info) => logToSentry(err, info)}
  resetKey={userId}  // resets boundary when user changes
>
  <UserProfile />
</ErrorBoundary>
```

---

## `react-error-boundary` Library

The community standard — wraps all of the above:

```jsx
import { ErrorBoundary, useErrorBoundary } from 'react-error-boundary';

// Throw from async code into the boundary
function AsyncComponent() {
  const { showBoundary } = useErrorBoundary();

  async function loadData() {
    try {
      await fetchData();
    } catch (err) {
      showBoundary(err); // manually trigger the boundary
    }
  }
}

<ErrorBoundary
  FallbackComponent={ErrorFallback}
  onReset={({ reason }) => console.log('Reset reason:', reason)}
  resetKeys={[userId]}
>
  <App />
</ErrorBoundary>
```

---

## Granularity Strategy

```jsx
// Page-level: catch catastrophic failures
<ErrorBoundary fallback={<ErrorPage />}>
  <App />
</ErrorBoundary>

// Feature-level: isolate widgets so one failure doesn't kill the page
<ErrorBoundary fallback={<p>Chart failed to load</p>}>
  <AnalyticsChart />
</ErrorBoundary>

<ErrorBoundary fallback={<p>Feed unavailable</p>}>
  <ActivityFeed />
</ErrorBoundary>
```

---

## Common Interview Questions

**Q: Why can't you catch async errors with error boundaries?**
Error boundaries intercept errors thrown during the React render cycle. `async` functions return promises — their rejections happen outside the render cycle, after React has already finished processing.

**Q: How do you catch async errors and show a boundary fallback?**
Use `react-error-boundary`'s `useErrorBoundary()` hook and call `showBoundary(error)` in a catch block. Or use `useState` to store the error and rethrow it in a `useEffect`.

**Q: What are the two lifecycle methods for error boundaries?**
`getDerivedStateFromError(error)` — static, pure, runs during render, returns new state to show fallback. `componentDidCatch(error, info)` — runs after commit, use for side effects like logging.

**Q: Can an error boundary catch errors thrown in itself?**
No. An error boundary cannot catch errors thrown in its own render method — that would cause an infinite loop. React unmounts the entire tree in that case.
