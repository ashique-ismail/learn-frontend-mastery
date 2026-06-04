# Error Boundaries

## Overview

Error Boundaries are React components that catch JavaScript errors anywhere in their child component tree, log those errors, and display a fallback UI instead of crashing the entire application. They act as a safety net, preventing a single component error from breaking the entire app.

Error Boundaries are essential for building robust React applications that gracefully handle unexpected failures and provide good user experiences even when things go wrong.

## Core Concepts

### What Are Error Boundaries?

Error Boundaries:

1. **Catch errors during rendering** in child components
2. **Catch errors in lifecycle methods** and constructors
3. **Display fallback UI** instead of crashing
4. **Log error information** for debugging
5. **Don't catch errors** in event handlers, async code, SSR, or the boundary itself

### What Error Boundaries DON'T Catch

```javascript
// NOT caught by error boundaries:
// 1. Event handlers
<button onClick={() => { throw new Error('Boom!'); }} />

// 2. Async code
setTimeout(() => { throw new Error('Boom!'); }, 1000);

// 3. Server-side rendering
// 4. Errors in the error boundary itself
```

## Basic Implementation

### Class Component Error Boundary

```javascript
import React from 'react';

class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error) {
    // Update state so next render shows fallback UI
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    // Log error to error reporting service
    console.error('Error caught by boundary:', error, errorInfo);
    
    // You can also log to services like Sentry
    // logErrorToService(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return (
        <div>
          <h2>Something went wrong.</h2>
          <details style={{ whiteSpace: 'pre-wrap' }}>
            {this.state.error && this.state.error.toString()}
          </details>
        </div>
      );
    }

    return this.props.children;
  }
}

// Usage
function App() {
  return (
    <ErrorBoundary>
      <MyComponent />
    </ErrorBoundary>
  );
}
```

### Error Boundary with Reset

```javascript
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null, errorInfo: null };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    this.setState({ errorInfo });
    console.error('Error:', error, errorInfo);
  }

  resetError = () => {
    this.setState({ hasError: false, error: null, errorInfo: null });
  };

  render() {
    if (this.state.hasError) {
      return (
        <div className="error-boundary">
          <h2>Oops! Something went wrong</h2>
          <p>{this.state.error?.message}</p>
          {process.env.NODE_ENV === 'development' && (
            <details>
              <summary>Error Details</summary>
              <pre>{this.state.errorInfo?.componentStack}</pre>
            </details>
          )}
          <button onClick={this.resetError}>Try Again</button>
        </div>
      );
    }

    return this.props.children;
  }
}
```

### Custom Fallback Component

```javascript
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    this.props.onError?.(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      // Use custom fallback if provided
      if (this.props.fallback) {
        return this.props.fallback;
      }

      // Or fallback render function
      if (this.props.FallbackComponent) {
        return (
          <this.props.FallbackComponent
            error={this.state.error}
            resetError={() => this.setState({ hasError: false })}
          />
        );
      }

      // Default fallback
      return <h2>Something went wrong</h2>;
    }

    return this.props.children;
  }
}

// Usage with custom fallback
function ErrorFallback({ error, resetError }) {
  return (
    <div role="alert">
      <h2>Error Occurred</h2>
      <p>{error.message}</p>
      <button onClick={resetError}>Retry</button>
    </div>
  );
}

function App() {
  return (
    <ErrorBoundary FallbackComponent={ErrorFallback}>
      <MyComponent />
    </ErrorBoundary>
  );
}
```

## Advanced Patterns

### Multiple Error Boundaries

```javascript
function App() {
  return (
    <div>
      {/* Top-level boundary for critical errors */}
      <ErrorBoundary
        fallback={<CriticalErrorPage />}
        onError={(error) => logToService(error)}
      >
        <Header />

        {/* Isolated boundary for main content */}
        <ErrorBoundary
          fallback={<ContentErrorFallback />}
          onError={(error) => console.error('Content error:', error)}
        >
          <MainContent />
        </ErrorBoundary>

        {/* Isolated boundary for sidebar */}
        <ErrorBoundary
          fallback={<div>Sidebar unavailable</div>}
        >
          <Sidebar />
        </ErrorBoundary>

        <Footer />
      </ErrorBoundary>
    </div>
  );
}
```

### Error Boundary with Retry Logic

```javascript
class RetryErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      hasError: false,
      error: null,
      retryCount: 0,
    };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    console.error('Error:', error, errorInfo);
  }

  handleRetry = () => {
    this.setState((state) => ({
      hasError: false,
      error: null,
      retryCount: state.retryCount + 1,
    }));
  };

  render() {
    const { maxRetries = 3 } = this.props;

    if (this.state.hasError) {
      const canRetry = this.state.retryCount < maxRetries;

      return (
        <div>
          <h2>Error Loading Component</h2>
          <p>{this.state.error?.message}</p>
          {canRetry ? (
            <button onClick={this.handleRetry}>
              Retry ({this.state.retryCount + 1}/{maxRetries})
            </button>
          ) : (
            <p>Maximum retries exceeded. Please refresh the page.</p>
          )}
        </div>
      );
    }

    // Pass retry count as key to force remount
    return (
      <React.Fragment key={this.state.retryCount}>
        {this.props.children}
      </React.Fragment>
    );
  }
}
```

### Error Boundary with Context

```javascript
const ErrorContext = React.createContext();

class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      hasError: false,
      error: null,
      errors: [], // Track all errors
    };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    this.setState((state) => ({
      errors: [...state.errors, { error, errorInfo, timestamp: Date.now() }],
    }));
  }

  clearError = () => {
    this.setState({ hasError: false, error: null });
  };

  render() {
    const contextValue = {
      hasError: this.state.hasError,
      error: this.state.error,
      errors: this.state.errors,
      clearError: this.clearError,
    };

    if (this.state.hasError) {
      return (
        <ErrorContext.Provider value={contextValue}>
          {this.props.fallback || <DefaultErrorFallback />}
        </ErrorContext.Provider>
      );
    }

    return (
      <ErrorContext.Provider value={contextValue}>
        {this.props.children}
      </ErrorContext.Provider>
    );
  }
}

// Hook to access error context
function useErrorHandler() {
  const context = React.useContext(ErrorContext);
  if (!context) {
    throw new Error('useErrorHandler must be used within ErrorBoundary');
  }
  return context;
}
```

## Real-World Patterns

### Async Error Handling

```javascript
// Error boundaries don't catch async errors, so we need a workaround
function useAsyncError() {
  const [, setError] = useState();

  return useCallback((error) => {
    setError(() => {
      throw error; // This will be caught by error boundary
    });
  }, []);
}

function AsyncComponent() {
  const throwError = useAsyncError();

  useEffect(() => {
    fetch('/api/data')
      .then((res) => res.json())
      .catch((error) => {
        throwError(error); // Now error boundary catches it
      });
  }, [throwError]);

  return <div>Loading...</div>;
}

function App() {
  return (
    <ErrorBoundary>
      <AsyncComponent />
    </ErrorBoundary>
  );
}
```

### Event Handler Error Handling

```javascript
// Error boundaries don't catch event handler errors
function useErrorBoundary() {
  const [, setError] = useState();

  const showError = useCallback((error) => {
    setError(() => {
      throw error;
    });
  }, []);

  return showError;
}

function MyComponent() {
  const showError = useErrorBoundary();

  const handleClick = () => {
    try {
      // Some code that might throw
      riskyOperation();
    } catch (error) {
      showError(error); // Throw to error boundary
    }
  };

  return <button onClick={handleClick}>Click me</button>;
}
```

### Error Boundary with React Query

```javascript
import { QueryErrorResetBoundary } from '@tanstack/react-query';

function App() {
  return (
    <QueryErrorResetBoundary>
      {({ reset }) => (
        <ErrorBoundary
          onReset={reset}
          FallbackComponent={({ error, resetError }) => (
            <div>
              <h2>Query Error</h2>
              <p>{error.message}</p>
              <button
                onClick={() => {
                  reset(); // Reset React Query
                  resetError(); // Reset boundary
                }}
              >
                Try Again
              </button>
            </div>
          )}
        >
          <MyQueryComponent />
        </ErrorBoundary>
      )}
    </QueryErrorResetBoundary>
  );
}
```

### Route-Level Error Boundaries

```javascript
import { Routes, Route } from 'react-router-dom';

function App() {
  return (
    <Routes>
      <Route
        path="/"
        element={
          <ErrorBoundary fallback={<HomeErrorPage />}>
            <Home />
          </ErrorBoundary>
        }
      />
      <Route
        path="/profile"
        element={
          <ErrorBoundary fallback={<ProfileErrorPage />}>
            <Profile />
          </ErrorBoundary>
        }
      />
      <Route
        path="/settings"
        element={
          <ErrorBoundary fallback={<SettingsErrorPage />}>
            <Settings />
          </ErrorBoundary>
        }
      />
    </Routes>
  );
}
```

### Error Boundary with Logging Service

```javascript
// Integrate with Sentry, LogRocket, etc.
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null, errorInfo: null };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    this.setState({ errorInfo });

    // Log to external service
    if (window.Sentry) {
      window.Sentry.withScope((scope) => {
        scope.setContext('errorBoundary', {
          componentStack: errorInfo.componentStack,
        });
        window.Sentry.captureException(error);
      });
    }

    // Log to custom analytics
    if (window.analytics) {
      window.analytics.track('Error Occurred', {
        error: error.toString(),
        componentStack: errorInfo.componentStack,
        url: window.location.href,
      });
    }

    // Call custom error handler
    this.props.onError?.(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || <DefaultErrorFallback />;
    }

    return this.props.children;
  }
}
```

## Best Practices

### 1. Place Boundaries Strategically

```javascript
// Good: Multiple granular boundaries
function App() {
  return (
    <div>
      <ErrorBoundary fallback={<NavError />}>
        <Navigation />
      </ErrorBoundary>

      <ErrorBoundary fallback={<MainError />}>
        <MainContent />
      </ErrorBoundary>

      <ErrorBoundary fallback={<SidebarError />}>
        <Sidebar />
      </ErrorBoundary>
    </div>
  );
}

// Bad: Single boundary for everything
function App() {
  return (
    <ErrorBoundary>
      <Navigation />
      <MainContent />
      <Sidebar />
    </ErrorBoundary>
  );
}
```

### 2. Provide Meaningful Error Messages

```javascript
// Good: User-friendly error with actions
function ErrorFallback({ error, resetError }) {
  return (
    <div>
      <h2>We're sorry, something went wrong</h2>
      <p>The page couldn't be loaded. Please try again.</p>
      <button onClick={resetError}>Reload</button>
      <button onClick={() => window.location.href = '/'}>
        Go Home
      </button>
    </div>
  );
}

// Bad: Technical error message
function ErrorFallback({ error }) {
  return <div>Error: {error.stack}</div>;
}
```

### 3. Log Errors for Debugging

```javascript
componentDidCatch(error, errorInfo) {
  // Always log in development
  if (process.env.NODE_ENV === 'development') {
    console.error('Error:', error);
    console.error('Error Info:', errorInfo);
  }

  // Log to service in production
  if (process.env.NODE_ENV === 'production') {
    logErrorToService(error, errorInfo);
  }
}
```

### 4. Handle Async Errors

```javascript
// Create helper hook for async errors
function useAsyncError() {
  const [, setError] = useState();
  return useCallback((e) => setError(() => { throw e; }), []);
}

// Use in components
function MyComponent() {
  const throwError = useAsyncError();

  useEffect(() => {
    asyncOperation().catch(throwError);
  }, [throwError]);
}
```

## Common Mistakes

### 1. Not Using Error Boundaries

```javascript
// Bad: No error boundary
function App() {
  return <MyComponent />; // Errors crash entire app
}

// Good: Error boundary protects app
function App() {
  return (
    <ErrorBoundary>
      <MyComponent />
    </ErrorBoundary>
  );
}
```

### 2. Single Top-Level Boundary Only

```javascript
// Bad: One boundary for everything
<ErrorBoundary>
  <Header />
  <Content />
  <Footer />
</ErrorBoundary>

// Good: Granular boundaries
<>
  <ErrorBoundary fallback={<HeaderError />}>
    <Header />
  </ErrorBoundary>
  <ErrorBoundary fallback={<ContentError />}>
    <Content />
  </ErrorBoundary>
  <Footer /> {/* Non-critical, no boundary needed */}
</>
```

### 3. Not Resetting Error State

```javascript
// Bad: No way to recover
class ErrorBoundary extends React.Component {
  // ... no reset method
  render() {
    if (this.state.hasError) {
      return <div>Error!</div>; // User is stuck
    }
  }
}

// Good: Provide reset mechanism
class ErrorBoundary extends React.Component {
  resetError = () => {
    this.setState({ hasError: false });
  };

  render() {
    if (this.state.hasError) {
      return (
        <div>
          <p>Error!</p>
          <button onClick={this.resetError}>Try Again</button>
        </div>
      );
    }
  }
}
```

## Testing Error Boundaries

```javascript
import { render, screen } from '@testing-library/react';

// Suppress error boundary console errors in tests
beforeAll(() => {
  jest.spyOn(console, 'error').mockImplementation(() => {});
});

afterAll(() => {
  console.error.mockRestore();
});

const ThrowError = ({ shouldThrow }) => {
  if (shouldThrow) {
    throw new Error('Test error');
  }
  return <div>No error</div>;
};

test('catches errors and displays fallback', () => {
  render(
    <ErrorBoundary fallback={<div>Error occurred</div>}>
      <ThrowError shouldThrow={true} />
    </ErrorBoundary>
  );

  expect(screen.getByText('Error occurred')).toBeInTheDocument();
});

test('renders children when no error', () => {
  render(
    <ErrorBoundary fallback={<div>Error occurred</div>}>
      <ThrowError shouldThrow={false} />
    </ErrorBoundary>
  );

  expect(screen.getByText('No error')).toBeInTheDocument();
  expect(screen.queryByText('Error occurred')).not.toBeInTheDocument();
});

test('calls onError callback', () => {
  const onError = jest.fn();

  render(
    <ErrorBoundary onError={onError}>
      <ThrowError shouldThrow={true} />
    </ErrorBoundary>
  );

  expect(onError).toHaveBeenCalled();
});

test('reset error works', () => {
  const { rerender } = render(
    <ErrorBoundary
      FallbackComponent={({ resetError }) => (
        <button onClick={resetError}>Reset</button>
      )}
    >
      <ThrowError shouldThrow={true} />
    </ErrorBoundary>
  );

  expect(screen.getByText('Reset')).toBeInTheDocument();

  fireEvent.click(screen.getByText('Reset'));

  rerender(
    <ErrorBoundary>
      <ThrowError shouldThrow={false} />
    </ErrorBoundary>
  );

  expect(screen.getByText('No error')).toBeInTheDocument();
});
```

## React 19 and Future

React 19 is exploring function component error boundaries:

```javascript
// Proposed API (not yet available)
function ErrorBoundary({ children }) {
  const [error, setError] = useError();

  if (error) {
    return <div>Error: {error.message}</div>;
  }

  return children;
}
```

For now, class components are required for error boundaries.

## Key Takeaways

1. **Always Use Error Boundaries**: Prevent errors from crashing entire app
2. **Multiple Boundaries**: Use granular boundaries to isolate failures
3. **Meaningful Fallbacks**: Show user-friendly error messages
4. **Reset Mechanism**: Provide way to recover from errors
5. **Log Errors**: Integrate with logging services for debugging
6. **Handle Async**: Use helpers to catch async/event errors
7. **Test Error Cases**: Write tests for error scenarios
8. **Strategic Placement**: Put boundaries at logical UI boundaries

## Additional Resources

- [React Error Boundaries](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary)
- [Error Boundaries Legacy Docs](https://legacy.reactjs.org/docs/error-boundaries.html)
- [react-error-boundary library](https://github.com/bvaughn/react-error-boundary)
- [Sentry React Integration](https://docs.sentry.io/platforms/javascript/guides/react/)
