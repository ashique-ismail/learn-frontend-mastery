# React State Management Interview Questions

## The Idea

**In plain English:** State management is about deciding where your app "remembers" information — like which user is logged in, what's in a shopping cart, or whether a menu is open — and how different parts of the app can read or change that information.

**Real-world analogy:** Think of a busy restaurant. The head chef (the app) needs to track every table's order. A waiter at one table writes the order on a small notepad just for that table — but if two waiters both need to know the total bill, the manager at the front desk keeps a shared copy everyone can check.

- The waiter's notepad = local state (only one component needs it)
- The manager's front desk record = lifted state (a parent holds it so siblings can share it)
- The restaurant's central computer system = global state (any part of the app can read or update it)

---

## Overview

State management questions test your understanding of local vs global state, when to use different solutions (Context, Redux, Zustand, etc.), and how to architect scalable state in React applications.

---

## Core State Management Questions

### Q1: Explain the difference between local state, lifted state, and global state. When do you use each?

**Answer:**

Understanding state scope and placement is fundamental to React architecture.

**Local State:**

State that belongs to a single component and isn't shared.

```javascript
// Local state example
function Counter() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}

// ✓ Use for: Form inputs, toggles, local UI state
// ✓ Benefits: Isolated, simple, no prop passing needed
// ✗ Limitations: Can't share with other components
```

**When to Use Local State:**

```javascript
// 1. Form inputs
function LoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  
  return (
    <form>
      <input value={email} onChange={e => setEmail(e.target.value)} />
      <input value={password} onChange={e => setPassword(e.target.value)} />
    </form>
  );
}

// 2. UI toggles
function Accordion() {
  const [isOpen, setIsOpen] = useState(false);
  
  return (
    <div>
      <button onClick={() => setIsOpen(!isOpen)}>Toggle</button>
      {isOpen && <div>Content</div>}
    </div>
  );
}

// 3. Component-specific state
function SearchInput() {
  const [query, setQuery] = useState('');
  const [suggestions, setSuggestions] = useState([]);
  
  useEffect(() => {
    if (query) {
      fetchSuggestions(query).then(setSuggestions);
    }
  }, [query]);
  
  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <SuggestionsList suggestions={suggestions} />
    </div>
  );
}
```

**Lifted State:**

Moving state to a common parent when multiple children need it.

```javascript
// Before: Can't share between siblings
function ParentBefore() {
  return (
    <div>
      <CounterA />
      <CounterB />
    </div>
  );
}

function CounterA() {
  const [count, setCount] = useState(0);
  return <div>A: {count}</div>;
}

function CounterB() {
  const [count, setCount] = useState(0);
  return <div>B: {count}</div>;
}

// After: Lifted to parent for sharing
function ParentAfter() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <CounterA count={count} setCount={setCount} />
      <CounterB count={count} setCount={setCount} />
    </div>
  );
}

function CounterA({ count, setCount }) {
  return (
    <div>
      A: {count}
      <button onClick={() => setCount(count + 1)}>+</button>
    </div>
  );
}

function CounterB({ count, setCount }) {
  return (
    <div>
      B: {count}
      <button onClick={() => setCount(count + 1)}>+</button>
    </div>
  );
}

// Now both components share the same count!
```

**Real-World Lifted State Example:**

```javascript
function ShoppingCart() {
  const [cart, setCart] = useState([]);
  
  const addItem = (item) => {
    setCart([...cart, item]);
  };
  
  const removeItem = (id) => {
    setCart(cart.filter(item => item.id !== id));
  };
  
  const total = cart.reduce((sum, item) => sum + item.price, 0);
  
  return (
    <div>
      <ProductList onAddItem={addItem} />
      <CartSummary cart={cart} onRemoveItem={removeItem} total={total} />
    </div>
  );
}

// ProductList and CartSummary both need cart data
// So it's lifted to ShoppingCart parent
```

**When to Use Lifted State:**

1. Siblings need to share state
2. Parent needs to coordinate children
3. Limited scope (not app-wide)
4. Keeps state close to where it's used

**Global State:**

State accessible throughout the entire application.

```javascript
// Global state with Context
const AuthContext = createContext();

function App() {
  const [user, setUser] = useState(null);
  
  const login = async (credentials) => {
    const userData = await api.login(credentials);
    setUser(userData);
  };
  
  const logout = () => {
    setUser(null);
  };
  
  return (
    <AuthContext.Provider value={{ user, login, logout }}>
      <Router>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/dashboard" element={<Dashboard />} />
          <Route path="/profile" element={<Profile />} />
        </Routes>
      </Router>
    </AuthContext.Provider>
  );
}

// Any component can access auth state
function Dashboard() {
  const { user } = useContext(AuthContext);
  return <div>Welcome, {user?.name}</div>;
}

function Profile() {
  const { user, logout } = useContext(AuthContext);
  return (
    <div>
      <h1>{user?.name}</h1>
      <button onClick={logout}>Logout</button>
    </div>
  );
}
```

**When to Use Global State:**

1. Authentication/user data
2. Theme/preferences
3. Language/localization
4. App-wide settings
5. Shopping cart (e-commerce)
6. Notifications
7. Feature flags

**Decision Tree:**

```
Does only one component need this state?
├─ Yes → Local State ✓
└─ No → Continue

    Do only siblings need it?
    ├─ Yes → Lifted State ✓
    └─ No → Continue
        
        Do many unrelated components need it?
        ├─ Yes → Global State ✓
        └─ No → Lifted State ✓
```

**Anti-Pattern: Premature Global State:**

```javascript
// ❌ BAD: Making everything global
const AppContext = createContext();

function App() {
  const [modalOpen, setModalOpen] = useState(false);
  const [selectedTab, setSelectedTab] = useState(0);
  const [inputValue, setInputValue] = useState('');
  // ... 20 more state variables
  
  return (
    <AppContext.Provider value={{ /* everything */ }}>
      <Page />
    </AppContext.Provider>
  );
}

// ✓ GOOD: Keep state local, lift when needed
function App() {
  return <Page />;
}

function Page() {
  const [selectedTab, setSelectedTab] = useState(0);
  
  return (
    <div>
      <Tabs selectedTab={selectedTab} setSelectedTab={setSelectedTab} />
      <TabContent selectedTab={selectedTab} />
    </div>
  );
}

function Modal() {
  const [open, setOpen] = useState(false);
  // Local state - only Modal needs it
  
  return (
    <>
      <button onClick={() => setOpen(true)}>Open</button>
      {open && <ModalContent onClose={() => setOpen(false)} />}
    </>
  );
}
```

**What Interviewers Look For:**
- Clear understanding of state scope
- Ability to choose appropriate state location
- Avoiding premature optimization
- Keeping state close to where it's used

---

### Q2: Compare Redux, Context, and Zustand. When would you choose each?

**Answer:**

Each solution has different strengths, complexity levels, and use cases.

**Context API:**

React's built-in solution for sharing state.

```javascript
// Context implementation
const ThemeContext = createContext();

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');
  
  const value = useMemo(() => ({ theme, setTheme }), [theme]);
  
  return (
    <ThemeContext.Provider value={value}>
      {children}
    </ThemeContext.Provider>
  );
}

// Usage
function Button() {
  const { theme } = useContext(ThemeContext);
  return <button className={theme}>Click</button>;
}
```

**Pros:**
- Built into React (no dependencies)
- Simple for basic needs
- Good for low-frequency updates
- Perfect for theme, auth, locale

**Cons:**
- All consumers re-render on any change
- No built-in middleware
- No dev tools
- Boilerplate for complex state

**When to Use Context:**
- Simple global state (theme, auth)
- Small to medium apps
- Infrequent updates
- No complex state logic

---

**Redux:**

Predictable state container with unidirectional data flow.

```javascript
// Redux setup
import { createSlice, configureStore } from '@reduxjs/toolkit';

// Slice
const todoSlice = createSlice({
  name: 'todos',
  initialState: [],
  reducers: {
    addTodo: (state, action) => {
      state.push({ id: Date.now(), text: action.payload, done: false });
    },
    toggleTodo: (state, action) => {
      const todo = state.find(t => t.id === action.payload);
      if (todo) todo.done = !todo.done;
    }
  }
});

// Store
const store = configureStore({
  reducer: {
    todos: todoSlice.reducer
  }
});

// Component
function TodoList() {
  const todos = useSelector(state => state.todos);
  const dispatch = useDispatch();
  
  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>
          <input
            type="checkbox"
            checked={todo.done}
            onChange={() => dispatch(toggleTodo(todo.id))}
          />
          {todo.text}
        </li>
      ))}
    </ul>
  );
}
```

**Pros:**
- Excellent dev tools
- Time-travel debugging
- Middleware ecosystem
- Predictable state updates
- Great for large apps
- Selector optimization

**Cons:**
- Significant boilerplate
- Learning curve
- Overkill for simple apps
- Bundle size (~12KB)

**When to Use Redux:**
- Large, complex applications
- Need dev tools and debugging
- Multiple developers
- Complex state logic
- Need middleware (sagas, thunks)

---

**Zustand:**

Lightweight state management with hooks.

```javascript
// Zustand setup
import create from 'zustand';

// Store
const useTodoStore = create((set) => ({
  todos: [],
  addTodo: (text) =>
    set((state) => ({
      todos: [...state.todos, { id: Date.now(), text, done: false }]
    })),
  toggleTodo: (id) =>
    set((state) => ({
      todos: state.todos.map(t =>
        t.id === id ? { ...t, done: !t.done } : t
      )
    }))
}));

// Component
function TodoList() {
  const todos = useTodoStore(state => state.todos);
  const toggleTodo = useTodoStore(state => state.toggleTodo);
  
  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>
          <input
            type="checkbox"
            checked={todo.done}
            onChange={() => toggleTodo(todo.id)}
          />
          {todo.text}
        </li>
      ))}
    </ul>
  );
}
```

**Pros:**
- Minimal boilerplate
- Small bundle size (~1KB)
- Built-in selectors
- No provider needed
- Easy to learn
- Good dev tools

**Cons:**
- Less mature ecosystem
- Smaller community
- Fewer middleware options
- Less structured than Redux

**When to Use Zustand:**
- Medium apps
- Want simplicity with power
- Don't need Redux complexity
- Like hook-based API
- Want small bundle size

---

**Comparison Table:**

| Feature | Context | Redux | Zustand |
|---------|---------|-------|---------|
| Bundle Size | 0KB | ~12KB | ~1KB |
| Boilerplate | Medium | High | Low |
| Learning Curve | Low | High | Low |
| Dev Tools | No | Excellent | Good |
| Selectors | Manual | Built-in | Built-in |
| Middleware | No | Rich | Basic |
| Re-render Control | Poor | Good | Excellent |

**Real-World Example: E-commerce App:**

```javascript
// Use ALL three based on needs:

// 1. Context for theme (simple, infrequent)
<ThemeContext.Provider value={theme}>

  // 2. Redux for products/cart (complex, needs middleware)
  <Provider store={reduxStore}>
  
    // 3. Zustand for UI state (simple, frequent)
    <App />
    
  </Provider>
</ThemeContext.Provider>

// Theme (Context)
function ThemeToggle() {
  const { theme, setTheme } = useContext(ThemeContext);
  return <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
    Toggle
  </button>;
}

// Products (Redux - complex async logic)
function ProductList() {
  const products = useSelector(state => state.products.items);
  const dispatch = useDispatch();
  
  useEffect(() => {
    dispatch(fetchProducts());
  }, []);
  
  return products.map(p => <ProductCard key={p.id} product={p} />);
}

// UI state (Zustand - simple, frequent updates)
const useUIStore = create(set => ({
  sidebarOpen: false,
  toggleSidebar: () => set(state => ({ sidebarOpen: !state.sidebarOpen }))
}));

function Sidebar() {
  const { sidebarOpen, toggleSidebar } = useUIStore();
  return sidebarOpen ? <aside>...</aside> : null;
}
```

**What Interviewers Look For:**
- Understanding of each solution's strengths
- Ability to choose based on requirements
- Knowledge of trade-offs
- Experience with multiple solutions

**Red Flags:**
- "Redux is always best"
- "Context can replace Redux everywhere"
- Not knowing trade-offs
- Can't explain when to use which

---

### Q3: How do you handle async state in React?

**Answer:**

Async state management is crucial for real-world apps that fetch data, handle loading, and manage errors.

**Pattern 1: Basic useState with Loading/Error:**

```javascript
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    async function fetchUser() {
      setLoading(true);
      setError(null);
      
      try {
        const response = await fetch(`/api/users/${userId}`);
        const data = await response.json();
        setUser(data);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    }
    
    fetchUser();
  }, [userId]);
  
  if (loading) return <Spinner />;
  if (error) return <Error message={error} />;
  if (!user) return null;
  
  return <div>{user.name}</div>;
}
```

**Pattern 2: useReducer for Complex State:**

```javascript
const initialState = {
  data: null,
  loading: false,
  error: null
};

function reducer(state, action) {
  switch (action.type) {
    case 'FETCH_START':
      return { ...state, loading: true, error: null };
    case 'FETCH_SUCCESS':
      return { data: action.payload, loading: false, error: null };
    case 'FETCH_ERROR':
      return { ...state, loading: false, error: action.payload };
    default:
      return state;
  }
}

function UserProfile({ userId }) {
  const [state, dispatch] = useReducer(reducer, initialState);
  
  useEffect(() => {
    async function fetchUser() {
      dispatch({ type: 'FETCH_START' });
      
      try {
        const response = await fetch(`/api/users/${userId}`);
        const data = await response.json();
        dispatch({ type: 'FETCH_SUCCESS', payload: data });
      } catch (error) {
        dispatch({ type: 'FETCH_ERROR', payload: error.message });
      }
    }
    
    fetchUser();
  }, [userId]);
  
  if (state.loading) return <Spinner />;
  if (state.error) return <Error message={state.error} />;
  if (!state.data) return null;
  
  return <div>{state.data.name}</div>;
}
```

**Pattern 3: Custom Hook for Reusability:**

```javascript
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    let cancelled = false;
    
    async function fetchData() {
      setLoading(true);
      
      try {
        const response = await fetch(url);
        const json = await response.json();
        
        if (!cancelled) {
          setData(json);
          setError(null);
        }
      } catch (err) {
        if (!cancelled) {
          setError(err.message);
        }
      } finally {
        if (!cancelled) {
          setLoading(false);
        }
      }
    }
    
    fetchData();
    
    return () => {
      cancelled = true;
    };
  }, [url]);
  
  return { data, loading, error };
}

// Usage
function UserProfile({ userId }) {
  const { data: user, loading, error } = useFetch(`/api/users/${userId}`);
  
  if (loading) return <Spinner />;
  if (error) return <Error message={error} />;
  
  return <div>{user.name}</div>;
}
```

**Pattern 4: React Query (Recommended):**

```javascript
import { useQuery } from '@tanstack/react-query';

function UserProfile({ userId }) {
  const { data: user, isLoading, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetch(`/api/users/${userId}`).then(r => r.json())
  });
  
  if (isLoading) return <Spinner />;
  if (error) return <Error message={error.message} />;
  
  return <div>{user.name}</div>;
}

// Benefits:
// - Automatic caching
// - Background refetching
// - Deduplication
// - Pagination support
// - Optimistic updates
```

**Pattern 5: Redux Toolkit Query:**

```javascript
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

const api = createApi({
  reducerPath: 'api',
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  endpoints: (builder) => ({
    getUser: builder.query({
      query: (id) => `users/${id}`
    })
  })
});

export const { useGetUserQuery } = api;

// Usage
function UserProfile({ userId }) {
  const { data: user, isLoading, error } = useGetUserQuery(userId);
  
  if (isLoading) return <Spinner />;
  if (error) return <Error message={error.message} />;
  
  return <div>{user.name}</div>;
}
```

**Handling Race Conditions:**

```javascript
function SearchResults() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  
  useEffect(() => {
    let cancelled = false;
    
    async function search() {
      if (!query) {
        setResults([]);
        return;
      }
      
      const response = await fetch(`/api/search?q=${query}`);
      const data = await response.json();
      
      // Only update if this is still the latest query
      if (!cancelled) {
        setResults(data);
      }
    }
    
    search();
    
    return () => {
      cancelled = true;
    };
  }, [query]);
  
  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <ResultsList results={results} />
    </div>
  );
}
```

**What Interviewers Look For:**
- Proper loading/error handling
- Understanding of race conditions
- Knowledge of modern solutions (React Query)
- Cleanup in useEffect

---

## Key Takeaways

**Essential Concepts:**
1. Start with local state, lift when needed
2. Use global state sparingly
3. Choose state management based on complexity
4. Handle async state properly (loading, error, success)
5. Clean up async operations

**Best Practices:**
- Keep state as local as possible
- Lift state to lowest common ancestor
- Use Context for simple global needs
- Use Redux for complex apps
- Use Zustand for medium apps
- Use React Query for server state

**Common Mistakes:**
- Making everything global
- Not handling loading/error states
- Ignoring race conditions
- Over-engineering simple state
- Choosing Redux for small apps

**Interview Tips:**
- Explain state scope decisions
- Show async handling patterns
- Discuss trade-offs between solutions
- Demonstrate cleanup in effects
- Mention modern libraries (React Query)

**Red Flags to Avoid:**
- "Everything should be global"
- Not handling errors
- Ignoring loading states
- Can't explain when to use which solution
- Never used state management libraries
