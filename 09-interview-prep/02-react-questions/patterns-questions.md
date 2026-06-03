# React Design Patterns Interview Questions

## Overview

Design pattern questions test your ability to write reusable, maintainable React code. Understanding patterns like HOCs, render props, compound components, and composition is essential for senior roles.

---

## Core Pattern Questions

### Q1: Explain Higher-Order Components (HOCs). When would you use them?

**Answer:**

A Higher-Order Component is a function that takes a component and returns a new component with enhanced functionality.

**Basic HOC Structure:**

```javascript
// HOC function
function withLoading(Component) {
  return function WithLoadingComponent({ isLoading, ...props }) {
    if (isLoading) return <div>Loading...</div>;
    return <Component {...props} />;
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
    <UserListWithLoading isLoading={loading} users={users} />
  );
}
```

**Real-World HOC: Authentication:**

```javascript
function withAuth(Component) {
  return function AuthenticatedComponent(props) {
    const { user, loading } = useAuth();
    
    if (loading) return <Spinner />;
    if (!user) return <Navigate to="/login" />;
    
    return <Component {...props} user={user} />;
  };
}

// Usage
function Dashboard({ user }) {
  return <div>Welcome, {user.name}</div>;
}

export default withAuth(Dashboard);
```

**HOC with Configuration:**

```javascript
function withLogger(Component, componentName) {
  return function LoggedComponent(props) {
    useEffect(() => {
      console.log(`${componentName} mounted`);
      return () => console.log(`${componentName} unmounted`);
    }, []);
    
    console.log(`${componentName} rendered with props:`, props);
    
    return <Component {...props} />;
  };
}

const UserListWithLogger = withLogger(UserList, 'UserList');
```

**Composing Multiple HOCs:**

```javascript
// Individual HOCs
const withAuth = (Component) => (props) => {
  const { user } = useAuth();
  if (!user) return <Navigate to="/login" />;
  return <Component {...props} user={user} />;
};

const withAnalytics = (Component) => (props) => {
  useEffect(() => {
    trackPageView(Component.name);
  }, []);
  
  return <Component {...props} />;
};

const withErrorBoundary = (Component) => {
  return class extends React.Component {
    state = { hasError: false };
    
    static getDerivedStateFromError() {
      return { hasError: true };
    }
    
    render() {
      if (this.state.hasError) return <ErrorPage />;
      return <Component {...this.props} />;
    }
  };
};

// Compose HOCs
const enhance = compose(
  withAuth,
  withAnalytics,
  withErrorBoundary
);

const EnhancedDashboard = enhance(Dashboard);

// Or nest manually
const Dashboard = withAuth(
  withAnalytics(
    withErrorBoundary(DashboardComponent)
  )
);
```

**Pros:**
- Reusable logic across components
- Separation of concerns
- Props injection
- Conditional rendering logic

**Cons:**
- Wrapper hell (deeply nested)
- Prop naming collisions
- Refs don't pass through (need forwardRef)
- Hard to trace in DevTools

**Modern Alternative (Hooks):**

```javascript
// Instead of HOC
function withUser(Component) {
  return function WithUserComponent(props) {
    const user = useUser();
    return <Component {...props} user={user} />;
  };
}

// Use custom hook
function Dashboard() {
  const user = useUser();
  return <div>Welcome, {user.name}</div>;
}
```

**What Interviewers Look For:**
- Understanding of HOC pattern
- Knowledge of composition
- Awareness of modern alternatives
- Ability to explain trade-offs

---

### Q2: What are render props and how do they differ from HOCs?

**Answer:**

Render props is a pattern where a component takes a function as a prop and calls it to render content.

**Basic Render Prop:**

```javascript
// Component with render prop
function Mouse({ render }) {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  
  useEffect(() => {
    const handleMove = (e) => {
      setPosition({ x: e.clientX, y: e.clientY });
    };
    
    window.addEventListener('mousemove', handleMove);
    return () => window.removeEventListener('mousemove', handleMove);
  }, []);
  
  return render(position);
}

// Usage
function App() {
  return (
    <Mouse
      render={({ x, y }) => (
        <div>
          Mouse position: {x}, {y}
        </div>
      )}
    />
  );
}
```

**Children as Function (Variant):**

```javascript
function Mouse({ children }) {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  
  useEffect(() => {
    const handleMove = (e) => {
      setPosition({ x: e.clientX, y: e.clientY });
    };
    
    window.addEventListener('mousemove', handleMove);
    return () => window.removeEventListener('mousemove', handleMove);
  }, []);
  
  return children(position);
}

// Usage
function App() {
  return (
    <Mouse>
      {({ x, y }) => (
        <div>Mouse: {x}, {y}</div>
      )}
    </Mouse>
  );
}
```

**Real-World Example: Data Fetcher:**

```javascript
function DataFetcher({ url, children }) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    setLoading(true);
    fetch(url)
      .then(r => r.json())
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [url]);
  
  return children({ data, loading, error });
}

// Usage
function UserProfile({ userId }) {
  return (
    <DataFetcher url={`/api/users/${userId}`}>
      {({ data: user, loading, error }) => {
        if (loading) return <Spinner />;
        if (error) return <Error message={error.message} />;
        return <div>{user.name}</div>;
      }}
    </DataFetcher>
  );
}
```

**HOC vs Render Props Comparison:**

```javascript
// Same functionality implemented both ways

// 1. HOC approach
function withData(Component, url) {
  return function WithDataComponent(props) {
    const [data, setData] = useState(null);
    const [loading, setLoading] = useState(true);
    
    useEffect(() => {
      fetch(url)
        .then(r => r.json())
        .then(setData)
        .finally(() => setLoading(false));
    }, []);
    
    return <Component {...props} data={data} loading={loading} />;
  };
}

const UserProfile = withData(
  ({ data, loading }) => {
    if (loading) return <Spinner />;
    return <div>{data.name}</div>;
  },
  '/api/user'
);

// 2. Render Props approach
function DataFetcher({ url, render }) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    fetch(url)
      .then(r => r.json())
      .then(setData)
      .finally(() => setLoading(false));
  }, [url]);
  
  return render({ data, loading });
}

const UserProfile = () => (
  <DataFetcher
    url="/api/user"
    render={({ data, loading }) => {
      if (loading) return <Spinner />;
      return <div>{data.name}</div>;
    }}
  />
);
```

**Advantages of Render Props:**

1. **Dynamic Composition:**
```javascript
<DataFetcher url="/api/user">
  {({ data, loading }) => (
    <div>
      {loading ? <Spinner /> : <UserCard user={data} />}
      <DataFetcher url="/api/posts">
        {({ data: posts }) => <PostList posts={posts} />}
      </DataFetcher>
    </div>
  )}
</DataFetcher>
```

2. **No Prop Naming Conflicts:**
```javascript
// HOC can have naming collisions
const Enhanced = withUser(withTheme(Component));
// What if both provide a 'data' prop?

// Render props: you control naming
<User>
  {(userData) => (
    <Theme>
      {(themeData) => (
        <Component user={userData} theme={themeData} />
      )}
    </Theme>
  )}
</User>
```

**Disadvantages:**

1. **Callback Hell:**
```javascript
<A>
  {(a) => (
    <B>
      {(b) => (
        <C>
          {(c) => (
            <D>
              {(d) => (
                <Component a={a} b={b} c={c} d={d} />
              )}
            </D>
          )}
        </C>
      )}
    </B>
  )}
</A>
```

2. **Performance:**
```javascript
// Creates new function every render
<Mouse render={({ x, y }) => <div>{x}, {y}</div>} />

// Solution: extract to component
const MouseDisplay = ({ x, y }) => <div>{x}, {y}</div>;
<Mouse render={MouseDisplay} />
```

**Modern Alternative (Hooks):**

```javascript
// Custom hook replaces both HOC and render props
function useMouse() {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  
  useEffect(() => {
    const handleMove = (e) => {
      setPosition({ x: e.clientX, y: e.clientY });
    };
    
    window.addEventListener('mousemove', handleMove);
    return () => window.removeEventListener('mousemove', handleMove);
  }, []);
  
  return position;
}

// Usage: Clean and simple
function App() {
  const { x, y } = useMouse();
  return <div>Mouse: {x}, {y}</div>;
}
```

**What Interviewers Look For:**
- Understanding of render props pattern
- Ability to compare with HOCs
- Awareness of hooks as modern alternative
- Knowledge of trade-offs

---

### Q3: Explain the Compound Component pattern

**Answer:**

Compound components work together to form a complete UI component, sharing implicit state.

**Basic Example:**

```javascript
// Create context for shared state
const TabsContext = createContext();

function Tabs({ children, defaultTab }) {
  const [activeTab, setActiveTab] = useState(defaultTab);
  
  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

function TabList({ children }) {
  return <div className="tab-list">{children}</div>;
}

function Tab({ id, children }) {
  const { activeTab, setActiveTab } = useContext(TabsContext);
  const isActive = activeTab === id;
  
  return (
    <button
      className={isActive ? 'active' : ''}
      onClick={() => setActiveTab(id)}
    >
      {children}
    </button>
  );
}

function TabPanels({ children }) {
  return <div className="tab-panels">{children}</div>;
}

function TabPanel({ id, children }) {
  const { activeTab } = useContext(TabsContext);
  
  if (activeTab !== id) return null;
  
  return <div className="tab-panel">{children}</div>;
}

// Attach as properties
Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panels = TabPanels;
Tabs.Panel = TabPanel;

// Usage
function App() {
  return (
    <Tabs defaultTab="home">
      <Tabs.List>
        <Tabs.Tab id="home">Home</Tabs.Tab>
        <Tabs.Tab id="profile">Profile</Tabs.Tab>
        <Tabs.Tab id="settings">Settings</Tabs.Tab>
      </Tabs.List>
      
      <Tabs.Panels>
        <Tabs.Panel id="home">
          <h1>Home Content</h1>
        </Tabs.Panel>
        <Tabs.Panel id="profile">
          <h1>Profile Content</h1>
        </Tabs.Panel>
        <Tabs.Panel id="settings">
          <h1>Settings Content</h1>
        </Tabs.Panel>
      </Tabs.Panels>
    </Tabs>
  );
}
```

**Real-World Example: Accordion:**

```javascript
const AccordionContext = createContext();

function Accordion({ children, allowMultiple = false }) {
  const [openItems, setOpenItems] = useState([]);
  
  const toggleItem = (id) => {
    if (allowMultiple) {
      setOpenItems(prev =>
        prev.includes(id)
          ? prev.filter(i => i !== id)
          : [...prev, id]
      );
    } else {
      setOpenItems(prev => prev.includes(id) ? [] : [id]);
    }
  };
  
  return (
    <AccordionContext.Provider value={{ openItems, toggleItem }}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
}

function AccordionItem({ id, children }) {
  const { openItems } = useContext(AccordionContext);
  const isOpen = openItems.includes(id);
  
  return (
    <div className={`accordion-item ${isOpen ? 'open' : ''}`}>
      {children}
    </div>
  );
}

function AccordionHeader({ id, children }) {
  const { toggleItem } = useContext(AccordionContext);
  
  return (
    <button
      className="accordion-header"
      onClick={() => toggleItem(id)}
    >
      {children}
    </button>
  );
}

function AccordionContent({ id, children }) {
  const { openItems } = useContext(AccordionContext);
  const isOpen = openItems.includes(id);
  
  if (!isOpen) return null;
  
  return <div className="accordion-content">{children}</div>;
}

Accordion.Item = AccordionItem;
Accordion.Header = AccordionHeader;
Accordion.Content = AccordionContent;

// Usage
function FAQ() {
  return (
    <Accordion allowMultiple>
      <Accordion.Item id="q1">
        <Accordion.Header id="q1">What is React?</Accordion.Header>
        <Accordion.Content id="q1">
          React is a JavaScript library...
        </Accordion.Content>
      </Accordion.Item>
      
      <Accordion.Item id="q2">
        <Accordion.Header id="q2">What are hooks?</Accordion.Header>
        <Accordion.Content id="q2">
          Hooks are functions that let you...
        </Accordion.Content>
      </Accordion.Item>
    </Accordion>
  );
}
```

**Benefits:**

1. **Flexible API:**
```javascript
// Can customize structure
<Tabs>
  <Tabs.List>
    <Tabs.Tab id="a">Tab A</Tabs.Tab>
    <div>Some other content</div>
    <Tabs.Tab id="b">Tab B</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panels>
    <Tabs.Panel id="a">Content A</Tabs.Panel>
    <Tabs.Panel id="b">Content B</Tabs.Panel>
  </Tabs.Panels>
</Tabs>
```

2. **Clear Hierarchy:**
```javascript
// Self-documenting structure
<Accordion>
  <Accordion.Item>
    <Accordion.Header>Title</Accordion.Header>
    <Accordion.Content>Body</Accordion.Content>
  </Accordion.Item>
</Accordion>
```

3. **Shared State:**
```javascript
// No props drilling needed
// All sub-components access state via context
```

**What Interviewers Look For:**
- Understanding of implicit state sharing
- Knowledge of Context usage
- Ability to design flexible APIs
- Awareness of component composition

---

## Key Takeaways

**Essential Concepts:**
1. HOCs wrap components to add functionality
2. Render props delegate rendering to consumers
3. Compound components share implicit state
4. Hooks are the modern alternative to most patterns
5. Composition over inheritance

**Best Practices:**
- Prefer hooks for reusable logic
- Use compound components for related UI
- Keep HOCs simple and focused
- Avoid deep nesting with render props
- Document component APIs clearly

**Common Mistakes:**
- Over-using HOCs (use hooks instead)
- Render prop callback hell
- Not forwarding refs in HOCs
- Poor naming in compound components
- Mixing patterns unnecessarily

**Interview Tips:**
- Show multiple pattern approaches
- Explain trade-offs
- Demonstrate modern alternatives
- Provide real-world examples
- Discuss when to use each pattern

**Red Flags to Avoid:**
- "HOCs are always better than hooks"
- Not knowing modern alternatives
- Can't explain when to use patterns
- Over-engineering simple components
- Ignoring composition principles
