# Higher-Order Components (HOCs)

## Overview

A **Higher-Order Component (HOC)** is a pattern that involves a function that takes a component and returns a new component with additional props or behavior.

```jsx
const EnhancedComponent = higherOrderComponent(WrappedComponent);
```

**Key concept**: HOCs are **functions**, not components. They transform components by wrapping them.

## Basic Structure

```jsx
// HOC skeleton
function withEnhancement(WrappedComponent) {
  return function EnhancedComponent(props) {
    // Additional logic here
    const enhancedProps = { /* ... */ };
    
    return <WrappedComponent {...props} {...enhancedProps} />;
  };
}

// Usage
const EnhancedButton = withEnhancement(Button);
```

## Simple Examples

### Example 1: With Loading State

```jsx
function withLoading(WrappedComponent) {
  return function WithLoadingComponent({ isLoading, ...props }) {
    if (isLoading) {
      return <div>Loading...</div>;
    }
    
    return <WrappedComponent {...props} />;
  };
}

// Usage
function UserList({ users }) {
  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}

const UserListWithLoading = withLoading(UserList);

// In parent component
function App() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  
  return (
    <UserListWithLoading
      users={users}
      isLoading={loading}
    />
  );
}
```

### Example 2: With Authentication

```jsx
function withAuth(WrappedComponent) {
  return function WithAuthComponent(props) {
    const { user, loading } = useAuth();
    
    if (loading) {
      return <Spinner />;
    }
    
    if (!user) {
      return <Navigate to="/login" />;
    }
    
    return <WrappedComponent {...props} user={user} />;
  };
}

// Usage
function Dashboard({ user }) {
  return (
    <div>
      <h1>Welcome, {user.name}</h1>
      {/* Dashboard content */}
    </div>
  );
}

const ProtectedDashboard = withAuth(Dashboard);
```

### Example 3: With Logger

```jsx
function withLogger(WrappedComponent) {
  return function WithLoggerComponent(props) {
    useEffect(() => {
      console.log('Component mounted:', WrappedComponent.name);
      console.log('Props:', props);
      
      return () => {
        console.log('Component unmounted:', WrappedComponent.name);
      };
    });
    
    return <WrappedComponent {...props} />;
  };
}

// Usage
const ButtonWithLogger = withLogger(Button);
```

## Practical Examples

### Example 4: With Data Fetching

```jsx
function withData(url) {
  return function (WrappedComponent) {
    return function WithDataComponent(props) {
      const [data, setData] = useState(null);
      const [loading, setLoading] = useState(true);
      const [error, setError] = useState(null);
      
      useEffect(() => {
        fetch(url)
          .then(res => res.json())
          .then(data => {
            setData(data);
            setLoading(false);
          })
          .catch(error => {
            setError(error);
            setLoading(false);
          });
      }, []);
      
      return (
        <WrappedComponent
          {...props}
          data={data}
          loading={loading}
          error={error}
        />
      );
    };
  };
}

// Usage
function UserProfile({ data, loading, error }) {
  if (loading) return <Spinner />;
  if (error) return <Error message={error.message} />;
  
  return (
    <div>
      <h1>{data.name}</h1>
      <p>{data.email}</p>
    </div>
  );
}

const UserProfileWithData = withData('/api/user')(UserProfile);
```

### Example 5: With Click Outside

```jsx
function withClickOutside(WrappedComponent) {
  return function WithClickOutsideComponent(props) {
    const ref = useRef(null);
    
    useEffect(() => {
      const handleClickOutside = (event) => {
        if (ref.current && !ref.current.contains(event.target)) {
          props.onClickOutside?.();
        }
      };
      
      document.addEventListener('mousedown', handleClickOutside);
      return () => {
        document.removeEventListener('mousedown', handleClickOutside);
      };
    }, [props.onClickOutside]);
    
    return (
      <div ref={ref}>
        <WrappedComponent {...props} />
      </div>
    );
  };
}

// Usage
function Dropdown({ items, onClickOutside }) {
  const [isOpen, setIsOpen] = useState(false);
  
  return (
    <div>
      <button onClick={() => setIsOpen(!isOpen)}>Toggle</button>
      {isOpen && (
        <ul>
          {items.map(item => (
            <li key={item.id}>{item.label}</li>
          ))}
        </ul>
      )}
    </div>
  );
}

const DropdownWithClickOutside = withClickOutside(Dropdown);

function App() {
  return (
    <DropdownWithClickOutside
      items={[{ id: 1, label: 'Item 1' }]}
      onClickOutside={() => console.log('Clicked outside')}
    />
  );
}
```

### Example 6: With Error Boundary

```jsx
function withErrorBoundary(WrappedComponent) {
  return class WithErrorBoundary extends React.Component {
    state = { hasError: false, error: null };
    
    static getDerivedStateFromError(error) {
      return { hasError: true, error };
    }
    
    componentDidCatch(error, errorInfo) {
      console.error('Error caught by boundary:', error, errorInfo);
    }
    
    render() {
      if (this.state.hasError) {
        return (
          <div>
            <h1>Something went wrong</h1>
            <details>
              <summary>Error details</summary>
              <pre>{this.state.error.toString()}</pre>
            </details>
          </div>
        );
      }
      
      return <WrappedComponent {...this.props} />;
    }
  };
}

// Usage
const SafeUserProfile = withErrorBoundary(UserProfile);
```

### Example 7: With Performance Tracking

```jsx
function withPerformanceTracking(WrappedComponent) {
  return function WithPerformanceComponent(props) {
    const renderCount = useRef(0);
    
    useEffect(() => {
      renderCount.current += 1;
    });
    
    useEffect(() => {
      const startTime = performance.now();
      
      return () => {
        const endTime = performance.now();
        console.log(`${WrappedComponent.name} was mounted for ${endTime - startTime}ms`);
      };
    }, []);
    
    return (
      <>
        {process.env.NODE_ENV === 'development' && (
          <div style={{ fontSize: '10px', color: 'gray' }}>
            Renders: {renderCount.current}
          </div>
        )}
        <WrappedComponent {...props} />
      </>
    );
  };
}
```

### Example 8: With Responsive Behavior

```jsx
function withResponsive(WrappedComponent) {
  return function WithResponsiveComponent(props) {
    const [windowSize, setWindowSize] = useState({
      width: window.innerWidth,
      height: window.innerHeight
    });
    
    useEffect(() => {
      const handleResize = () => {
        setWindowSize({
          width: window.innerWidth,
          height: window.innerHeight
        });
      };
      
      window.addEventListener('resize', handleResize);
      return () => window.removeEventListener('resize', handleResize);
    }, []);
    
    const isMobile = windowSize.width < 768;
    const isTablet = windowSize.width >= 768 && windowSize.width < 1024;
    const isDesktop = windowSize.width >= 1024;
    
    return (
      <WrappedComponent
        {...props}
        windowSize={windowSize}
        isMobile={isMobile}
        isTablet={isTablet}
        isDesktop={isDesktop}
      />
    );
  };
}

// Usage
function Layout({ isMobile, isDesktop, children }) {
  return (
    <div className={isMobile ? 'mobile-layout' : 'desktop-layout'}>
      {children}
    </div>
  );
}

const ResponsiveLayout = withResponsive(Layout);
```

### Example 9: With Theme

```jsx
const ThemeContext = createContext(null);

function withTheme(WrappedComponent) {
  return function WithThemeComponent(props) {
    const theme = useContext(ThemeContext);
    
    if (!theme) {
      console.warn('withTheme: No theme provided');
    }
    
    return <WrappedComponent {...props} theme={theme} />;
  };
}

// Usage
function Button({ theme, children }) {
  return (
    <button
      style={{
        backgroundColor: theme.primary,
        color: theme.text
      }}
    >
      {children}
    </button>
  );
}

const ThemedButton = withTheme(Button);

function App() {
  const theme = { primary: 'blue', text: 'white' };
  
  return (
    <ThemeContext.Provider value={theme}>
      <ThemedButton>Click me</ThemedButton>
    </ThemeContext.Provider>
  );
}
```

## Composing Multiple HOCs

### Sequential Composition

```jsx
const EnhancedComponent = 
  withAuth(
    withTheme(
      withResponsive(
        Component
      )
    )
  );
```

### Using compose Utility

```jsx
function compose(...fns) {
  return (Component) =>
    fns.reduceRight((acc, fn) => fn(acc), Component);
}

const enhance = compose(
  withAuth,
  withTheme,
  withResponsive,
  withLogger
);

const EnhancedComponent = enhance(Component);
```

### Recompose-style (for reference)

```jsx
import { compose } from 'redux'; // or create your own

const enhance = compose(
  withAuth,
  withData('/api/user'),
  withLoading,
  withErrorBoundary
);

const UserProfile = enhance(({ data, user }) => (
  <div>
    <h1>{user.name}</h1>
    <p>{data.bio}</p>
  </div>
));
```

## Best Practices

### 1. Don't Mutate the Component

```jsx
// ✗ Bad: Mutating component
function bad(WrappedComponent) {
  WrappedComponent.prototype.componentWillReceiveProps = function() {
    // Mutation!
  };
  return WrappedComponent;
}

// ✓ Good: Return new component
function good(WrappedComponent) {
  return class extends React.Component {
    componentWillReceiveProps() {
      // No mutation
    }
    
    render() {
      return <WrappedComponent {...this.props} />;
    }
  };
}
```

### 2. Pass Unrelated Props

```jsx
// ✓ Good: Pass through all props
function withExtra(WrappedComponent) {
  return function WithExtraComponent(props) {
    const extra = { /* ... */ };
    
    return (
      <WrappedComponent
        {...props}  // Pass all props through
        extra={extra}
      />
    );
  };
}
```

### 3. Maximize Composability

```jsx
// Instead of
function withDataAndAuth(WrappedComponent) {
  // Both data fetching and auth in one HOC
}

// Better: Separate concerns
function withData(WrappedComponent) { /* ... */ }
function withAuth(WrappedComponent) { /* ... */ }

const Component = withAuth(withData(MyComponent));
```

### 4. Use Display Name for Debugging

```jsx
function withExtra(WrappedComponent) {
  function WithExtraComponent(props) {
    // ...
  }
  
  WithExtraComponent.displayName = 
    `WithExtra(${getDisplayName(WrappedComponent)})`;
  
  return WithExtraComponent;
}

function getDisplayName(WrappedComponent) {
  return WrappedComponent.displayName || 
         WrappedComponent.name || 
         'Component';
}
```

### 5. Copy Static Methods

```jsx
import hoistNonReactStatics from 'hoist-non-react-statics';

function withExtra(WrappedComponent) {
  function WithExtraComponent(props) {
    return <WrappedComponent {...props} />;
  }
  
  // Copy static methods
  hoistNonReactStatics(WithExtraComponent, WrappedComponent);
  
  return WithExtraComponent;
}
```

### 6. Wrap Display Name

```jsx
function withSubscription(WrappedComponent) {
  class WithSubscription extends React.Component {
    // ...
  }
  
  WithSubscription.displayName = 
    `WithSubscription(${getDisplayName(WrappedComponent)})`;
  
  return WithSubscription;
}
```

## TypeScript

```tsx
import { ComponentType } from 'react';

// HOC type definition
function withData<P extends object>(
  url: string
) {
  return function<T extends P>(
    WrappedComponent: ComponentType<T>
  ): ComponentType<Omit<T, 'data' | 'loading'>> {
    return function WithDataComponent(props: Omit<T, 'data' | 'loading'>) {
      const [data, setData] = useState(null);
      const [loading, setLoading] = useState(true);
      
      useEffect(() => {
        fetch(url)
          .then(res => res.json())
          .then(setData)
          .finally(() => setLoading(false));
      }, []);
      
      return (
        <WrappedComponent
          {...(props as T)}
          data={data}
          loading={loading}
        />
      );
    };
  };
}

// Usage with type safety
interface UserProfileProps {
  data: User;
  loading: boolean;
  additionalProp: string;
}

function UserProfile({ data, loading, additionalProp }: UserProfileProps) {
  if (loading) return <div>Loading...</div>;
  return <div>{data.name} - {additionalProp}</div>;
}

const UserProfileWithData = withData<UserProfileProps>('/api/user')(UserProfile);

// Now UserProfileWithData only requires 'additionalProp'
<UserProfileWithData additionalProp="test" />
```

## Common Pitfalls

### 1. Don't Use HOCs Inside render

```jsx
// ✗ Bad: Creates new component each render
function Parent() {
  const EnhancedComponent = withExtra(MyComponent);
  return <EnhancedComponent />;
}

// ✓ Good: Create outside
const EnhancedComponent = withExtra(MyComponent);

function Parent() {
  return <EnhancedComponent />;
}
```

### 2. Refs Aren't Passed Through

```jsx
// Problem: ref points to wrapper, not wrapped component

// Solution: Use React.forwardRef
function withExtra(WrappedComponent) {
  function WithExtraComponent(props, ref) {
    return <WrappedComponent {...props} ref={ref} />;
  }
  
  return React.forwardRef(WithExtraComponent);
}
```

### 3. Display Name Not Set

```jsx
// Without display name:
// <Unknown /> in DevTools

// With display name:
// <WithAuth(UserProfile)> in DevTools
```

## Modern Alternatives

### Custom Hooks (Preferred)

```jsx
// HOC
const UserWithData = withData('/api/user')(User);

// Custom Hook (simpler)
function User() {
  const { data, loading } = useData('/api/user');
  
  if (loading) return <Spinner />;
  return <div>{data.name}</div>;
}

function useData(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    fetch(url)
      .then(res => res.json())
      .then(data => {
        setData(data);
        setLoading(false);
      });
  }, [url]);
  
  return { data, loading };
}
```

### Why Hooks Are Better

1. **Simpler**: No wrapper components
2. **Clearer**: Logic is in component
3. **Better TypeScript**: Easier to type
4. **No Name Collision**: Multiple hooks don't clash
5. **Better DevTools**: Cleaner component tree

## When to Use HOCs

### Still Useful For:

1. **React Class Components**: Hooks don't work in classes
2. **Cross-Cutting Concerns**: Like error boundaries
3. **Third-Party Libraries**: Some libraries use HOC pattern
4. **Legacy Codebases**: Already using HOCs extensively

### Prefer Hooks For:

1. **New code**: Hooks are the modern approach
2. **Logic reuse**: Hooks are simpler
3. **Component composition**: Hooks avoid wrapper hell

## Testing

```jsx
import { render, screen } from '@testing-library/react';

// Test the HOC
describe('withAuth', () => {
  it('redirects when not authenticated', () => {
    const Component = withAuth(() => <div>Protected</div>);
    
    render(<Component />, {
      wrapper: ({ children }) => (
        <AuthProvider user={null}>
          {children}
        </AuthProvider>
      )
    });
    
    expect(screen.queryByText('Protected')).not.toBeInTheDocument();
  });
  
  it('renders when authenticated', () => {
    const Component = withAuth(() => <div>Protected</div>);
    
    render(<Component />, {
      wrapper: ({ children }) => (
        <AuthProvider user={{ id: 1, name: 'John' }}>
          {children}
        </AuthProvider>
      )
    });
    
    expect(screen.getByText('Protected')).toBeInTheDocument();
  });
});
```

## Resources

- [React Docs: Higher-Order Components](https://legacy.reactjs.org/docs/higher-order-components.html)
- [Recompose](https://github.com/acdlite/recompose) (archived, but good reference)
- [Why React Hooks](https://www.robinwieruch.de/react-hooks-migration/)
- [HOCs vs Hooks](https://kentcdodds.com/blog/react-hooks-whats-going-to-happen-to-higher-order-components)
