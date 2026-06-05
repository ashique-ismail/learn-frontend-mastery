# Higher-Order Component vs Custom Hook

## The Idea

**In plain English:** A Higher-Order Component (HOC) and a Custom Hook are two ways to reuse the same piece of logic across many screens in an app. A Higher-Order Component is a wrapper — it takes one screen component and returns a beefed-up version of it; a Custom Hook is a shareable function that any screen can call to get the same data or behavior without any wrapping.

**Real-world analogy:** Think of a security guard booth at the entrance of different office buildings. Every building (component) needs to check whether a visitor is allowed in before letting them through. One approach is to put a pre-built security booth (HOC) around each building's front door — the booth makes the decision and either waves the visitor in or sends them away. Another approach is to give each front-desk receptionist (Custom Hook) a shared checklist they call themselves so they can decide what to show without needing a separate booth around the whole building.

- The security booth = the Higher-Order Component (wraps the component and controls what renders)
- The front-desk receptionist calling the shared checklist = the Custom Hook (called inside the component to get the data)
- The office building = the React component that needs the auth logic

---

## What They Are

**Higher-Order Component (HOC)**: A function that takes a component and returns a new enhanced component. A *component composition* pattern.

**Custom Hook**: A function starting with `use` that encapsulates stateful logic and can call other hooks. A *logic reuse* pattern.

---

## HOC Pattern

```tsx
// HOC: withAuth wraps any component to add authentication guard
function withAuth<P extends {}>(WrappedComponent: React.ComponentType<P>) {
  return function AuthenticatedComponent(props: P) {
    const { user, isLoading } = useAuth();

    if (isLoading) return <Spinner />;
    if (!user) return <Navigate to="/login" />;

    return <WrappedComponent {...props} />;
  };
}

// Usage
const ProtectedDashboard = withAuth(Dashboard);
const ProtectedProfile = withAuth(Profile);
```

---

## Custom Hook Pattern

```tsx
// Custom hook: same auth logic, but as a hook
function useAuth() {
  const [user, setUser] = useState(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    getSession().then(u => { setUser(u); setIsLoading(false); });
  }, []);

  return { user, isLoading };
}

function useRequireAuth() {
  const { user, isLoading } = useAuth();
  const navigate = useNavigate();

  useEffect(() => {
    if (!isLoading && !user) navigate('/login');
  }, [user, isLoading, navigate]);

  return { user, isLoading };
}

// Usage
function Dashboard() {
  const { user, isLoading } = useRequireAuth();
  if (isLoading) return <Spinner />;
  return <div>Welcome, {user.name}</div>;
}
```

---

## Direct Comparison

| | HOC | Custom Hook |
|---|---|---|
| Introduced | React pre-hooks era | React 16.8+ |
| Reuse mechanism | Component wrapping | Function composition |
| Access to JSX | Yes (can render fallbacks) | No (returns data/functions) |
| Multiple enhancements | Wrapper hell | Clean composition |
| TypeScript | Complex (prop forwarding) | Straightforward |
| Debugging | Extra component layers in DevTools | No extra layers |
| Conditional logic | Hard (can't call hooks conditionally) | Easy |
| Testing | Render the HOC wrapper | Test the hook directly |

---

## When HOCs Are Still Useful

1. **Conditional rendering based on logic** — the HOC needs to decide whether to render the component at all:

```tsx
// HOC: don't even render the component if feature is off
const withFeatureFlag = (flag: string) => (Component: ComponentType) =>
  function FeatureFlagged(props: any) {
    return useFeatureFlag(flag) ? <Component {...props} /> : null;
  };
```

2. **Adding props from external sources** (legacy patterns, library integrations):

```tsx
// Class component libraries still use HOCs
const ConnectedComponent = connect(mapState, mapDispatch)(MyClassComponent);
```

3. **Error boundaries** — must be class components, so wrapped via HOC:

```tsx
const withErrorBoundary = (Component: ComponentType, fallback: ReactNode) =>
  function WithBoundary(props: any) {
    return (
      <ErrorBoundary fallback={fallback}>
        <Component {...props} />
      </ErrorBoundary>
    );
  };
```

---

## When Custom Hooks Are Better

Almost always in modern React — share logic without the wrapper layer:

```tsx
// HOC version — wrapper hell
const EnhancedComponent = withLogging(withAuth(withTheme(withData(MyComponent))));

// Hook version — flat, composable
function MyComponent() {
  const { user } = useAuth();
  const theme = useTheme();
  const data = useData();
  useLogging('MyComponent');
  // ...
}
```

---

## Common Interview Questions

**Q: What problem do HOCs solve?**
Cross-cutting concerns — auth, logging, error handling, feature flags — that multiple components need. They were the standard reuse mechanism before hooks.

**Q: What's "wrapper hell" with HOCs?**
Composing many HOCs creates deeply nested component trees that are hard to read in React DevTools and confusing to type in TypeScript. `withA(withB(withC(Component)))` creates 3 extra component layers.

**Q: Can you rewrite any HOC as a custom hook?**
Not always — HOCs can render conditional JSX (show a spinner, redirect, render a fallback). Hooks only return values/functions, not JSX. For rendering control, HOCs (or Render Props) are still needed. For logic reuse, hooks are always cleaner.
