# Context Propagation

## Overview

React Context provides a way to pass data through the component tree without manually passing props at every level. Understanding how context propagates, how React tracks consumers, and the performance characteristics of context updates is essential for using context effectively and avoiding common pitfalls.

This guide explores the internal mechanics of context: the context stack, provider/consumer relationships, propagation algorithm, and optimization strategies.

## Context Stack Architecture

React maintains a stack of context values as it traverses the component tree:

```javascript
// Example 1: Context stack structure (simplified)
type ContextStack = {
  cursor: StackCursor<any>;
  defaultValue: any;
  _currentValue: any;
  _currentValue2: any; // For concurrent rendering
  Provider: React.Provider;
  Consumer: React.Consumer;
};

const valueCursor = createCursor(null);

// Example 2: How context stack works during render
function renderTree() {
  // Stack starts empty
  const stack = [];
  
  // Entering Provider
  push(stack, ThemeContext._currentValue);
  ThemeContext._currentValue = 'dark';
  
  // Render children - they see 'dark'
  renderChildren();
  
  // Exiting Provider
  ThemeContext._currentValue = pop(stack);
}
```

### Provider Implementation

```javascript
// Example 3: Simplified Provider internals
function updateContextProvider(workInProgress) {
  const providerType = workInProgress.type;
  const context = providerType._context;
  const newProps = workInProgress.pendingProps;
  const oldProps = workInProgress.memoizedProps;
  
  const newValue = newProps.value;
  
  // Push new value onto stack
  pushProvider(workInProgress, context, newValue);
  
  if (oldProps !== null) {
    const oldValue = oldProps.value;
    
    // Check if value changed
    if (Object.is(oldValue, newValue)) {
      // Value unchanged - bail out if children haven't changed
      if (oldProps.children === newProps.children) {
        return bailoutOnAlreadyFinishedWork(workInProgress);
      }
    } else {
      // Value changed - propagate change to consumers
      propagateContextChange(workInProgress, context, newValue);
    }
  }
  
  // Render children with new context value
  const newChildren = newProps.children;
  reconcileChildren(workInProgress, newChildren);
  return workInProgress.child;
}

// Example 4: Basic Provider usage
const ThemeContext = createContext('light');

function App() {
  const [theme, setTheme] = useState('dark');
  
  return (
    <ThemeContext.Provider value={theme}>
      {/* All descendants can access 'theme' */}
      <Header />
      <Main />
      <Footer />
    </ThemeContext.Provider>
  );
}
```

## Consumer Tracking

React tracks which components consume context to optimize updates:

```javascript
// Example 5: Consumer dependencies
type ContextDependency = {
  context: ReactContext<any>;
  next: ContextDependency | null;
  memoizedValue: any;
};

type Fiber = {
  dependencies: {
    lanes: Lanes;
    firstContext: ContextDependency | null;
  } | null;
  // ... other properties
};

// Example 6: Reading context creates dependency
function readContext(context) {
  const value = context._currentValue;
  
  // Record that this fiber depends on this context
  const contextItem = {
    context,
    memoizedValue: value,
    next: null
  };
  
  if (lastContextDependency === null) {
    // First context dependency
    currentlyRenderingFiber.dependencies = {
      lanes: NoLanes,
      firstContext: contextItem
    };
    lastContextDependency = contextItem;
  } else {
    // Append to dependency list
    lastContextDependency = lastContextDependency.next = contextItem;
  }
  
  return value;
}
```

### useContext Implementation

```javascript
// Example 7: useContext internals
function useContext(context) {
  const value = readContext(context);
  return value;
}

// Example 8: Using useContext
function ThemedButton() {
  // Creates dependency: ThemedButton → ThemeContext
  const theme = useContext(ThemeContext);
  
  return (
    <button className={`button-${theme}`}>
      Click me
    </button>
  );
}

// When ThemeContext value changes:
// 1. React finds all fibers depending on ThemeContext
// 2. Schedules updates for those fibers
// 3. Those components re-render with new value
```

## Context Propagation Algorithm

When a Provider's value changes, React propagates the change to consumers:

```javascript
// Example 9: Simplified propagation algorithm
function propagateContextChange(workInProgress, context, nextValue) {
  let fiber = workInProgress.child;
  
  if (fiber !== null) {
    fiber.return = workInProgress;
  }
  
  // Traverse fiber tree depth-first
  while (fiber !== null) {
    let nextFiber;
    
    // Check if fiber consumes this context
    const dependencies = fiber.dependencies;
    if (dependencies !== null) {
      nextFiber = fiber.child;
      
      let dependency = dependencies.firstContext;
      while (dependency !== null) {
        if (dependency.context === context) {
          // Found consumer - schedule update
          if (fiber.tag === ClassComponent) {
            // Class component
            const update = createUpdate(lane);
            enqueueUpdate(fiber, update);
          }
          
          // Mark fiber for update
          fiber.lanes = mergeLanes(fiber.lanes, lane);
          
          // Mark all ancestors for update
          let alternate = fiber.alternate;
          if (alternate !== null) {
            alternate.lanes = mergeLanes(alternate.lanes, lane);
          }
          scheduleContextWorkOnParentPath(fiber.return, lane);
          
          break;
        }
        dependency = dependency.next;
      }
    } else if (fiber.tag === ContextProvider) {
      // If we hit another provider for same context, stop
      if (fiber.type === workInProgress.type) {
        nextFiber = null;
      } else {
        nextFiber = fiber.child;
      }
    } else {
      nextFiber = fiber.child;
    }
    
    // Continue traversing
    if (nextFiber !== null) {
      nextFiber.return = fiber;
      fiber = nextFiber;
    } else {
      // Backtrack
      fiber = fiber.return;
    }
  }
}
```

## Multiple Contexts

```javascript
// Example 10: Multiple context providers
const ThemeContext = createContext('light');
const UserContext = createContext(null);
const LanguageContext = createContext('en');

function App() {
  return (
    <ThemeContext.Provider value="dark">
      <UserContext.Provider value={currentUser}>
        <LanguageContext.Provider value="es">
          <Dashboard />
        </LanguageContext.Provider>
      </UserContext.Provider>
    </ThemeContext.Provider>
  );
}

function Dashboard() {
  // Can consume multiple contexts
  const theme = useContext(ThemeContext);    // 'dark'
  const user = useContext(UserContext);      // currentUser
  const lang = useContext(LanguageContext);  // 'es'
  
  // Fiber has dependency list:
  // ThemeContext → UserContext → LanguageContext
  
  return <div>{/* ... */}</div>;
}

// When any context changes, Dashboard re-renders
```

### Nested Providers

```javascript
// Example 11: Nested providers of same context
function App() {
  return (
    <ThemeContext.Provider value="dark">
      <div>
        {/* This area sees 'dark' */}
        <Header />
        
        <ThemeContext.Provider value="light">
          {/* This area sees 'light' */}
          <Sidebar />
        </ThemeContext.Provider>
        
        {/* This area sees 'dark' again */}
        <Footer />
      </div>
    </ThemeContext.Provider>
  );
}

// Context stack during render:
// 1. Enter outer Provider → stack: ['dark']
// 2. Render Header → sees 'dark'
// 3. Enter inner Provider → stack: ['dark', 'light']
// 4. Render Sidebar → sees 'light'
// 5. Exit inner Provider → stack: ['dark']
// 6. Render Footer → sees 'dark'
```

## Performance Characteristics

Context updates can trigger many re-renders:

```javascript
// Example 12: Context update performance
const AppContext = createContext(null);

function App() {
  const [state, setState] = useState({
    user: null,
    theme: 'light',
    language: 'en'
  });
  
  return (
    <AppContext.Provider value={state}>
      {/* Every consumer re-renders when ANY part of state changes */}
      <Header />     {/* Only needs user */}
      <Sidebar />    {/* Only needs theme */}
      <Content />    {/* Only needs language */}
    </AppContext.Provider>
  );
}

function Header() {
  const { user } = useContext(AppContext);
  // Re-renders when theme or language change (unnecessary)
  return <div>{user?.name}</div>;
}

// Problem: Changing theme causes Header to re-render
// even though it only uses user
```

### Split Contexts

```javascript
// Example 13: Solution - split contexts
const UserContext = createContext(null);
const ThemeContext = createContext('light');
const LanguageContext = createContext('en');

function App() {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('light');
  const [language, setLanguage] = useState('en');
  
  return (
    <UserContext.Provider value={user}>
      <ThemeContext.Provider value={theme}>
        <LanguageContext.Provider value={language}>
          <Header />
          <Sidebar />
          <Content />
        </LanguageContext.Provider>
      </ThemeContext.Provider>
    </UserContext.Provider>
  );
}

function Header() {
  const user = useContext(UserContext); // Only this context
  // Now only re-renders when user changes
  return <div>{user?.name}</div>;
}
```

### Memoization Pattern

```javascript
// Example 14: Memoizing context value
function App() {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('light');
  
  // WRONG: New object every render
  const value = { user, theme };
  
  // RIGHT: Memoized value
  const memoizedValue = useMemo(
    () => ({ user, theme }),
    [user, theme]
  );
  
  return (
    <AppContext.Provider value={memoizedValue}>
      <Content />
    </AppContext.Provider>
  );
}

function Content() {
  const context = useContext(AppContext);
  
  // With memoized value: only re-renders when user or theme actually change
  // Without memoization: re-renders on every parent render
  
  return <div>{/* ... */}</div>;
}
```

### Selector Pattern

```javascript
// Example 15: Context with selectors (advanced)
const StoreContext = createContext(null);

function createStore(initialState) {
  const listeners = new Set();
  let state = initialState;
  
  return {
    getState: () => state,
    setState: (newState) => {
      state = newState;
      listeners.forEach(listener => listener());
    },
    subscribe: (listener) => {
      listeners.add(listener);
      return () => listeners.delete(listener);
    }
  };
}

function useStoreSelector(selector) {
  const store = useContext(StoreContext);
  const [, forceUpdate] = useReducer(x => x + 1, 0);
  const selectedValue = useRef();
  
  useEffect(() => {
    const checkForUpdates = () => {
      const newValue = selector(store.getState());
      if (newValue !== selectedValue.current) {
        selectedValue.current = newValue;
        forceUpdate();
      }
    };
    
    return store.subscribe(checkForUpdates);
  }, [store, selector]);
  
  return selectedValue.current;
}

// Usage
function Header() {
  // Only re-renders when user changes, not when theme changes
  const user = useStoreSelector(state => state.user);
  return <div>{user?.name}</div>;
}
```

## Context vs Props

```javascript
// Example 16: When to use context vs props
// Props: Explicit, good for nearby components
function Button({ theme, onClick, children }) {
  return (
    <button className={`button-${theme}`} onClick={onClick}>
      {children}
    </button>
  );
}

// Context: Implicit, good for distant components
const ThemeContext = createContext('light');

function DeepComponent() {
  // Doesn't need theme passed through 10 layers
  const theme = useContext(ThemeContext);
  return <div className={`container-${theme}`} />;
}

// Example 17: Component composition > context
// LESS GOOD: Using context for component customization
function Page() {
  return (
    <LayoutContext.Provider value={{ sidebar: <CustomSidebar /> }}>
      <Layout />
    </LayoutContext.Provider>
  );
}

function Layout() {
  const { sidebar } = useContext(LayoutContext);
  return <div>{sidebar}</div>;
}

// BETTER: Use component composition
function Page() {
  return <Layout sidebar={<CustomSidebar />} />;
}

function Layout({ sidebar }) {
  return <div>{sidebar}</div>;
}
```

## Default Values

```javascript
// Example 18: Default context values
const ThemeContext = createContext('light');

function Component() {
  const theme = useContext(ThemeContext);
  // If no Provider above, uses default 'light'
  return <div className={`theme-${theme}`} />;
}

// Without Provider
function App() {
  return <Component />; // Uses default 'light'
}

// With Provider
function App() {
  return (
    <ThemeContext.Provider value="dark">
      <Component /> {/* Uses 'dark' */}
    </ThemeContext.Provider>
  );
}

// Example 19: Default values with functions
const ComplexContext = createContext({
  user: null,
  login: () => console.warn('No provider'),
  logout: () => console.warn('No provider')
});

// Default functions warn about missing provider
// Helps catch bugs in development
```

## Context and Reconciliation

```javascript
// Example 20: Context doesn't block reconciliation
function App() {
  const [count, setCount] = useState(0);
  
  return (
    <ThemeContext.Provider value="dark">
      <button onClick={() => setCount(count + 1)}>
        {count}
      </button>
      <ExpensiveTree />
    </ThemeContext.Provider>
  );
}

const ExpensiveTree = React.memo(function ExpensiveTree() {
  const theme = useContext(ThemeContext);
  // Expensive rendering...
  return <div>{/* ... */}</div>;
});

// When count changes:
// 1. Provider props changed (children is new)
// 2. But context value ('dark') unchanged
// 3. React.memo prevents ExpensiveTree re-render
// 4. Context doesn't block memoization
```

## Common Pitfalls

```javascript
// Example 21: Inline object as value
function BadApp() {
  const [user, setUser] = useState(null);
  
  return (
    // WRONG: New object every render
    <UserContext.Provider value={{ user, setUser }}>
      <Content />
    </UserContext.Provider>
  );
}

// Every render creates new object → all consumers re-render

// Example 22: Fix with useMemo
function GoodApp() {
  const [user, setUser] = useState(null);
  
  const value = useMemo(
    () => ({ user, setUser }),
    [user] // setUser is stable
  );
  
  return (
    <UserContext.Provider value={value}>
      <Content />
    </UserContext.Provider>
  );
}
```

## Common Misconceptions

1. **"Context is for global state"** - Not exactly. Context is for dependency injection. State management is a separate concern. Context just passes values down.

2. **"Context makes props obsolete"** - False. Props are still better for explicit, nearby data flow. Context is for implicit, distant dependencies.

3. **"Context causes performance problems"** - Partially true. Context itself is efficient. Problems come from poor value management (new objects every render) or too-coarse contexts.

4. **"You should have one context for your entire app"** - False. Multiple smaller contexts are better for performance and maintainability.

5. **"Context updates are synchronous"** - True for the value change, but re-renders are scheduled and may be batched like any other update.

## Performance Implications

1. **Provider Value Changes** - When Provider value changes (Object.is), React traverses the tree looking for consumers. This is O(n) where n is tree size, but skips subtrees without consumers.

2. **Consumer Re-renders** - All consumers re-render when context changes. No way to prevent this except splitting contexts or using selectors.

3. **Memoization Compatibility** - React.memo works with context. Components can skip re-renders if props haven't changed, even if they consume context that hasn't changed.

4. **Multiple Contexts** - Using multiple contexts in one component creates dependencies on all of them. Any change triggers re-render.

## Interview Questions

1. **Q: How does React track which components consume a context?**
   A: When a component calls useContext, React creates a context dependency and adds it to a linked list on the fiber. This dependencies list is checked during context propagation to determine which components need to re-render.

2. **Q: Explain the context stack and how it works.**
   A: React maintains a stack of context values as it traverses the tree. When entering a Provider, it pushes the new value onto the stack. While rendering children, they read from the stack top. When exiting the Provider, it pops the stack, restoring the previous value.

3. **Q: What happens when a Provider's value changes?**
   A: React uses Object.is() to compare old and new values. If different, it traverses the fiber tree depth-first, checking each fiber's dependencies. Fibers consuming that context are marked for update, and React schedules re-renders for them.

4. **Q: Why does putting an object literal as a Provider value cause problems?**
   A: A new object literal is created every render, so Object.is(oldValue, newValue) is always false. This causes React to propagate changes and re-render all consumers even when the data inside hasn't actually changed. Memoize the value with useMemo.

5. **Q: How do nested Providers of the same context work?**
   A: Inner Providers shadow outer ones. React pushes the inner value onto the context stack, and components inside see that value. When exiting the inner Provider, the stack pops back to the outer value. Components can only see the nearest Provider.

6. **Q: Can React.memo prevent re-renders from context?**
   A: Only if the context value hasn't changed. If a memoized component consumes context, it still re-renders when that context changes. React.memo only prevents re-renders from unchanged props, not from context updates.

7. **Q: What's the performance difference between one big context vs multiple small contexts?**
   A: Multiple small contexts allow fine-grained updates. With one big context, any change causes all consumers to re-render. With split contexts, only consumers of the changed context re-render, significantly reducing unnecessary work.

8. **Q: Why can't you use context to avoid prop drilling completely?**
   A: You can technically, but it's often worse. Props provide explicit data flow that's easier to understand and refactor. Context should be for truly global concerns (theme, auth, i18n), not for passing data a few levels down.

## Key Takeaways

1. React maintains a context stack as it traverses the component tree
2. useContext creates a dependency from the fiber to the context
3. Provider value changes trigger propagation to all consumers
4. Object.is() is used to detect context value changes
5. All consumers re-render when context changes—no partial updates
6. Split contexts for better performance and maintainability
7. Memoize context values to prevent unnecessary propagations
8. Context is for dependency injection, not necessarily state management

## Resources

- [Context Documentation](https://react.dev/learn/passing-data-deeply-with-context)
- [Context API Reference](https://react.dev/reference/react/createContext)
- [How to Use Context Effectively](https://kentcdodds.com/blog/how-to-use-react-context-effectively)
- [Context Source Code](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberNewContext.js)
- [When to Use Context](https://react.dev/learn/passing-data-deeply-with-context#before-you-use-context)
