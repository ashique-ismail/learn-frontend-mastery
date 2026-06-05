# Conditional Rendering Patterns

## The Idea

**In plain English:** Conditional rendering means your webpage can show different things depending on a situation — like showing a "Log In" button when you are not signed in, or showing your profile when you are. It is the code equivalent of "if this is true, show that; otherwise, show something else."

**Real-world analogy:** Think of a bouncer at a club checking a list at the door. If your name is on the VIP list, you get taken to the VIP lounge. If you are just a regular guest, you go to the main floor. If you are not on any list at all, you are turned away.

- The bouncer checking the list = the condition being evaluated in code
- The VIP lounge shown to VIP guests = the component rendered when the condition is true
- The main floor shown to regular guests = the component rendered when the condition is false

---

## Overview

Conditional rendering in React allows you to render different UI elements based on certain conditions. Since JSX is JavaScript, you can use all JavaScript conditional operators and control flow. However, some patterns are more idiomatic and maintainable than others.

## Basic Conditional Rendering

### if Statement

Use if statements outside JSX for complex conditions:

```jsx
function Greeting({ user }) {
  if (!user) {
    return <div>Please sign in</div>;
  }
  
  if (user.isAdmin) {
    return <div>Welcome, Admin {user.name}!</div>;
  }
  
  return <div>Welcome, {user.name}!</div>;
}
```

### Ternary Operator

The most common pattern for inline conditionals:

```jsx
function UserStatus({ isLoggedIn }) {
  return (
    <div>
      {isLoggedIn ? (
        <Dashboard />
      ) : (
        <LoginForm />
      )}
    </div>
  );
}

// Inline with text
function Greeting({ name }) {
  return <h1>{name ? `Hello, ${name}` : 'Hello, stranger'}</h1>;
}
```

### Logical AND (&&)

Render something only if condition is true:

```jsx
function Notifications({ count }) {
  return (
    <div>
      <span>Notifications</span>
      {count > 0 && <span className="badge">{count}</span>}
    </div>
  );
}

// Multiple conditions
function UserProfile({ user, showEmail, showPhone }) {
  return (
    <div>
      <h2>{user.name}</h2>
      {showEmail && <p>Email: {user.email}</p>}
      {showPhone && <p>Phone: {user.phone}</p>}
    </div>
  );
}
```

### Logical OR (||)

Provide fallback values:

```jsx
function UserName({ user }) {
  return <div>{user.name || 'Anonymous'}</div>;
}

// With optional chaining
function UserProfile({ user }) {
  return (
    <div>
      <h2>{user?.name || 'Unknown User'}</h2>
      <p>{user?.email || 'No email provided'}</p>
    </div>
  );
}
```

### Nullish Coalescing (??)

More precise than ||, only falls back on null/undefined:

```jsx
function Score({ score }) {
  // 0 is falsy but valid, ?? handles this correctly
  return <div>Score: {score ?? 'Not available'}</div>;
}

// Comparison
function Component({ value }) {
  // Wrong: 0, false, '' trigger fallback
  return <div>{value || 'default'}</div>;
  
  // Correct: only null/undefined trigger fallback
  return <div>{value ?? 'default'}</div>;
}
```

## Common Rendering Patterns

### 1. Early Return

Exit early for special cases:

```jsx
function UserList({ users, isLoading, error }) {
  if (error) {
    return <ErrorMessage error={error} />;
  }
  
  if (isLoading) {
    return <LoadingSpinner />;
  }
  
  if (!users || users.length === 0) {
    return <EmptyState message="No users found" />;
  }
  
  return (
    <ul>
      {users.map(user => (
        <UserItem key={user.id} user={user} />
      ))}
    </ul>
  );
}
```

### 2. Element Variables

Store elements in variables for complex conditions:

```jsx
function Dashboard({ user, isLoading }) {
  let content;
  
  if (isLoading) {
    content = <LoadingSpinner />;
  } else if (!user) {
    content = <LoginPrompt />;
  } else if (user.isAdmin) {
    content = <AdminDashboard user={user} />;
  } else {
    content = <UserDashboard user={user} />;
  }
  
  return (
    <div className="dashboard">
      <Header />
      {content}
      <Footer />
    </div>
  );
}
```

### 3. IIFE (Immediately Invoked Function Expression)

For complex logic inside JSX (use sparingly):

```jsx
function Component({ status }) {
  return (
    <div>
      {(() => {
        switch (status) {
          case 'loading':
            return <LoadingSpinner />;
          case 'error':
            return <ErrorMessage />;
          case 'success':
            return <SuccessMessage />;
          default:
            return <IdleState />;
        }
      })()}
    </div>
  );
}
```

### 4. Object Mapping

Map conditions to components:

```jsx
function StatusIcon({ status }) {
  const icons = {
    success: <SuccessIcon />,
    error: <ErrorIcon />,
    warning: <WarningIcon />,
    info: <InfoIcon />
  };
  
  return icons[status] || <DefaultIcon />;
}

// With functions for lazy evaluation
function ContentDisplay({ type }) {
  const components = {
    text: () => <TextContent />,
    image: () => <ImageContent />,
    video: () => <VideoContent />
  };
  
  const Component = components[type];
  return Component ? <Component /> : <DefaultContent />;
}
```

### 5. Component Mapping

Map component types dynamically:

```jsx
function DynamicComponent({ type, props }) {
  const components = {
    button: Button,
    input: Input,
    select: Select,
    textarea: Textarea
  };
  
  const Component = components[type];
  
  if (!Component) {
    console.warn(`Unknown component type: ${type}`);
    return null;
  }
  
  return <Component {...props} />;
}

// Usage
<DynamicComponent type="button" props={{ onClick: handleClick, text: 'Click me' }} />
```

### 6. Render Props

Delegate rendering decision to parent:

```jsx
function DataFetcher({ url, children }) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    fetch(url)
      .then(res => res.json())
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [url]);
  
  return children({ data, loading, error });
}

// Usage - parent controls rendering
<DataFetcher url="/api/users">
  {({ data, loading, error }) => {
    if (loading) return <Spinner />;
    if (error) return <Error message={error.message} />;
    return <UserList users={data} />;
  }}
</DataFetcher>
```

## Conditional CSS Classes

### Using Template Literals

```jsx
function Button({ variant, disabled, active }) {
  return (
    <button
      className={`
        btn 
        btn-${variant} 
        ${disabled ? 'btn-disabled' : ''} 
        ${active ? 'btn-active' : ''}
      `}
    >
      Click me
    </button>
  );
}
```

### Using classnames Library

```jsx
import classNames from 'classnames';

function Button({ variant, disabled, active, large }) {
  return (
    <button
      className={classNames('btn', {
        [`btn-${variant}`]: variant,
        'btn-disabled': disabled,
        'btn-active': active,
        'btn-large': large
      })}
    >
      Click me
    </button>
  );
}
```

### Custom Class Helper

```jsx
function cx(...classes) {
  return classes.filter(Boolean).join(' ');
}

function Button({ variant, disabled, active }) {
  return (
    <button
      className={cx(
        'btn',
        variant && `btn-${variant}`,
        disabled && 'btn-disabled',
        active && 'btn-active'
      )}
    >
      Click me
    </button>
  );
}
```

## Multiple Conditions

### Nested Ternaries (Use Sparingly)

```jsx
// Hard to read
function Status({ status }) {
  return (
    <div>
      {status === 'loading' ? (
        <Spinner />
      ) : status === 'error' ? (
        <Error />
      ) : status === 'success' ? (
        <Success />
      ) : (
        <Idle />
      )}
    </div>
  );
}

// Better: Use early returns or object mapping
```

### Switch Statement Alternative

```jsx
function StatusDisplay({ status }) {
  const getStatusComponent = () => {
    switch (status) {
      case 'loading':
        return <LoadingSpinner />;
      case 'error':
        return <ErrorMessage />;
      case 'success':
        return <SuccessMessage />;
      case 'idle':
        return <IdleState />;
      default:
        return null;
    }
  };
  
  return <div className="status">{getStatusComponent()}</div>;
}
```

### Multiple AND Conditions

```jsx
function SpecialOffer({ user, isPremium, isActive }) {
  return (
    <div>
      {user && isPremium && isActive && (
        <div className="special-offer">
          <h3>Exclusive Premium Offer!</h3>
          <p>Available only for you</p>
        </div>
      )}
    </div>
  );
}

// More readable with variable
function SpecialOffer({ user, isPremium, isActive }) {
  const shouldShowOffer = user && isPremium && isActive;
  
  return (
    <div>
      {shouldShowOffer && (
        <div className="special-offer">
          <h3>Exclusive Premium Offer!</h3>
          <p>Available only for you</p>
        </div>
      )}
    </div>
  );
}
```

## Conditional Styles

### Inline Style Objects

```jsx
function Box({ isActive, size }) {
  return (
    <div
      style={{
        backgroundColor: isActive ? 'blue' : 'gray',
        width: size === 'large' ? '200px' : '100px',
        height: size === 'large' ? '200px' : '100px',
        opacity: isActive ? 1 : 0.5
      }}
    >
      Content
    </div>
  );
}

// With style object
function Box({ isActive, size }) {
  const boxStyle = {
    backgroundColor: isActive ? 'blue' : 'gray',
    width: size === 'large' ? '200px' : '100px',
    height: size === 'large' ? '200px' : '100px',
    opacity: isActive ? 1 : 0.5
  };
  
  return <div style={boxStyle}>Content</div>;
}
```

### Conditional Style Merging

```jsx
function Button({ variant, disabled }) {
  const baseStyle = {
    padding: '10px 20px',
    border: 'none',
    borderRadius: '4px',
    cursor: disabled ? 'not-allowed' : 'pointer'
  };
  
  const variantStyles = {
    primary: { backgroundColor: 'blue', color: 'white' },
    secondary: { backgroundColor: 'gray', color: 'black' }
  };
  
  const disabledStyle = disabled ? { opacity: 0.5 } : {};
  
  return (
    <button
      style={{
        ...baseStyle,
        ...variantStyles[variant],
        ...disabledStyle
      }}
    >
      Click me
    </button>
  );
}
```

## Conditional Props

### Spread Props Conditionally

```jsx
function Input({ error, ...rest }) {
  return (
    <input
      {...rest}
      {...(error && {
        'aria-invalid': true,
        'aria-describedby': 'error-message'
      })}
    />
  );
}

// Cleaner with variable
function Input({ error, ...rest }) {
  const errorProps = error
    ? {
        'aria-invalid': true,
        'aria-describedby': 'error-message'
      }
    : {};
  
  return <input {...rest} {...errorProps} />;
}
```

### Conditional Event Handlers

```jsx
function Button({ onClick, disabled }) {
  return (
    <button onClick={disabled ? undefined : onClick}>
      Click me
    </button>
  );
}

// Or with conditional logic inside
function Button({ onClick, disabled }) {
  const handleClick = (e) => {
    if (disabled) return;
    onClick?.(e);
  };
  
  return <button onClick={handleClick}>Click me</button>;
}
```

## Common Gotchas

### 1. Falsy Values and &&

```jsx
// Dangerous: renders 0
function Component({ count }) {
  return <div>{count && <span>Count: {count}</span>}</div>;
}
// When count is 0, renders: <div>0</div>

// Fix: Explicit boolean conversion
function Component({ count }) {
  return <div>{count > 0 && <span>Count: {count}</span>}</div>;
}

// Or use ternary
function Component({ count }) {
  return <div>{count ? <span>Count: {count}</span> : null}</div>;
}
```

### 2. String Concatenation

```jsx
// Wrong: Always renders something
function Component({ message }) {
  return <div>{message && "Message: " + message}</div>;
}
// When message is empty, renders: <div>false</div>

// Correct
function Component({ message }) {
  return <div>{message ? `Message: ${message}` : null}</div>;
}
```

### 3. Array Length

```jsx
// Dangerous: renders 0 when empty
function List({ items }) {
  return <div>{items.length && <ul>...</ul>}</div>;
}

// Correct
function List({ items }) {
  return <div>{items.length > 0 && <ul>...</ul>}</div>;
}
```

### 4. Null vs Undefined

```jsx
// Both render nothing
function Component() {
  return null; // Explicitly renders nothing
}

function Component() {
  return undefined; // Also renders nothing but less clear
}

// Be explicit
function Component({ show }) {
  if (!show) return null;
  return <div>Content</div>;
}
```

## Performance Considerations

### 1. Memoize Conditional Components

```jsx
function ExpensiveComponent({ data }) {
  // Expensive computation
  const processedData = expensiveProcess(data);
  return <div>{processedData}</div>;
}

function Parent({ showExpensive, data }) {
  // Memoize to prevent unnecessary recalculation
  const expensiveComponent = useMemo(
    () => <ExpensiveComponent data={data} />,
    [data]
  );
  
  return (
    <div>
      {showExpensive && expensiveComponent}
    </div>
  );
}
```

### 2. Lazy Loading Conditional Components

```jsx
import { lazy, Suspense } from 'react';

const AdminPanel = lazy(() => import('./AdminPanel'));
const UserPanel = lazy(() => import('./UserPanel'));

function Dashboard({ isAdmin }) {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      {isAdmin ? <AdminPanel /> : <UserPanel />}
    </Suspense>
  );
}
```

### 3. Avoid Inline Objects

```jsx
// Bad: Creates new object every render
function Component({ show }) {
  return show && <Child data={{ name: 'Alice' }} />;
}

// Good: Stable reference
const data = { name: 'Alice' };

function Component({ show }) {
  return show && <Child data={data} />;
}

// Or with useMemo
function Component({ show, name }) {
  const data = useMemo(() => ({ name }), [name]);
  return show && <Child data={data} />;
}
```

## Best Practices

### 1. Prefer Early Returns

```jsx
// Cleaner
function Component({ user, loading, error }) {
  if (loading) return <Spinner />;
  if (error) return <Error />;
  if (!user) return <Login />;
  return <Dashboard user={user} />;
}

// vs nested ternaries
function Component({ user, loading, error }) {
  return loading ? <Spinner /> : error ? <Error /> : !user ? <Login /> : <Dashboard user={user} />;
}
```

### 2. Extract Complex Conditions

```jsx
// Bad: Hard to understand
function Component({ user, subscription, trial }) {
  return (
    <div>
      {user && subscription && subscription.active && !trial && (
        <PremiumFeature />
      )}
    </div>
  );
}

// Good: Clear intent
function Component({ user, subscription, trial }) {
  const hasActivePremiumSubscription =
    user && subscription?.active && !trial;
  
  return (
    <div>
      {hasActivePremiumSubscription && <PremiumFeature />}
    </div>
  );
}
```

### 3. Use Meaningful Names

```jsx
// Bad
function Component({ flag1, flag2, status }) {
  if (flag1 && flag2 && status === 1) {
    return <A />;
  }
  return <B />;
}

// Good
function Component({ isLoggedIn, isPremium, accountStatus }) {
  const hasFullAccess = isLoggedIn && isPremium && accountStatus === 'active';
  
  if (hasFullAccess) {
    return <PremiumDashboard />;
  }
  return <FreeDashboard />;
}
```

### 4. Keep JSX Clean

```jsx
// Too much logic in JSX
function Component({ users, filter, sort }) {
  return (
    <div>
      {users
        .filter(u => u.name.includes(filter))
        .sort((a, b) => (sort === 'asc' ? a.age - b.age : b.age - a.age))
        .map(user => (
          <UserCard key={user.id} user={user} />
        ))}
    </div>
  );
}

// Better: Extract logic
function Component({ users, filter, sort }) {
  const filteredUsers = users.filter(u =>
    u.name.includes(filter)
  );
  
  const sortedUsers = [...filteredUsers].sort((a, b) =>
    sort === 'asc' ? a.age - b.age : b.age - a.age
  );
  
  return (
    <div>
      {sortedUsers.map(user => (
        <UserCard key={user.id} user={user} />
      ))}
    </div>
  );
}
```

### 5. Document Complex Conditionals

```jsx
function PricingDisplay({ user, subscription, trial }) {
  // Show special pricing in these cases:
  // 1. New users in trial period
  // 2. Users with expired subscription
  // 3. Users with cancelled subscription but still in grace period
  const shouldShowSpecialPricing =
    (trial && trial.isActive) ||
    (subscription && subscription.status === 'expired') ||
    (subscription && subscription.status === 'cancelled' && subscription.inGracePeriod);
  
  return shouldShowSpecialPricing ? (
    <SpecialPricing />
  ) : (
    <StandardPricing />
  );
}
```

## Summary

- Use if statements for early returns and complex conditions
- Ternary operators for simple inline conditionals
- Logical && for conditional rendering (watch out for falsy values)
- Object mapping for multiple conditions
- Extract complex conditions into variables with clear names
- Prefer early returns over nested ternaries
- Be careful with falsy values (0, "", false) with &&
- Keep JSX clean by extracting logic
- Use explicit boolean conversions when needed
- Consider performance with conditional components
- Document complex conditional logic

Mastering conditional rendering patterns helps you write cleaner, more maintainable React code. Choose the right pattern for each situation and always prioritize readability.
